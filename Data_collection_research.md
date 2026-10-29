# ADAE / AIMLE 資料收集效率研究：成因、所需元件、FL 介入點與改良方向

- 日期：2026-10-29
- 依據規格（本機 `spec/` 資料夾）：
  - TS 23.436 V20.2.0（ADAE）
  - TR 23.700-36 V18.1.0（ADAE 研究報告）
  - TS 23.482 V20.3.0（AIMLE）
  - TS 23.288 V20.2.0（NWDAF/DCCF/ADRF）
  - TS 23.433 V20.4.0（SEALDD）
- 標記方式：**【規格】** 表示規格明文；**【推論】** 表示我根據規格做的分析或主張；**【文獻】** 表示網路上的研究或報導。

---

## 0. 摘要

1. **低效率的根本原因不是「缺少一個更聰明的演算法」，而是應用層的資料收集框架少了幾塊元件**，主要有四項：
   - **目標導向的收集規劃**：現在是 Analytics ID 到收集事件的靜態對應。
   - **資料減量工具**：5GC 的 DCCF 已經有，ADAE 的 A-DCCF 沒有。
   - **資料品質與價值的描述**：兩份規格都完全沒有。
   - **ADAE 與 AIMLE 之間的資料平面銜接**：AIMLE 觸發 A-DCCF 的方式還是 Editor's Note。
2. 要提升效率，**最可能需要的元件依序是**：
   1. 收集規劃器（Collection Planner，可以由 agent 擔任）；
   2. 擴充後的 Data Producer Profile 與資料目錄（Dataset Catalog）；
   3. A-DCCF 的減量指令集（移植 5GC TS 23.288 5A.4、6.2.6.1、6.2.7）；
   4. 「端側運算」模式，也就是 Federated Analytics；
   5. 從模型或分析效能回到收集策略的閉環控制。
3. **FL 有明確的介入空間，但介入點不只是「訓練」**。最有價值的是：
   - **(a)** ADAE 的統計型分析改用 **Federated Analytics**。AIMLE 8.15 的 client 端 data analysis 機制已經是 FA 的雛形，ADAE 卻沒有用到。
   - **(b)** ADAES 以 AF 身分和 NWDAF 做 **VFL**（TS 23.288 6.2H 已支援），避免把網路資料搬到應用層。
   - **(c)** 用 FL 學習「收集策略本身」。
   - FL 不適用的情況：需要跨 UE 關聯的分析、資料量小的一次性分析、標籤只在 server 端的任務。
4. **改良路線分三層**：
   - L1：與標準相容的補強，例如把 5GC 機制移植過來、補上 IE 定義；
   - L2：架構增強，例如 Planner、Catalog、FA 模式、閉環控制；
   - L3：研究與 6G，例如 agent 規劃、VoI/bandit 取樣、跨域統一資料框架。
   - **研究題目建議以 L2 + L3 的「agent 規劃、標準功能執行」為主軸**，用 L1 當 baseline。

---

## 1. 現況回顧（細節見 `ADAE/`、`AIMLE/` 中的整理）

| | ADAE | AIMLE |
|---|---|---|
| 收集模式 | 集中式：ADAES 依 Analytics ID 對應到 Data Collection Event ID 和 Data Producer ID，再訂閱或請求（TS 23.436 8.2.2 step 3–4） | 資料不動：用 Dataset ID（必填）指定 client 端資料，派送模型與處理函式（TS 23.482 8.3、8.12、8.15） |
| 協調與儲存 | A-DCCF（8.12）、A-ADRF（8.10） | 無專屬收集協調實體；ML repository 存模型與成員資訊 |
| UE 端資料 | ADAE client，重用 EVEX（TS 26.531）（5.3 NOTE） | 經 EVEX 收集後分配 dataset ID（8.15 前置條件） |
| 對方的角色 | ADAES 透過 AIML-X 消費 AIMLE 做訓練（5.2.4） | AIMLE 消費 ADAES 的分析作為輔助資訊（8.14、8.16 等） |

---

## 2. 問題成因分析

### RC1：收集計畫是「靜態對應」，不是「目標導向」

