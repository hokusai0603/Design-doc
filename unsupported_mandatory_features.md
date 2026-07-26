# OpenDaisy 尚未支援之 3GPP ML Model Provisioning 必填/核心功能清單

本文件依據 [mlmodelprovision_mapping.md](file:///home/king25158986/Design%20doc/mlmodelprovision_mapping.md) 的欄位對照以及 [ideal_mlmodelprovision_flow.md](file:///home/king25158986/Design%20doc/ideal_mlmodelprovision_flow.md) 的架構討論，整理出若要符合 3GPP TS 23.288 與 TS 29.520 規範，**OpenDaisy 目前尚未支援且必須實作的核心功能清單**。

---

## 1. 訂閱生命週期管理功能 (Subscription Lifecycle Management)

3GPP 規範要求 MTLF 必須將訂閱視為持久的「資源」來管理。OpenDaisy 目前僅有一次性任務發佈機制，缺少以下必填功能：

* **修改訂閱服務 (`PUT /subscriptions/{subscriptionId}`)**：
  * **標準要求**：消費者 (AnLF) 在訓練過程中可以修改訂閱參數（例如調整評估門檻、變更通知網址 `notifUri`、調整預測目標時間 `mLTargetPeriod` 等）。
  * **OpenDaisy 現況**：無此 API，無法中途變更已發佈的任務配置。
* **取消訂閱/終止任務服務 (`DELETE /subscriptions/{subscriptionId}`)**：
  * **標準要求**：消費者可以隨時中止訂閱，MTLF 應立即停止對應的聯邦學習訓練並釋放系統資源。
  * **OpenDaisy 現況**：尚未實作。一旦發佈任務，除非重啟 Master 進程，否則無法透過 API 中止背景執行的聯邦學習任務。
* **訂閱資源狀態維護 (`subscriptionId` 追蹤)**：
  * **標準要求**：MTLF 必須為每個訂閱生成唯一的 `subscriptionId` 並維護其狀態（Active / Suspended / Terminated）。
  * **OpenDaisy 現況**：僅有單一的 `task_id`，沒有生命週期狀態管理。

---

## 2. 3GPP 標準 API 協議格式相容 (Standard API Schema Compliance)

OpenDaisy 的 RESTful API 參數為客製化的扁平 JSON 格式，無法與標準核心網網元 (NF) 直接對接：

* **標準 Analytics ID 映射 (`mLEvent` 與 `mLEventFilter`)**：
  * **標準要求**：訂閱必須指定標準事件類型（例如：`LOAD_LEVELING` 負載均衡、`UE_MOBILITY` 使用者移動性）與過濾條件（如特定切片 S-NSSAI 或地理區域 AoI）。
  * **OpenDaisy 現況**：不支援事件與過濾，僅識別自訂的 `TID` (Task ID) 並跑固定的 PyTorch TCN 結構。
* **標準 JSON 封裝與適配層 (Adapter Layer)**：
  * **標準要求**：API 端點應相容 TS 29.520 定義的 `NwdafMLModelProvSubsc` (Request) 與 `NwdafMLModelProvNotif` (Notification Callback) 格式。
  * **OpenDaisy 現況**：實作於 [server_api_handler.py](file:///home/king25158986/OpenDaisy/src/py/daisyfl/master/server_api_handler.py)，目前的格式為自訂結構。

---

## 3. 模型準確度監控與退化主動回報 (Accuracy Monitoring & Degradation Notify - TS 23.288 6.2E)

3GPP 的核心訴求之一是「閉環監控」，確保模型在真實網絡中退化時能自動觸發更新：

* **動態準確度門檻監控 (`mLModelAccuracyThreshold`)**：
  * **標準要求**：消費者在訂閱時可指定模型精度的容忍門檻。當模型精度低於此門檻時，MTLF 應主動發送通知。
  * **OpenDaisy 現況**：雖然聯邦學習策略中支援 Early Stopping，但該數值是寫死在代碼或配置中，無法透過 API 進行外部動態訂閱與門檻越界監控。
* **中間進度與指標主動推送**：
  * **標準要求**：在長達數天的訓練過程中，MTLF 應依據訂閱模式（如週期性）回報當前的模型指標給 AnLF。
  * **OpenDaisy 現況**：目前僅在訓練結束時推送一次結果，中途進度完全不透光（僅能透過外部 Prometheus 主動 Pull `/metrics` 取得，無法主動 Push 告知 AnLF 訂閱端）。

---

## 4. 與 ADRF 的標準化資料對接 (Standardized ADRF Integration)

在 3GPP 架構中，MTLF 不直接接觸原始資料，必須透過 ADRF (Ad-hoc Data Repository Function) 存取：

* **基於 `DataSetTag` 的資料提取**：
  * **標準要求**：訓練時，Client 應使用訂閱中指定的 `DataSetTag` 與 `ADRF ID`，向 ADRF 請求對應的歷史數據包。
  * **OpenDaisy 現況**：Client 僅能從本地硬編碼的文件夾（例如 `ees_training_data`）載入 CSV/Parquet 檔案，或使用非標準的 MongoDB 查詢。
* **標準化模型註冊**：
  * **標準要求**：訓練完成的模型應註冊至 ADRF，回報時提供 ADRF ID 供 AnLF 檢索，而非傳送臨時的 HTTP 下載連結。
  * **OpenDaisy 現況**：直接將 `.tar.gz` 檔案存放在 Master 節點本地磁碟的 `artifacts/` 下，無法在 5G 核心網跨主機共享。

---

## 5. 電信級高可用解耦架構 (Carrier-grade HA Decoupling)

考慮到 OpenDaisy 長時間通訊與 gRPC 長連接的特性，以下系統架構功能尚未支援：

* **無狀態控制面與非同步任務排隊**：
  * **標準要求**：接收 API 請求的 HTTP 伺服器必須是無狀態的，將任務寫入資料庫（如 MongoDB）後立即回傳，防範 HTTP 逾時。
  * **OpenDaisy 現況**：API Handler 內部直接使用 `threading.Thread` 在記憶體中拉起 gRPC Master，當大量任務併發或進程重啟時，排隊中的任務將會徹底遺失。
* **生產級 gRPC 連線防護 (Keepalive & 動態容錯)**：
  * **標準要求**：gRPC 通訊在長達數天的 FL 過程中，需防範網路抖動與 Client 隨機斷線。
  * **OpenDaisy 現況**：缺乏健全的心跳檢測，若部分 Client 中斷，可能導致整個 FL 輪數卡死或聚合失敗。
