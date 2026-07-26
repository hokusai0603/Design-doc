# Go-based NWDAF MTLF 與 OpenDaisy 整合分析及差距設計文檔

本文件分析了 Go 語言實作的 5G 核心網網元 **NWDAF MTLF**（位於 [5G_Infrastructure/NWDAF/NWDAF](file:///home/king25158986/5G_Infrastructure/NWDAF/NWDAF/)）與 Python 實作的 **OpenDaisy** 聯邦學習後端引擎之間的協同工作原理，並整理出在**兩者對接整合**時所產生的功能差距。

---

## 1. 現有對接架構與流程

在現有的系統中，Go 語言的 NWDAF 扮演了 **3GPP 控制面代理 (Front-end)**，而 OpenDaisy 則作為 **聯邦學習計算後端 (FL Engine)**：

```
[ AnLF / 外部消費者 ] 
       │ (標準 3GPP HTTP API)
       ▼
┌─────────────────────────────────────────────────────────┐
│ Go NWDAF (SBI Server)                                   │
│  ├─ api_mlmodelprovision.go (處理標準訂閱 / 註冊)          │
│  ├─ adrf_retrieval.go (對接 ADRF 獲取數據與上傳)           │
│  └─ training.go (調度任務、接收 callback 並執行模型熱插拔)  │
└──────────────────────────┬──────────────────────────────┘
                           │ (非標準 RESTful API: /publish_task 與 /upload_data)
                           ▼
┌─────────────────────────────────────────────────────────┐
│ OpenDaisy Master (Python)                               │
│  ├─ server_api_handler.py (提供任務接收 API)              │
│  └─ daisyfl (聯邦學習 Master/Clients 引擎)               │
└─────────────────────────────────────────────────────────┘
```

### 1.1 主要對接點：
1. **訂閱建立**：當 [api_mlmodelprovision.go:HandleCreateMLModelProvisionSubscription](file:///home/king25158986/5G_Infrastructure/NWDAF/NWDAF/internal/sbi/api_mlmodelprovision.go#L120) 收到訂閱時，它會將 `subscriptionId` 作為任務的 `TID`，呼叫 [training.go:SubmitDaisyTask](file:///home/king25158986/5G_Infrastructure/NWDAF/NWDAF/internal/mtlf/training.go#L256) 發送 `/publish_task` 給 OpenDaisy，並帶入回呼 URL `/mtlf/training-complete`。
2. **ADRF 數據對接**：如果啟用 ADRF，[adrf_retrieval.go](file:///home/king25158986/5G_Infrastructure/NWDAF/NWDAF/internal/mtlf/adrf_retrieval.go) 會向 ADRF 訂閱並獲取數據 batch，並呼叫 OpenDaisy 的 `/upload_data` API 將數據上傳並存入 MongoDB，最後才觸發訓練。
3. **訓練回呼與熱插拔**：OpenDaisy 訓練完成後，POST 通知 Go NWDAF 的 `/mtlf/training-complete`。Go 端的 [training.go:HandleTrainingComplete](file:///home/king25158986/5G_Infrastructure/NWDAF/NWDAF/internal/mtlf/training.go#L158) 會在本地 AnLF 中進行模型熱插拔 (`swapModelAfterRetrain`)，並向原始消費者發送標準的 3GPP `NwdafMLModelProvNotif` 通知。

---

## 2. 整合後的關鍵功能差距 (Integration Gaps)

雖然 Go NWDAF 本身實作了 3GPP 的標準 API 接口，但在**與 OpenDaisy 聯動時**，因為 OpenDaisy 後端功能的缺失，導致以下 3GPP 規範動作無法完整執行：

### 2.1 訂閱刪除/取消未傳播 (Unsubscribe / Task Cancellation Gap)
* **現象**：當 AnLF 呼叫 DELETE 取消訂閱時，Go 端的 [api_mlmodelprovision.go:HandleDeleteMLModelProvisionSubscription](file:///home/king25158986/5G_Infrastructure/NWDAF/NWDAF/internal/sbi/api_mlmodelprovision.go#L178) 會在 Go 本地 Context 中刪除該訂閱，**但完全不會通知 OpenDaisy**。
* **差距**：OpenDaisy 目前沒有停止背景任務的 API。因此，在 OpenDaisy Master/Clients 運作的聯邦學習任務仍會繼續執行直到結束，白白浪費網絡與運算資源。
* **解決方案**：OpenDaisy 的 [server_api_handler.py](file:///home/king25158986/OpenDaisy/src/py/daisyfl/master/server_api_handler.py) 必須提供 `DELETE /tasks/{tid}` 接口，而 Go MTLF 應在處理 Delete 時同步向 OpenDaisy 送出中止任務請求。

### 2.2 訂閱更新未傳播 (Subscription Update Gap)
* **現象**：當 AnLF 呼叫 PUT 更新訂閱時，Go 端的 [api_mlmodelprovision.go:HandleUpdateMLModelProvisionSubscription](file:///home/king25158986/5G_Infrastructure/NWDAF/NWDAF/internal/sbi/api_mlmodelprovision.go#L153) 僅更新了 Go 內部的 context。
* **差距**：新的任務配置（例如：消費者更新了 `NUM_ROUNDS`、精度門檻或回報間隔）無法同步傳遞給已經在背景執行的 OpenDaisy 任務。
* **解決方案**：OpenDaisy 需要支援動態更新執行中任務配置的機制，或提供任務重啟的機制。

### 2.3 精度門檻監控與 Retraining 的動態連動 (Accuracy Threshold Loop Gap)
* **現象**：Go 端的 [training.go:startRetrainWorkflow](file:///home/king25158986/5G_Infrastructure/NWDAF/NWDAF/internal/mtlf/training.go#L89) 可因 AnLF 監測到模型精度退化而觸發 retraining。
* **差距**：消費者在訂閱時指定的 `mLModelAccuracyThreshold` 參數僅保留在 Go 端的記憶體中。OpenDaisy Master 在進行每一輪評估時，無法直接獲取這個動態門檻值，也無法在訓練中途若精度已達標時提前結束訓練並回報。
* **解決方案**：應將 `mLModelAccuracyThreshold` 傳遞給 OpenDaisy，並在 OpenDaisy 策略（Strategy）中動態調整 Early Stopping 的門檻。

### 2.4 模型打包元數據不一致 (Model Packaging Metadata Out of Sync)
* **現象**：OpenDaisy 訓練結束後會執行 `_package_artifacts`，打包並寫入 `config.json` 包含模型架構與基本預測配置。
* **差距**：打包程序是由 OpenDaisy 獨立完成，它無法得知 Go NWDAF 訂閱上下文中的標準 3GPP 欄位（例如：空間有效性 `spatialValidity`、時間有效性 `validityPeriod` 以及 ADRF 數據來源資訊 `addModelInfo`），導致最後產出的模型壓縮檔元數據不完整。
* **解決方案**：Go NWDAF 在觸發訓練或 callback 時，應將 3GPP 上下文元數據傳遞給 OpenDaisy，使其能夠將完整的標準元數據寫入 `config.json` 並封裝進模型產物包。