- **【規格】** ADAES 把 Analytics event ID 對應到資料收集事件與資料生產者，這組對應「may be preconfigured by OAM or may be determined by ADAES based on the analytics event type / vertical type and/or data producer profile」（TS 23.436 8.2.2 step 3；8.16.2.1 step 3 也一樣）。**決定方式完全沒有規範。**
- **【規格】** 研究階段已經指出，消費者通常不知道該收哪些資料，所以 ADAE 必須能挑選最佳來源（TR 23.700-36 KI#3，4.3）。正規化時只留下「對應」這一步，沒有定義怎麼選。
- **【推論】** 結果是實務上很可能採用 OAM 預設的「寬鬆對應」：為了保險，把所有可能有用的事件都訂閱起來。收集量和分析目標之間沒有回饋關係，也沒有依「分析準確度要求」或「信心水準」調整收集的機制。

### RC2：A-DCCF 缺少 5GC DCCF 已有的「減量工具」

- **【規格】** 5GC 的 DCCF 設計目的明寫是避免資料來源「handle multiple subscriptions for the same data and send multiple notifications containing the same information」（TS 23.288 5A.1）。為此它提供了一整組減量工具。
- **【規格】** ADAE 的 A-DCCF 雖然有去重複（TS 23.436 8.12.2.1 step 3–4），但只有一個**沒有定義內容的 `Process requirements` IE**：「Requirements on processing the collected data/analytics」（Table 8.12.3.2-1）。

| 機制 | 5GC（TS 23.288） | ADAE（TS 23.436） |
|---|---|---|
| 訂閱去重複、合併 | DCCF（5A.2） | A-DCCF（8.12） |
| **格式化**：event clubbing、時間窗、跨事件觸發、consumer 觸發的 fetch、固定時間、遞增週期 | 5A.4 | **無** |
| **摘要處理**：在 processing interval 內計算平均、變異數、偏度、最大最小值、次數等，用摘要報告取代逐筆事件 | 5A.4 | **無**（只有未定義的 Process requirements） |
| **Bulked data**：依 Analytics ID 打包 raw 或 pre-processed 樣本，**明確區分給 training 還是 inference 用** | 6.2.6.1 | **無** |
| **Event muting**：生產端先暫存，消費者需要時再取 | 6.2.7 | **無** |
| 訊息框架（MFAF）、fetch 間接傳遞 | 5A.3 | **無** |
| 收集頻率控制 | 以上機制都可用 | ADAE client「periodically or if a threshold is reached」（8.2.2 step 8b），由實作決定 |

我用關鍵字搜尋 TS 23.436 和 TS 23.482，「formatting」「muting」「bulk」「MFAF」「clubbing」都沒有出現。

- **【推論】** 這是**最容易論證、最接近標準化**的缺口。5GC 已經證明這些機制有用，應用層卻沒有對應的機制。

### RC3：沒有「資料品質與資料價值」的概念

- **【規格】** 兩份規格都沒有出現「data quality」。TS 23.436 的 data producer profile 只描述儲存期限、是否支援匿名化、資料產生速率與時程（TS 23.436 中 data producer profile 相關表格）。**沒有準確度、覆蓋率、偏差、新鮮度、與目標任務的相關性等欄位。**
- **【規格】** 研究階段有提到「同一指標的多來源資料先正規化與驗證，剔除不一致樣本」（TR 23.700-36 6.3.1.2.3 step 4），但沒有進入 TS。
- **【規格】** AIMLE 選 FL 成員的條件是能力、可用時間、資料量與特徵對齊（TS 23.482 8.18.2 step 9、8.19）；資料集可用性只描述「大小、新舊、特徵、ID」。**沒有「這份資料對模型有多少貢獻」的衡量。**
- **【文獻】** FL 文獻已經有 Shapley、gradient-based、Wasserstein 等資料價值與 client 貢獻評估方法（arXiv 2504.16668、2406.00569、2311.05304）；考慮資料品質的無線 FL client 選擇也有研究（arXiv 2208.04322）。但都沒有對應到 3GPP 的介面。
- **【推論】** 沒有價值的衡量，就無法在收集量與效益之間做取捨，只能「多收比較安全」。

