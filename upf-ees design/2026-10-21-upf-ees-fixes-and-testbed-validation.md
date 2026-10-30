# UPF-EES 修復、SMF 更新與 NWDAF 串接測試紀錄

- 測試日期：2026-10-21。
- 範圍：embedded UPF-EES、更新後的官方 SMF，以及 NWDAF 訂閱與通知收集流程。
- 本文件整理已完成的修復及測試紀錄；建立文件時未再次啟動 testbed，也未重新執行測試。

## 1. 結論與驗證範圍

先前評論的五項修正，即原編號 **1、2、3、4、6**，均通過對應回歸測試。原編號 5 的 URR 選擇則保留 ID 2，但明確限縮為 free5GC 單 UPF、每個 session 的 MAQE 使用情境，並非宣稱任意 SMF 或 ULCL 都適用。

SMF 改以當次確認的官方 `main` commit 為基底，不使用 release tag 作為最終更新版本。串接時發現 NWDAF 有三項實際不相容，修復後已在 Vagrant testbed 完成：

1. 真實 UE 註冊並建立 PDU session。
2. 透過 NWDAF API 建立訂閱，由 NWDAF 呼叫 SMF，再由 SMF 建立 UPF 訂閱。
3. UE 產生實際流量，UPF 依 GTP5G／PFCP 的真實量測發送 notifications。
4. NWDAF 正確解析通知並寫入 MongoDB，而非僅回覆 HTTP 204。
5. 驗證 ONE_TIME、session 釋放／重建及訂閱連鎖刪除。

這是指定版本與單一路徑的功能驗證，**不是 TS 29.564 全功能符合性認證，也不是完整 analytics／ML 閉環驗證**。以下區分程式錯誤、官方 SMF 實作限制與本地部署假設，不將所有限制都視為 3GPP 的普遍強制要求。

## 2. 版本與變更保存

| 元件 | 本次驗證版本／狀態 |
| --- | --- |
| UPF | `go-upf`，基底 `0eb726199928` 加上工作目錄內 EES 修正 |
| SMF | 官方 `main`：`0e2c7be5678d27d14eeddb3c12456d74d7586fab`，分支 `codex/smf-free5gc-main`，保留本地 multi-AN／threshold 修改 |
| NWDAF | 基底 `2d6534cb65d5`，分支 `codex/nwdaf-smf-main-compat`，加入本次相容性修正 |
| 其他 core NF | 使用 VM 原有 free5GC v4.1.0 執行檔，未升級整套 core |
| SMF 工具鏈／OpenAPI | Go 1.26.2；OpenAPI `v1.3.1-0.20261014070323-89accd9890ce` |

上述 SMF commit 是 **2026-10-21 更新時確認的 main HEAD**，不代表未來仍為遠端最新 commit。實際驗證包含未提交的工作目錄變更，不能只 checkout 基底 commit 就視為相同測試版本。

SMF 原有程式與中間版本保留於：

- `backup/smf-before-v4.3.0-20261021`：原 custom HEAD `8668372`。
- `backup/smf-before-main-20261021`：固定完整遷移前 stash 物件 `d1503b722fa99cdcfefaf0f07546c097d220de64`；其 tree 保存 tracked 工作檔，第三 parent 保存當時 untracked 檔，stash entry 也仍保留。

本次採用官方 EES handler／client／processor／store，不再疊加舊 custom EES。舊實作可由備份取回，並非同時啟用的第二套 API。修復與測試期間未 commit 或 push。

## 3. 五項 UPF 修正與 URR 2 決策

