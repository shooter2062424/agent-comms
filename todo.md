# TODO — 已發現的缺陷與改進項目

閱讀 `src/core` 與 `src/bridges` 時發現的問題，依嚴重度排序。
背景說明見 [`docs/process-mechanism.md`](docs/process-mechanism.md)。

---

## P0 — 網格會永久裂開

### 1. 協調者崩潰後沒有任何重選機制

**現況**：`becomeCoordinator()` 全專案只有兩個呼叫點——`MeshStore.init()`（行程啟動時）與
`handleBecomeCoordinator()`（收到 `become_coordinator` 訊息時）。而 `become_coordinator`
這則訊息**沒有任何地方送出**。同時 `coordinatorSocket` 上只掛了 `data` 與 `error`
（error 還是空實作），**沒有 `close` 處理器**，所以倖存 peer 根本不會察覺協調者已經消失。

**後果**：協調者死亡後，下一個啟動的 bridge 會綁上 19876 成為新協調者，但它的 `peerInfo`
是空的——它不知道倖存 peer 的存在，倖存 peer 也永遠不會知道它的存在。**網格分裂成兩個
互不相通的孤島，而且不會自癒。** 連帶停擺的還有：5 秒 PID 巡邏（只在協調者上跑）、
連線核可入口、聯邦鏈路入口。

CLAUDE.md 宣稱的「remaining peers race to bind the port（~100ms recovery）」未實作，
`src/test/` 也沒有任何 failover 測試。

**修法**（由小而大，建議至少做 1a + 1b）：

- [ ] **1a. 優雅移交**——`become_coordinator` 的 wire 型別、TCP/TLS 兩邊的接收路徑、
      `handleBecomeCoordinator()` 全都寫好了，就差沒人送。在協調者的 `shutdown()` 裡挑
      `startedAt` 最早的 peer 送出即可，涵蓋「使用者關掉某個 session」這個最常見情境。
- [ ] **1b. 崩潰偵測 + 錯開重選**——在 `coordinatorSocket` 掛 `close` 處理器，新增
      `onCoordinatorLost` transport 事件，MeshStore 依自己 `startedAt` 的排名延遲
      （最久 0ms、次之 150ms…）再嘗試 `becomeCoordinator()`；搶輸收到 `EADDRINUSE`
      就改連上贏家。**注意**：必須是「一次判斷」而非緊密重試迴圈，否則會踩到
      `init()` 註解提到的 Node TLS session cache bug 而凍結 event loop。
- [ ] **1c. 重選後重新交握**——新協調者上任時，把 `peerConnections` 裡既有連線的
      PeerInfo 灌回 `peerInfo` 並廣播 `peer_list`；倖存 peer 重送 `introduce`。
      這一步才真正修好分裂。
- [ ] **1d. 孤島自救**——`init()` 因 `EADDRINUSE` 降級的 peer 目前永久放棄，
      加一個低頻（如 30 秒）重試探測，讓它在孤兒行程消失後能回到網格。
- [ ] **1e.** 補 integration test：殺掉協調者行程 → 斷言剩餘 peer 在 N 毫秒內
      選出新協調者，且訊息仍然送得到。

---

## P1 — 房間會變成無主狀態

### 2. owner 離開後沒有 ownership 移交

**現況**：`room.owner` 是建立者的 agent ID，而三條離開路徑沒有任何一條會更新它：

| 離開方式 | 行為 | owner |
|---|---|---|
| `leave_room` | 從 members 移除、廣播 room_upsert | 不變 |
| `SIGTERM` 優雅關閉 | `setAgentOffline()` 只改 status | 不變，人還留在 members |
| 行程崩潰 | 5 秒探測標 offline，30 分鐘後從 agents 刪除 | 不變，成為死指標 |

`leaveRoom()` 尾端只有一個特例——`members.length === 0 && owner === agentId` 才
`destroyRoom()`。**owner 離開但房裡還有人 → 房間立刻無主**，`invite`、`kick`、
`destroy_room` 對所有人永久回傳 `NOT_OWNER`。