### RC4：ADAE 與 AIMLE 的資料平面是分開的

- **【規格】** AIMLE 收集模型效能資訊時，可以「directly or via A-DCCF data collection service」，但「How the AIMLE server can trigger A-DCCF data collection service will be further discussed during normative phase」（TS 23.482 8.31 step 3，Editor's Note）。
- **【規格】** ML repository 的模型資訊會記錄「訓練用了哪些資料（data sources, volume, freshness）」（TS 23.482 8.11 相關 IE），但**這些資訊和 A-ADRF 的資料集之間沒有任何關聯機制**。AIMLE 的 Dataset ID 也沒有定義和 ADAE 的 Data Collection Event ID 之間的關係。
- **【推論】** 兩者都從 UE 經 EVEX 收集資料，又各自維護資料集。同一個 UE 的同一類資料，可能被 ADAE（為了分析）和 AIMLE（為了訓練）**各收一次**。資料的血緣（lineage）也斷了：模型劣化時（TS 23.436 8.19、TS 23.482 8.22），沒辦法回頭追是哪一批資料造成的，也沒辦法只補收缺少的部分。

### RC5：UE 端收集的成本、能耗與隱私沒有納入收集決策

- **【規格】** UE 的資料收集需要使用者同意（TS 23.436 8.16 NOTE 1；TS 23.482 中也有 user consent 的 NOTE）。匿名化在 producer profile 中只是一個能力旗標。
- **【規格】** SEALDD 已經有 Background Data Transfer，可以在非尖峰時段以較低成本傳輸（TS 23.433 4.7、9.11），也有自適應傳輸（9.13，會借助 AIMLE）。**但 ADAE 和 AIMLE 的收集流程都沒有引用 BDT 來排程大量上傳。**
- **【規格】** ADAE 已經有「AIMLE client energy sustainability analytics」（TS 23.436 8.23）和 AI/ML 能耗分析（8.22），但這些是**給 AIMLE 選成員用的，不是給收集規劃用的**。
- **【推論】** UE 端的取樣率、上傳時機、是否先在端側摘要，目前都是實作決定，沒有一個元件同時考慮能耗、成本和隱私。

### RC6：跨層、跨域沒有統一的資料框架

- **【文獻】** Samsung 指出 5G 的問題在於「lack of unified data collection and transfer mechanism among different domains within 3GPP」。6G 的資料框架希望統一 SA2（核心網）、RAN1/2（無線）、SA5（管理）的資料收集與傳遞（Samsung Research blog）。
- **【文獻】** 6G 的 SA2 研究（TR 23.801-01）有 24 個 key issue，要求把「model and data handling」作為架構服務。CT 也有「unified core data framework」的研究項目；SA6 的 6G 研究是 TR 23.801-02（arXiv 2610.05694，狀態快照日期為 2026-06-30）。
- **【推論】** ADAE 等於在應用層「複製」了一套 NWDAF 式的框架（ADAES 對 NWDAF、A-DCCF 對 DCCF、A-ADRF 對 ADRF），但只複製了一部分（見 RC2）。這兩套框架之間也只能透過 NEF 與 AF 介面互通。6G 是重新設計的時機。

### RC7：收集與模型效能之間是開環

- **【規格】** 模型劣化偵測與重訓已經有規範（TS 23.436 8.19、8.20；TS 23.482 8.21、8.22），觸發的動作是「重訓、換模型、換成員」（TS 23.482 8.22 step 6）。**但沒有任何「調整收集計畫」的動作**，例如補收哪些特徵、提高哪一區的取樣率、停收哪些無效的事件。
- **【規格】** AIMLE 8.15 已經能做 **data drift detection**（client 端執行，TS 23.482 8.15），TR 23.700-83 也在強化這部分。但偵測結果只回報給消費者，沒有連回收集端。
- **【推論】** 偵測漂移所需的訊號都已經存在，缺的是「把這些訊號轉成收集策略」的元件。

### 成因總覽