| 原評論 | 原因與風險 | 修復內容 | 驗證證據 |
| --- | --- | --- | --- |
| 1. PFCP response usage report 重入 queue | 已放入 response 的 usage 再進 PFCP queue，可能重複回報；Remote SEID 被當 Local SEID；主迴圈在滿 queue 時可能阻塞 | 使用 EES-only、僅緩衝資料的獨立 hook；清理前保存 local／remote SEID、UE IP 與 generation，不把 response usage 再送回 PFCP report queue | EES 開啟／關閉與滿 queue 回歸通過；實機 modification／deletion accepted。實機未刻意塞滿 queue |
| 2. UE IPv4 未被維護 | 特定 UE 訂閱無法匹配；anyUe 即使收集到 report，也因缺 IP 無法通知 | 成功建立／更新 PDR 後，從 PDI 的 UE IP 維護 session 身分；未帶新 IP 的更新保留舊值，移除 PDR 時重新計算 | PDR 生命週期測試通過；實機成功匹配 `10.10.0.1` |
| 3. UPF API 與 SMF 不相容 | `ueIpAddress` 型別、週期欄位、Location／subscriptionId 不一致 | 接受 IP object 與 `eventReportingMode.repPeriod`；Location 和 subscriptionId 回傳同一個絕對 URI；支援正確 DELETE 路徑及 h2c | 實際 generated client contract、SMF 串接測試及真實 NWDAF→SMF→UPF 建立／刪除通過 |
| 4. session maps 同步不足 | reader 使用 LocalNode lock，但 PDR／URR writer 未使用同一保護，可能 race 或 concurrent map iteration/write | 以 session 的共享 RWMutex 保護相關寫入及 snapshot；session 配置 generation 區別 SEID 重用 | EES／PFCP 三輪 race 回歸，包括 concurrent snapshot/write，通過 |
| 6. ONE_TIME 將無資料當零值 | 沒有真實量測時捏造零值與區間，甚至無匹配 session 就刪訂閱 | 等待真實量測；真實零值可通知；成功送達才移除，失敗保留資料重試，無匹配 session 保持等待 | 單元測試通過；實機有 session 收到 1 筆、無 session 收到 0 筆且訂閱仍存在 |

### 3.1 為什麼保留原第 5 點的 URR ID 2

URR ID 是 **session-scoped**，不是所有 UE 共用一個全域計數器。不同 UE／PDU session 可以各自具有 URR 2，對應不同的規則與量測狀態。此次 SMF 回歸也確認每個 session 的保留 URR ID 1～6，以及後續 charging ID 的配置行為。

UPF 保留 `SingleUPFMAQEURRID = 2`，依據專案採用的 free5GC 單 UPF MAQE（QoS enforcement 後量測）配置慣例。這不是 3GPP 規定必須使用數字 2。

- 不直接加總所有 URR，因不同規則的量測可能重疊而重複計算。
- SMF 必須配置該 session 的 URR 2 與 reporting triggers；若沒有，就等待資料，不能捏造零值。
- 原評論對可攜性的 concern 仍成立：其他 SMF 編號策略、ULCL、application／flow charging 不在本次支援範圍。
- 若未來擴充上述情境，應改以 session／rule association 選擇量測來源，並處理重疊計量。

