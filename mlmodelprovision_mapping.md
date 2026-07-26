# 3GPP ML Model Provisioning 映射與差距分析設計文檔

本文件分析了 3GPP NWDAF ML Model Provisioning 規範（**TS 23.288** 與 **TS 29.520**）與 **OpenDaisy** 聯邦學習框架（以 [07_MTLF_training](file:///home/king25158986/OpenDaisy/examples/07_MTLF_training/) 範例為參考實作）之間的欄位映射與實作差距。

---

## 1. 架構與流程對照

在 3GPP 標準中，ML 模型配置（Provisioning）由 **`Nnwdaf_MLModelProvision`** 服務提供：
- **Subscribe (POST)**：訂閱 ML 模型。
- **Notify (POST 回呼)**：當模型完成訓練、更新或退化時，通知消費者。
- **Unsubscribe (DELETE)**：取消訂閱。

OpenDaisy 映射方式：
- **`Nnwdaf_MLModelProvision_Subscribe`** -> 映射至 OpenDaisy [server_api_handler.py](file:///home/king25158986/OpenDaisy/src/py/daisyfl/master/server_api_handler.py#L115) 的 `/publish_task` (POST)。
- **`Nnwdaf_MLModelProvision_Notify`** -> 映射至 OpenDaisy 訓練結束後的 `CALLBACK_URL` 回呼機制（POST 回傳任務狀態與下載 URL）。
- **`Nnwdaf_MLModelProvision_Unsubscribe`** -> **尚未實作**。OpenDaisy 目前沒有取消執行中背景任務的 API。

---

## 2. 訂閱/請求階段欄位對照表 (`NwdafMLModelProvSubsc` vs. `/publish_task`)

此階段由消費者（如 AnLF）向 MTLF 訂閱或請求模型。以下是標準欄位與 OpenDaisy 的支援狀況：

| 3GPP 標準欄位名 (TS 29.520 / 23.288) | 資料類型 | OpenDaisy 對應欄位 | 支援狀態 | 目前實作與差距說明 |
| :--- | :--- | :--- | :--- | :--- |
| **`notifUri`** | String (URI) | `CALLBACK_URL` | **部分支援** | OpenDaisy 使用扁平的 `CALLBACK_URL` 接收回呼。建議修改為標準的 `notifUri`。 |
| **`mLEventSubscs.mLEvent`** | Enum | `TID` | **部分支援** | 3GPP 使用標準的 Analytics ID（如 `LOAD_LEVELING`）。OpenDaisy 使用自訂任務 ID（`TID`）。建議建立對照映射。 |
| **`mLEventSubscs.mLEventFilter`** | Object | 無 | **尚未實作** | 缺少 3GPP 標準的過濾條件，如 S-NSSAI、地理區域限制（AoI）等。 |
| **`tgtUe`** | Object | MongoDB 組別比對 | **部分支援** | 3GPP 用於指定目標 UE 列表或 Internal-Group-Id。OpenDaisy 會根據 `TID` 從 MongoDB 讀取數據群組來啟動 Client，但 API 參數中沒有開放此欄位輸入。 |
| **`mLTargetPeriod`** | Object | 無 | **尚未實作** | 缺少預測目標的時間區間（UTC 時間窗）。 |
| **`modelInterInfo`** | Object | 無 (固定使用 `.npy`) | **尚未實作** | 互通性設定（例如指定要求 ONNX、TensorFlow 或 PyTorch 格式模型）在 OpenDaisy 中尚未實作。 |
| **`nfConsumerInfo`** / **`nfConsumerInfo.nfInstanceId`** | Object | 無 | **尚未實作** | 尚未實作消費者 Vendor ID 或 NF 實例 ID 的來源追蹤。 |
| **`timeModelNeeded`** | String (Date-Time) | 無 | **尚未實作** | 尚未支援指定模型最晚需要交付的截止時間。 |
| **`inferDataForModel`** / **`DataSetTag & ADRF ID`** | Object | `SEED_MODEL_PATH`<br>`SEED_SCALER_PATH` | **部分支援** | 3GPP 會指定 ADRF 中的 DataSetTag 來引導 retraining。OpenDaisy 目前只能使用本機檔案路徑載入初始權重。 |
| **`useCaseCxt`** | String | 無 | **尚未實作** | 用於在多個模型中篩選特定場景模型的上下文標籤，OpenDaisy 尚未實作。 |
| **`mlEvRepCon`** | Object | `NUM_ROUNDS`<br>`EVALUATE_INTERVAL` | **部分支援** | 3GPP 的回報條件（如週期性或閥值觸發）。OpenDaisy 僅能在 `task.json` 中靜態設定訓練輪數與評估間隔。 |
| **`immRep`** | Boolean | 無 | **尚未實作** | 是否立即回報已有模型（Immediate Report）的標記，OpenDaisy 尚未實作。 |

---

## 3. 通知/回呼階段欄位對照表 (`NwdafMLModelProvNotif` vs. Callback Output)

此階段由 MTLF 將訓練完成的模型資訊推送給消費者：

| 3GPP 標準欄位名 (TS 29.520 / 23.288) | 資料類型 | OpenDaisy 對應欄位 | 支援狀態 | 目前實作與差距說明 |
| :--- | :--- | :--- | :--- | :--- |
| **`subscriptionId`** | String | `task_id` (TID) | **部分支援** | OpenDaisy 回傳 `task_id`，而非獨立的訂閱 ID。 |
| **`eventNotifs.event`** | Enum | 無 | **尚未實作** | 回呼 Payload 中沒有回傳標準的 ML 事件類型（如 `LOAD_LEVELING`）。 |
| **`eventNotifs.mLFileAddr`** | String (URI) | `model_url` | **已支援** | 提供模型的下載位址（`http://{ip}:{port}/download/{task_id}`）。 |
| **`eventNotifs.mLModelAdrf`** | Object | 無 | **尚未實作** | OpenDaisy 不支援將模型打包並註冊儲存到 ADRF 實例中，僅供直接 HTTP 下載。 |
| **`eventNotifs.validityPeriod`** | Object | 無 | **尚未實作** | 模型適用之有效時間區間在 OpenDaisy 中尚未實作。 |
| **`eventNotifs.spatialValidity`** | Object | 無 | **尚未實作** | 模型適用之地理空間有效範圍在 OpenDaisy 中尚未實作。 |
| **`eventNotifs.modelUniqueId`** | String | `task_id` | **部分支援** | 回傳的 `task_id` 即為模型的唯一識別碼。 |
| **`eventNotifs.modelProviderId`** | String | 無 | **尚未實作** | 尚未實作模型提供者（如 FL Server 或 AF 的 NF Instance ID）的宣告。 |
| **`eventNotifs.modelUpdateInd`** | Boolean | 無 | **尚未實作** | 尚未實作表示該模型為 retraining 更新版的指示器。 |
| **`eventNotifs.vflTrainStatus`** | Enum | 無 | **尚未實作** | 垂直聯邦學習（VFL）的訓練狀態追蹤在 OpenDaisy 中尚未實作。 |
| **`eventNotifs.addModelInfo`** | Object | `MODEL_META` in `config.json` | **部分支援** | 3GPP 要求包含數據統計資訊（均值、標準差、範圍等）。OpenDaisy 目前僅在打包檔的 `config.json` 中附帶了基本的網絡層結構元數據。 |

---

## 4. 模型準確度監控與再訓練對照 (TS 23.288 6.2E)

TS 23.288 6.2E 節定義了 MTLF 必須基於新收集的推理數據監控模型準確度，並在退化時觸發再訓練：

| 3GPP 標準功能特性 | OpenDaisy 實作做法 | 支援狀態 | 目前實作與差距說明 |
| :--- | :--- | :--- | :--- |
| **`mLModelMonitor`** | 無 | **尚未實作** | 沒有 API 端點或設定項來定義指標監控的回報模式（如週期性或閥值越界）。 |
| **`mLModelAccuracyThreshold`** | 無 (策略內部控制) | **尚未實作** | OpenDaisy 策略（如 [custom_strategy.py](file:///home/king25158986/OpenDaisy/examples/07_MTLF_training/custom_strategy.py)）內建 Early Stopping 機制，但該門檻無法透過 API 進行外部動態訂閱與調整。 |
| **`mLModelAccuracyInfo`** | Prometheus 端點 `/metrics` | **部分支援** | OpenDaisy 提供 GET `/metrics?task_id=...` 供 Prometheus 拉取準確度等指標，但無法自動在 Notify Callback 中主動推送標準的準確度狀態資訊。 |
| **基於 ADRF 數據觸發 Retraining** | 本地檔案或 MongoDB 更新 | **部分支援** | OpenDaisy 支援載入新資料進行訓練，但非透過標準 ADRF 的 API 查詢機制（例如呼叫 `Nadrf_MLModelManagement_RetrievalRequest`）。 |

---

## 5. 總結與改進建議

1. **API Schema 標準化**：OpenDaisy 使用的是扁平且自訂的 JSON 格式（如 `TID`, `CALLBACK_URL`），未來若要對接 3GPP 系統，應在 [server_api_handler.py](file:///home/king25158986/OpenDaisy/src/py/daisyfl/master/server_api_handler.py) 加上適配器層（Adapter Layer），將標準的 `NwdafMLModelProvSubsc` 結構解包並對應至 OpenDaisy 內部的策略與參數。
2. **新增生命週期控制**：增加 `DELETE /subscriptions/{subscriptionId}` 的 API 端點，使 AnLF 可以主動中止正在背景運行的訓練任務。
3. **擴展模型產物打包元數據**：在產生的 `config.json` 中，除了網路架構之外，補上模型空間、時間有效性及特徵數據統計資訊（`addModelInfo`），使其更符合 MTLF 的標準輸出格式。