```
           (RC6 跨域不統一)
                 │
 分析/訓練需求 ──► [RC1 靜態對應] ──► 訂閱很多來源 ──► [RC2 無減量工具] ──► 大量逐筆通知
                 ▲                        │                                   │
                 │                  [RC3 無品質/價值]                    [RC5 UE 成本/隱私]
                 │                        │                                   │
                 └──[RC7 開環]── 模型/分析效能 ◄── [RC4 ADAE/AIMLE 資料平面割裂，重複收集]
```

---

## 3. 要更有效率，需要哪些元件

依「對效率的影響程度」與「目前缺口的大小」排序。放置位置是【推論】。

| # | 元件 | 解決的成因 | 建議位置 | 功能 |
|---|---|---|---|---|
| **C1** | **收集規劃器（Collection Planner）** | RC1、RC3、RC7 | ADAES 內部，或 A-DCCF 的前端；AIMLE 透過 AIML-X 共用 | 輸入分析或訓練目標（Analytics ID 或 ML task、準確度與信心要求、區域與時間）、資料生產者目錄、成本與隱私限制；**輸出收集計畫**：收哪些事件、找誰收、取樣率、處理指令、送往哪裡、用 FA 還是集中收集。**這是 agent 最適合擔任的角色** |
| **C2** | **擴充的 Data Producer Profile 與資料目錄（Dataset Catalog）** | RC3、RC4 | A-ADRF 與 ML repository 共用的 metadata 層 | 在 profile 中加入品質欄位（準確度、覆蓋率、新鮮度、缺值率、偏差）；資料集登錄 lineage（由哪些 Data Collection Event 產生、被哪些 ML model ID 用過）；**讓 AIMLE 的 Dataset ID 和 ADAE 的資料能互相對應** |
| **C3** | **A-DCCF 減量指令集** | RC2 | A-DCCF，必要時加上應用層的 messaging 元件 | 移植 TS 23.288 5A.4 的格式化與摘要處理、6.2.6.1 的 bulked data（**區分 training 與 inference**）、6.2.7 的 muting；**正式定義 `Process requirements` 的內容** |
| **C4** | **端側運算與 Federated Analytics 模式** | RC5、RC2 | ADAE client，重用 AIMLE client 的 8.15 機制 | 讓 ADAES 能把「統計查詢」派給 client 在本地計算，只回傳摘要。見第 4 節 |
| **C5** | **收集閉環控制器** | RC7 | 與 C1 合併，或作為 C1 的觸發來源 | 訂閱 TS 23.436 8.19、8.20 和 TS 23.482 8.15（drift）、8.22 的結果，觸發 C1 重新規劃 |
| **C6** | **傳輸排程整合** | RC5 | SEALDD | 大量訓練資料或 bulked data 走 BDT（TS 23.433 9.11）；即時推論資料走自適應傳輸（9.13） |
| **C7** | **ADAE 與 AIMLE 的收集介面** | RC4 | AIML-X 與 A-DCCF 之間 | 補上 TS 23.482 8.31 的 Editor's Note：讓 AIMLE 可以向 A-DCCF 訂閱，A-DCCF 再做跨 ADAE 與 AIMLE 的去重複 |
| **C8** | **隱私、同意與政策引擎** | RC5 | ADAES 或 A-DCCF | 把 user consent、匿名化、資料最小化原則變成 C1 的硬性限制條件 |

**最小可行組合**是 C1 + C2 + C3：沒有 C2 的資訊，C1 無從規劃；沒有 C3 的工具，C1 規劃好也執行不了。

---

## 4. FL 的介入空間

### 4.1 目前已經有 FL 的地方（標準已涵蓋）

| 位置 | 內容 |
|---|---|
| AIMLE HFL（TS 23.482 8.12） | client 用本地資料集訓練，server 聚合 |
| AIMLE VFL（8.18、8.18.2b） | 不同 data domain 的特徵，經 NEF 做樣本對齊 |
| NWDAF 間 HFL（TS 23.288 5.3、6.2C） | 多個 NWDAF 之間的水平 FL |
| NWDAF 與 AF 的 VFL（5.4、6.2H） | **AF 可以當 VFL server 或 client** |
| ADAE 的 FL | ADAE 自己沒有 FL，只能透過 AIMLE（TS 23.436 5.2.4） |