相關專案背景連結：[free5GC CHF design](https://free5gc.org/doc/Chf/design/)。本次 URR 決策與限制亦記載於 [UPF EES profile](../../go-upf/internal/ees/README.md)。

### 3.2 其他一併修復的 EES 行為

- TrafficVolume 使用含單位字串，例如 `"123 B"`；真實零值明確為 `"0 B"`。
- 累積資料的 key 包含 local／remote SEID、generation 與 UE IP，避免不同 session 生命週期混合。
- 拒絕空 measurement、不支援的 measurement、多事件及與 PER_SESSION 不相容的 application／flow filters。
- 通知排程遵守各訂閱自己的 `repPeriod`，不再依賴全域 ticker 或第一個 URR 週期；`ees.periodSec` 已 deprecated。
- 不同訂閱可獨立送通知，同一訂閱最多一個 in-flight request；取消會傳遞至 HTTP，shutdown 等待 worker 結束。
- 失敗 batch 保留並與新資料合併，重試間隔至少一秒；明確刪除後不會恢復訂閱或 batch。
- DELETE 必須匹配精確資源路徑；限制 request body 為 1 MB 且只允許單一 JSON object。

通知重試仍為 at-least-once 語意：HTTP timeout 不代表接收端一定沒有處理，不能宣稱 exactly-once。

## 4. SMF 更新與 NWDAF 相容性修復

### 4.1 SMF 更新保留項目

依「以最新 repo commit 而非 tag 更新」的要求，最終採官方 main。保留的本地差異包括：

- 在不連通的多 AN topology 中搜尋可達 UPF。
- 匯出所有 AN component 的 links 並去重。
- 只有 volume threshold 大於零時才啟用對應 URR trigger。
- 保留部署所需 topology、位址及 `nupfEeApiRoot`。

此 commit 的官方 EES profile 要求有效的 `supi`／`pduSeId`、呼叫端 NF UUID、feature 26、有效通知資訊與正值週期，且接受 PERIODIC。這是該實作的接受條件，不等於所有 3GPP 實作都必須採相同限制。

### 4.2 實際發現的三項 NWDAF 不相容

| 問題 | 修復前的 HTTP／資料結果 | 修復 |
| --- | --- | --- |
| 請求缺 `pduSeId`、`nfId`、`supportedFeatures` | 官方 SMF 回 400，無法建立下游訂閱 | 明確設定 SUPI→PDU session，送 NWDAF 自己的 NF UUID 與 feature 26：`"02000000"` |
| UPF callback 指向 `/collector/notify` | UPF payload 被當作空 SMF event；回 204 卻沒有入庫 | 使用 top-level `notifUri` 指向 `/collector/upf-notify`，不再輸出舊 bundled callback 欄位；錯誤 schema 回 400 |
| NWDAF 將 TrafficVolume 定義為整數 JSON | 正確 callback 也無法解碼 `"123 B"`，回 400 | 使用精確十進位轉換解析單位字串，轉為內部 int64 bytes，並相容舊非負整數輸入 |

TrafficVolume 使用 `math/big.Rat` 避免浮點精度損失，支援 decimal SI 的 B／kB／MB／GB／TB。負值、未知單位、overflow 或無法表示為整數 byte 的結果明確拒絕；輸出序列化為 byte 單位字串。

主要 NWDAF 修改集中於 consumer request model／SMF client、factory 設定、data collection session 選擇、collector schema 驗證，以及新增的 TrafficVolume 解析器與回歸測試。未藉此升級 NWDAF 全部 runtime dependencies。

### 4.3 本次設定與請求範例

以下是測試使用的設定片段；PDU session ID 1 由實機 `nr-cli ... -e ps-list` 確認，不能假設所有 UE 都是 1。

```yaml
configuration:
  smf:
    enabled: true
    endpoints: [http://127.0.0.2:8000]
    pduSessionIds:
      imsi-208930000000001: 1
    notifUris:
      upf: http://192.168.107.5:8080/collector/upf-notify
```

缺少或超出範圍的 PDU 對應會記錄並跳過，不隱式使用 1。目前每個 SUPI 只設定一個 PDU session，並非自動 session discovery，也未支援同一 SUPI 同時收集多個 PDU session。此為啟動時設定，變更對應後需重新建立相關訂閱。

送至 `POST /nnwdaf-eventssubscription/v1/subscriptions` 的測試內容：

```json
{
  "eventSubscriptions": [{
    "event": "UE_COMMUNICATION",
    "tgtUe": {"supis": ["imsi-208930000000001"]}
  }],
  "evtReq": {"notifMethod": "PERIODIC", "repPeriod": 5},
  "notificationURI": "http://127.0.0.1:9099/callback"
}
```

`notificationURI` 是 NWDAF analytics 輸出的接收位置，與 `smf.notifUris.upf` 的 UPF 原始量測入口不同。NWDAF 建立訂閱回 201 本身不足以證明下游收集成功，仍須檢查 SMF／UPF 訂閱及實際入庫資料。

## 5. 測試過程

### 5.1 第一階段：本地回歸與 HTTP 串接診斷

先驗證五項 UPF 修正、SMF 更新及 generated-client contract；再以實際 HTTP 路徑診斷 NWDAF→SMF→UPF→NWDAF。

初期本地診斷使用測試 session 與注入 usage，確認上述 400、callback 204 但未入庫，以及 TrafficVolume 解碼失敗。此階段用來定位相容性問題，**不當作真實 UE／kernel 流量的證據**。修正 NWDAF 後，增加對應單元與 race 測試，再進入實機驗證。

### 5.2 第二階段：隔離 testbed 部署

1. 啟動既有 `5GC`、`UPF-EES`、`gNB` Vagrant VM，使用 `--no-provision`。
2. 5GC 共享資料夾掛載失敗，改用 SSH／SCP 傳檔，未重裝 Guest Additions。
3. 新執行檔與設定放在 VM 的 `/home/vagrant/ees-audit-20261021/`；`backup/` 保存舊檔，未覆蓋正常服務原檔。
4. 使用具名測試服務，不執行包含清除 MongoDB collection 動作的原 `run.sh`。
5. 使用獨立 MongoDB database `ees_audit_20261021`，不清除既有資料。
6. 關閉 ML inference／ADRF 相關功能，隔離訂閱與量測收集測試。

### 5.3 第三階段：真實 UE 訂閱與流量

```text
控制：NWDAF subscription API → SMF（選擇有效 PDU session）→ UPF EES
量測：UE → gNB → GTP5G／PFCP URR 2 → EES → NWDAF upf-notify → MongoDB
```

1. 確認新 SMF 與 UPF 建立 PFCP association，gNB 完成 NG Setup。
2. UE `imsi-208930000000001` 註冊成功；確認 PDU ID `1`、UE IPv4 `10.10.0.1`、DNN `internet`、slice `1/010203`。
3. 第一次測試請求未指定 `evtReq`，沿用舊預設 THRESHOLD，得到 501；將測試請求改為明確 PERIODIC／5 秒後成功。此處是測試請求對齊支援 profile，並非宣稱新增 THRESHOLD 支援。
4. NWDAF 建立訂閱回 201，並成功建立 SMF 與 UPF 的下游訂閱。
5. UE 執行 `ping -I uesimtun0 -c 20 -i 0.5 -W 2 192.168.103.10`，取得 **20/20 回覆、0% loss**。
6. 檢查 UPF notification、NWDAF 接收及 MongoDB 持久化，確認真實量測抵達。

本次主要資源識別碼：

| 層級 | 訂閱 ID |
| --- | --- |
| NWDAF | `4a69e4dd-8a47-403b-8992-97b604d1bb35` |
| SMF | `aad69967-7040-47a9-8bb8-d01dddccc079` |
| UPF | `sub-1792568170547054128-0000` |

### 5.4 量測結果

`corr-1` 共儲存 **27 筆 periodic 記錄**，其中兩筆非零；兩筆均匹配 `10.10.0.1`。

| 量測時間（UTC，2026-10-21） | UL bytes | DL bytes | Total bytes |
| --- | ---: | ---: | ---: |
| 07:36:10.579 | 640 | 640 | 1280 |
| 07:36:15.580 | 640 | 640 | 1280 |

其他零值來自真實週期性 URR 回報，不是 EES 在沒有量測時合成的資料。此結果證明量測路徑與入庫運作；未以此宣稱所有封包的計費口徑或完整 charging 流程已驗證。

### 5.5 ONE_TIME、session 生命週期與刪除

ONE_TIME 直接向 UPF 測試，因本次官方 SMF profile 只接受 PERIODIC。

| 情境 | 實際結果 |
| --- | --- |
| 已存在 UE `10.10.0.1` 的 ONE_TIME | 收到 1 筆通知；成功後資源自動移除，再 DELETE 回 404 |
| 不存在 UE `10.10.0.254` 的 ONE_TIME | 0 筆通知；資源仍保留，手動 DELETE 回 204 |
| 釋放 UE session | PFCP Session Deletion Accepted；UE 隨後自動重建並取得 `10.10.0.2`，舊 IP 訂閱未錯收新 IP 資料 |
| 刪除 NWDAF 訂閱 | 回 204，SMF→UPF cascade delete 成功；再次 DELETE 原 UPF 資源回 404 |
| 刪除後查詢資料庫 | periodic／ONE_TIME／無 session 筆數保持 **27／1／0**，沒有持續增加 |

## 6. 自動化測試與已知問題

下列是本次執行與結果摘要；各命令需在對應 repo／module 根目錄執行。

| 元件 | 命令／檢查 | 結果 |
| --- | --- | --- |
| UPF | `go test -race -count=3 ./internal/ees ./internal/pfcp` | 通過 |
| UPF | `go test -short ./...` | 通過 |
| UPF client contract | 在 `tests/contracts/nupf` 執行 `go test -race -count=2 ./...` | 通過 |
| SMF | `go test -count=1 -race -short ./...` | 通過 |
| SMF | short suite、`go vet ./...`、all-package build | 通過 |
| NWDAF | `go test -count=1 -short ./...` | 通過 |
| NWDAF | 下列 callback／client／collection／volume 三輪 race | 通過 |
| NWDAF | `go vet ./...`、`CGO_ENABLED=0 go build -o /tmp/ees-smf-nwdaf-audit.sOxVOg/nwdaf ./cmd` | 通過 |

NWDAF targeted race 命令：

```sh
go test -race -count=3 \
  -run 'TestMissingSessionMapping|TestTrafficVolume|TestSMFMain|TestTriggerTarget|TestHandleUpf|TestUPFTrafficVolume' \
  ./internal/sbi/...
```

不能將上述結果表述為所有 repo 的完整 race suite 全數通過：

- UPF 全套 race 仍有既有 periodic-forwarder test fixture 競爭，先前已在原始基底重現，未於 EES 任務中擴大修改。
- NWDAF 全套 race 仍有既有 ML fixture 的全域 config 競爭：測試恢復 `factory.NwdafConfig` 時，背景初始化尚未結束；已在未修改的 `2d6534c` 副本重現。
- 實機主流程完成後，NWDAF source 額外補上未初始化 context 的 nil 防護，已重新執行相關測試及編譯；該額外防護未再跑整套實機流程。

## 7. 測試後狀態與尚未涵蓋範圍

測試完成後，UE 已去註冊，測試服務停止，三台 VM 正常關機，恢復原本停止狀態。保留測試資料庫、VM 隔離目錄、備份與證據檔，沒有刪除既有資料。

測試版**未永久取代正常部署**。正式啟用仍需選用新版執行檔，並加入經確認的 `pduSessionIds` 與正確 callback 設定；舊 runtime 設定不會自動取得這些變更。

本次未涵蓋或仍有限制：

- 第二條 UPF／slice、IPv6、ULCL、所有多 UE／多 session 組合、實機滿 queue 壓力與長時間 soak test。
- UPF 以 IPv4 匹配，未將 SUPI／DNN／S-NSSAI 作為獨立 filter；同一 UPF 上重疊 IPv4 address domain 需要擴充身分模型。
- Feature negotiation、expiry／max-report 控制及 immediate-report requests 未完整實作。
- Kernel 量測產生速度不會因更短的 `repPeriod` 自動加快；無資料區間不產生人工通知。
- 靜態 PDU 對應不是自動 discovery，也未提供同一 SUPI 多 PDU 同時收集。
- ML／ADRF 在此測試停用，不代表 analytics 精度、模型推論或完整 charging 服務通過。
- 對外 scheme／path 與直連不同的 reverse proxy 仍需 advertised-root 設計，不能直接套用本次直連 URI 假設。

## 8. 原始紀錄與後續查閱

持久化於專案內的主要資料：

- [UPF 實機驗證報告](../../go-upf/tests/reports/2026-10-21-testbed.md)
- [MongoDB 測試結果摘要](../../go-upf/tests/reports/2026-10-21-testbed-db.json)
- [UPF EES 支援 profile 與回歸說明](../../go-upf/internal/ees/README.md)
- [SMF main 更新、備份與呼叫端遷移說明](../../free5gc-NWDAF/5G_Infrastructure/5GC/smf-nwdaf-ext/docs/impl/main-upgrade.md)
- [NWDAF 相容性修復與設定說明](../../free5gc-NWDAF/5G_Infrastructure/NWDAF/NWDAF/docs/impl/smf-main-compatibility.md)

補充診斷資料位於本機 `/tmp/ees-smf-nwdaf-audit.sOxVOg/`，包含 HTTP 診斷、NWDAF／SMF log 摘錄、測試請求、執行檔 SHA-256 與資料庫摘要；VM 內測試檔位於 `/home/vagrant/ees-audit-20261021/`。`/tmp` 非永久保存位置，重要結果已摘要於本文件及上列專案報告。

注意：暫存目錄中的初期 `REPORT.md` 記錄修復前診斷階段，其待測事項不代表最後狀態；應以本文件及最終 testbed 報告為準。SMF 遷移文件內「尚未部署」的敘述亦屬遷移階段，本次後續曾部署至隔離測試目錄，但未覆蓋正常服務。