- [ ] **2a.** 新增 `reassignOwnedRooms(agentId)`：掃該 agent 擁有的房間，owner 換成
      **最早加入且仍在線**的成員（`members` 本身就是加入順序），一個都沒有就 `room_delete`。
- [ ] **2b.** 在 `setAgentOffline()` 與 `probeStaleAgents()` 共用的下游呼叫它，
      讓正常關閉與崩潰兩條路徑都能收尾。
- [ ] **2c.** `leaveRoom()` 也要呼叫（正常離開同樣不會移交，這是目前就存在的 bug）。

> ⚠️ **依賴關係**：崩潰偵測只在協調者上跑，若掛掉的正好是協調者本人，沒有任何人會探測，
> 移交也就不會發生。**必須先做 #1，#2 的崩潰路徑才真正有效。**

### 3. 離線 / 已清除的 agent 不會被移出 room.members

**現況**：整個 codebase 只有 `leaveRoom()` 與 `kickFromRoom()` 會動 `room.members`，
`probeStaleAgents()` 只改 status。所以 30 分鐘後 agent 從 `agents` map 被清掉，
`room.members` 裡仍留著它的 ID。

**後果**：`joinRoom()` 組成員名單時 `agents.get(memberId)` 查不到就靜靜跳過（成員數對不上）；
訊息投遞則會持續對一個不存在的 agent 塞進 `deliveryQueues`，永遠沒人來收（記憶體洩漏）。

- [ ] **3a.** 在 `probeStaleAgents()` 的 purge 階段，一併把該 ID 從所有 `room.members`
      與 `room.invited` 移除並廣播 `room_upsert`。
- [ ] **3b.** 同時清掉該 agent 的 `deliveryQueues` 項目。

---

## P2 — 可見性語意與文件不符

### 4. `hidden` 在功能上等同 `visible`

**現況**：`MeshStore.listAgents()` 只做 `if (agent.visibility === "ghost" && ...) continue;`，
`CommsTool.listAgents()` 拿到結果後完全不再過濾，輸出還包含 `id / name / harness /
status / visibility / cwd / Rooms`。

**後果**：`hidden` 的 agent 連 ID、工作目錄、所屬房間全部被列出，甚至那欄還標著 `hidden`。
文件宣稱的「不列出，但知道 ID 的人仍可 DM」完全沒有實作，三段可見性實際上只剩兩段。

- [ ] **4a.** `listAgents()` 改成把 `hidden` 也一起濾掉（自己永遠看得到自己）：
      `if (agent.id !== requesterId && (visibility === "ghost" || visibility === "hidden")) continue;`
      `sendDm()` 不用動——它本來就只擋 `ghost`，剛好是正確語意。
- [ ] **4b.** 補測試：A 設 hidden → B 的 `list_agents` 看不到 A，但 B 用已知 ID 仍 DM 得到 A。

---

## P3 — 文件與實作落差

- [ ] **5.** CLAUDE.md 的「coordinator hands over to the longest-running peer」與
      「~100ms recovery」在 #1 完成前是不實描述，應標註為規劃中或先行修正。
- [ ] **6.** CLAUDE.md 的可見性表格在 #4 完成前與實作不符。
- [ ] **7.** CLAUDE.md 未涵蓋 repo 中已存在的功能：TLS transport、federation、
      mDNS / Tailscale discovery、listener policy、Web UI、web-push。
      （`docs/process-mechanism.md` 已補上這些。）

---

## 建議施作順序

1. **#1a + #1b**（協調者 failover）— 投報率最高，且是 #2 崩潰路徑的前提
2. **#2 + #3**（ownership 移交與成員清理）— 依賴 #1
3. **#4**（hidden 過濾）— 獨立、改動最小
4. **#5–#7**（文件同步）— 跟著上面的改動一起更新