### 4.2 可以介入的新位置

| # | 介入點 | 做法 | 為什麼有價值 | 依據 |
|---|---|---|---|---|
| **F1** | **ADAE 統計分析改用 Federated Analytics** | ADAES 的大部分輸出是統計量，例如平均或峰值吞吐量、E2E 延遲、jitter、PER（TS 23.436 8.2.2 step 6）。這類查詢可以交給 ADAE client 在本地計算，再做安全聚合，不必上傳逐筆量測 | 直接降低 UE 上傳量與隱私風險。**AIMLE 8.15 的「data analysis」已經讓 client 在本地計算各特徵的統計量並由 server 彙整**，等於 FA 已經存在於 AIMLE，只是 ADAE 沒有接上 | 【規格】TS 23.482 8.15；【文獻】FA for 6G（arXiv 2401.03878）；【推論】把兩者接起來 |
| **F2** | **ADAE 的 ML-enabled analytics 用 HFL，不先集中收集** | ADAES 經 AIML-X 請 AIMLE 用 HFL 訓練某個 Analytics ID 的模型，資料留在 ADAE client 或 edge ADAES，不先送進 A-ADRF | 減少集中收集；適合各區資料分布不同、又不能出域的情況 | 【規格】TS 23.436 5.2.4、TS 23.482 8.12；【推論】需要把 Dataset ID 對應到 ADAE 的資料（C2） |
| **F3** | **應用層與網路層之間用 VFL，取代資料搬移** | ADAES 以 AF 身分和 NWDAF 做 VFL：應用特徵（QoE、應用負載）留在 ADAE，網路特徵（QoS、移動性）留在 NWDAF | 現在 ADAES 要經 NEF 訂閱 NWDAF 的分析或資料（8.2.2 NOTE），屬於跨域資料搬移。VFL 只交換中間結果與梯度 | 【規格】TS 23.288 5.4、6.2H（支援 AF）；【推論】TS 23.436 沒有提到這種用法，是新的組合 |
| **F4** | **用 FL 學習「收集策略」本身** | 多個 edge ADAES（TS 23.436 Annex A.3、A.4 的 edge 與協同部署）各自學習「哪些事件或取樣率對準確度有用」的策略模型，再以 FL 聚合 | 收集策略具有地區性，但又可以共享通用的規律；不需要集中各區的原始資料 | 【推論】屬於研究型方向 |
| **F5** | **考慮資料價值的 FL 成員選擇** | 把 ADAE 的 AI/ML member capability analytics（TS 23.436 8.16），加上 C2 的品質與價值資訊，作為 AIMLE 選成員（8.9、8.13）的輸入 | 少選但選對，可以減少輪數和參與 UE 數；文獻顯示 LLM 選擇器偏重訓練時間，很少看資料特性（arXiv 2604.04895） | 【規格】8.16、8.9；【文獻】2208.04322、2504.16668 |

### 4.3 FL 不適用或要小心的地方

| 情況 | 原因 |
|---|---|
| 需要跨 UE 或跨實體關聯的分析，例如 UE-to-UE 效能（TS 23.436 8.4）、碰撞偵測（8.14）、location-related UE group（8.15） | 本質上需要把多方資料對齊在一起；FA 能處理部分統計，但難以做配對關聯 |
| 資料量小、一次性的分析 | FL 的多輪通訊與協調成本，可能高於直接收集一次 |
| 標籤只在 server 端 | HFL 不適用；VFL 可以處理，但 TS 23.482 註明 VFL 的協調方式「out of scope for this release」（8.18.2 NOTE） |
| UE 算力與能耗有限 | 需要結合 TS 23.436 8.23 的能源永續分析篩選成員 |
| 非 IID 資料、成員中途掉線 | FL 的老問題，會影響收斂與公平性 |

### 4.4 選擇方式的決策原則（【推論】，可以作為 C1 規劃器的規則基礎）

| 分析或任務特性 | 建議模式 |
|---|---|
| 需要統計量、可以分解成各端的摘要 | **FA**（F1） |
| 需要 ML 模型、資料同特徵但分散在各處、不能出域 | **HFL**（F2） |
| 特徵分散在應用與網路兩域、樣本相同 | **VFL**（F3） |
| 需要逐筆或跨實體關聯、資料量小 | **集中收集**，加上 C3 減量 |
| 要求即時推論、資料在 UE 端 | **協同或 split 推論**（TS 23.482 8.35、8.14） |

---

## 5. 如何改良：三層路線

### L1：與標準相容的補強（短期，可以寫成 CR 或 3GPP 文稿）

1. **定義 A-DCCF 的 `Process requirements`**：直接移植 TS 23.288 5A.4 的 Formatting Instructions 和 Processing Instructions。
2. **新增應用層 bulked data**：把 TS 23.288 6.2.6.1 的 raw 與 pre-processed 樣本、training 與 inference 用途區分帶進 A-DCCF 和 A-ADRF 的 API。
3. **新增 ADAE client 的 muting 與 fetch 機制**（參考 TS 23.288 6.2.7）：讓 UE 先在本地暫存，由 ADAES 依需要取回。
4. **在 Data Producer Profile 中增加品質欄位**（C2 的最小版本）。
5. **補上 TS 23.482 8.31 的 Editor's Note**：定義 AIMLE 觸發 A-DCCF 收集的流程與 IE（C7）。
6. 在 ADAE 的收集流程中加上 **SEALDD BDT 的選項**（C6）。

### L2：架構增強（中期，可以作為 6G SA6 研究，TR 23.801-02 方向的提案）

1. **收集規劃器（C1）+ 資料目錄（C2）+ 閉環控制（C5）**，形成完整迴路：「目標 → 規劃 → 執行（A-DCCF）→ 評估 → 重新規劃」。
2. **ADAE 的 FA 模式（F1）**：把 AIMLE 8.15 的 client data processing 擴充成 ADAE 也能使用的統計查詢服務。
3. **ADAE 與 NWDAF 的 VFL（F3）**：定義 ADAES 作為 TS 23.288 6.2H 中 AF 角色的使用方式。
4. **ADAE 與 AIMLE 的共同資料平面**：由同一個 A-DCCF 為兩者去重複，資料目錄共享 lineage。

### L3：研究與 6G（長期）

1. **Agent 型收集規劃器**：
   - 建議的設計原則（【推論】）是 **「agent 規劃、標準功能執行」**。agent 只產生結構化的收集計畫（可驗證的 JSON 或 IE 集合），實際的訂閱、去重複和減量，仍由 A-DCCF 等標準功能確定性地執行。
   - 這樣可以回應文獻中對 agentic 協定「non-deterministic and chatty」的疑慮（arXiv 2610.05694），也便於測試與互通。
   - Agent 可用的工具：
     - 查詢資料生產者目錄（C2）；
     - 查詢 A-ADRF 已有的資料，對應 TR 23.700-36 中「先找 repository」的步驟；
     - 向 A-DCCF 下訂閱（C3 指令）；
     - 發 FA 查詢（F1）；
     - 查詢能耗與成員能力分析（TS 23.436 8.16、8.22、8.23）。
   - 參考做法：
     - 用 MCP 連接 agent 與 NWDAF 等網路工具（arXiv 2607.16066）；
     - 核心網的 tool-use agent 原型（arXiv 2605.02811）；
     - LLM 把意圖轉成 O-RAN 資料收集策略（arXiv 2605.23809）。
     - 這幾篇都還停在 NWDAF 或 O-RAN 層，**沒有人做到 SA6 應用層**。
2. **取樣與收集頻率的學習式控制**：
   - 以 VoI 或 Task-oriented AoI（arXiv 2508.14575）衡量資料價值，用 bandit 或 RL 動態調整取樣率。
   - 文獻也指出收集週期、偵測準確度、反應時間之間存在取捨（arXiv 2601.01838）。
3. **FL 學習收集策略（F4）**與**資料價值導向的成員選擇（F5）**。
4. **跨域統一資料框架**：配合 6G SA2、CT 的 unified data framework，把應用層、核心網、管理層的收集統一在同一套 catalog、規劃與執行機制下。

### 改良後的資料流示意（L2 + L3）

```
 VAL server / AIMLE ──(目標：Analytics ID 或 ML task + 準確度/成本/隱私限制)──► [C1 收集規劃器 (agent)]
                                                                                    │ 查詢
                                                     [C2 Producer Profile + Dataset Catalog] ◄──┤
                                                                                    │ 產出收集計畫
              ┌──────────────────────────────┬────────────────────┬─────────────────┘
              ▼                              ▼                    ▼
     集中收集：A-DCCF + C3 減量         FA 查詢 (F1)         FL/VFL (F2/F3)
     (clubbing/摘要/bulked/muting)      ADAE client 端計算    經 AIMLE / NWDAF
              │  大量資料走 SEALDD BDT (C6) │                    │
              └──────────────┬───────────────┴────────────────────┘
                             ▼
                   分析結果 / 訓練好的模型 ──► 效能監控 (TS 23.436 8.19/8.20, TS 23.482 8.15 drift/8.22)
                             │                                   │
                             └──────── [C5 閉環] ─── 重新規劃 ◄───┘
```

---

## 6. 作為研究題目的評估

### 6.1 結論

**是一個好題目。** 理由有三：

1. **缺口明確而且有規格依據**：RC2 和 RC4 都能直接引用條文，包括未定義的 IE 和 Editor's Note。
2. **學術界還沒有人做**：agent 與資料收集的研究停在 NWDAF 和 O-RAN；ADAE、AIMLE 幾乎沒有學術論文。
3. **時機正好**：6G SA6 研究（TR 23.801-02）還在定義範圍，研究成果可以轉成標準提案。

### 6.2 建議的題目定位

> **「面向應用層分析與 AI/ML 的目標導向資料收集：以 agent 規劃、標準功能執行，整合集中收集、Federated Analytics 與 FL」**

貢獻可以分三塊：

1. 指出並形式化 RC1 到 RC7 的問題，以及 L1 的標準補強；
2. C1、C2、C5 的架構設計，加上 FA/FL/集中收集的選擇機制（第 4.4 節）；
3. 以 agent 實作 C1，並和 baseline 比較。

### 6.3 評估方式

| 項目 | 建議 |
|---|---|
| **Baseline** | (B0) OAM 靜態對應、全量訂閱；(B1) B0 + A-DCCF 去重複（現行 TS 23.436）；(B2) B1 + 移植的 5GC 減量工具（L1）；(B3) 規則式規劃器；(B4) RL 或 bandit 規劃器；**(Ours)** agent 規劃器 + FA/FL 選擇 |
| **效率指標** | 信令與通知數、上傳位元組數（UE 端與 DN 端分開）、UE 能耗、A-ADRF 儲存量、重複收集比例 |
| **效益指標** | 分析準確度與信心水準、模型準確度、資料新鮮度、從請求到產出分析或模型的時間 |
| **Agent 專屬指標** | 規劃的正確率（計畫能否通過政策與同意檢查）、token 和延遲成本、在新的 Analytics ID 上的泛化能力 |
| **情境** | VAL server 效能分析（TS 23.436 8.2）、edge load（8.7）、ADAE 與 NWDAF 的 VFL（F3）、模型劣化後補收資料（RC7） |
| **實驗平台** | 我**沒有找到開源的 ADAE 或 AIMLE 實作**。可行的方式：以 free5GC 加開源 NWDAF（arXiv 2606.11877、2601.01838）作為網路側，**自建輕量的 ADAES、A-DCCF、A-ADRF 模擬層**，FL/FA 用 Flower 等框架 |

### 6.4 風險與開放問題

- **「為什麼要用 agent？」**：必須證明 agent 在「開放式需求、異質來源描述、新的 Analytics ID」上，勝過規則式和 RL。建議把 B3 和 B4 做扎實。
- **自建平台的工作量**：這可能是最大的成本，但也可以當成一項貢獻（開源的 ADAE 模擬器）。
- **評估資料的真實性**：應用層 QoE 與網路 QoS 的聯合資料集不多，可能需要合成資料或自行量測。
- **標準時程不確定**：6G SA6 的範圍要到 2026 年 12 月全會才會核准（ATIS 簡報），研究的定位要能同時適用於 5G-Advanced（L1）和 6G（L2、L3）。
- **Agent 的安全與可控性**：計畫必須能被驗證，同意與政策檢查要由確定性的元件執行，不能交給 agent 判斷。

---

## 7. 參考資料

### 3GPP 規格（本機）
- TS 23.436 V20.2.0：5.2.4、5.3、5.5、8.2.2、8.2.4、8.10、8.12、8.16、8.19、8.20、8.22、8.23、Annex A
- TR 23.700-36 V18.1.0：4.3（KI#3）、5.1.3、6.3（Solution #2）、9.2
- TS 23.482 V20.3.0：8.3、8.9、8.11、8.12、8.14、8.15、8.18、8.19、8.22、8.31、8.35、8.36
- TS 23.288 V20.2.0：5.3、5.4、5A.1、5A.2、5A.4、6.2.6.1、6.2.7、6.2.8、6.2C、6.2H
- TS 23.433 V20.4.0：4.4、4.6、4.7、9.11、9.13

### 標準相關（網路）
- TR 23.700-83 AIMLE Phase 2（whatthespec）：https://whatthespec.net/friendlyspec/spec/23.700-83/20.1.0
- TEI19 ADAES work item：https://whatthespec.net/3gpp/wi/1070004-tei19-adaes
- 3GPP Rel-19 Summary（2026-04）：https://www.3gpp.org/ftp/Information/presentations/presentations_2026/2026_04_Rel19.pdf
- TR 23.801-01 status：https://www.3gpp.org/dynareport/23801-01.htm
- 3GPP Release 20：https://www.3gpp.org/specifications-technologies/releases/release-20
- ATIS 3GPP Release 20 webinar：https://cdn.atis.org/atis.org/2026/04/16110914/Combined-Slides_3GPP-Webinar-R20_2026.pdf
- Samsung Research, AI in 6G network – Service and System Aspect：https://research.samsung.com/blog/AI-in-6G-network-Service-and-System-Aspect

### 學術文獻（多為 arXiv 預印本，尚未經同儕審查）
- 6G as It Is Actually Being Built（arXiv 2610.05694）：https://arxiv.org/html/2610.05694v1
- LLM-Powered Agentic AI for 5G/6G Networks survey（arXiv 2607.16066）：https://arxiv.org/html/2607.16066v1
- Tool Use as Action: Towards Agentic Control in Mobile Core Networks（arXiv 2605.02811）：https://arxiv.org/abs/2605.02811
- Advanced AI Service Provisioning in O-RAN through LLM Engine Integration（arXiv 2605.23809）：https://arxiv.org/html/2605.23809
- LLM-Enabled NWDAF（arXiv 2606.11877）：https://arxiv.org/abs/2606.11877
- Enhanced Open-Source NWDAF for Event-Driven Analytics（arXiv 2601.01838）：https://arxiv.org/pdf/2601.01838
- Federated Analytics for 6G Networks（arXiv 2401.03878）：https://arxiv.org/abs/2401.03878
- Federated Learning for 6G: Paradigms, Taxonomy（arXiv 2312.04688）：https://arxiv.org/abs/2312.04688
- Agentic Federated Learning（arXiv 2604.04895）：https://arxiv.org/pdf/2604.04895
- Data quality-aware client selection for multiple FL services（arXiv 2208.04322）：https://arxiv.org/abs/2208.04322
- Efficient Data Valuation Approximation in FL（arXiv 2504.16668）：https://arxiv.org/pdf/2504.16668
- Redefining Contributions: Shapley-Driven FL（arXiv 2406.00569）：https://arxiv.org/html/2406.00569v1
- Data Valuation and Detections in FL（arXiv 2311.05304）：https://arxiv.org/html/2311.05304v3
- Minimizing Task-Oriented Age of Information（arXiv 2508.14575）：https://arxiv.org/pdf/2508.14575
- Goal-Oriented Integration of Sensing, Communication, Computing and Control（arXiv 2312.16064）：https://arxiv.org/pdf/2312.16064
- Towards an AI/ML-driven SMO Framework in O-RAN（arXiv 2409.05092）：https://arxiv.org/html/2409.05092v1
