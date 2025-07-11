
# 📘 實驗名稱：分析逐年書籍資料對統計式推薦系統分類效能之影響

---

## 一、研究目標與問題定義

- **研究問題**：新書資料每年加入統計式推薦系統後，是否對系統整體分類結果的準確度產生變化？若有，影響程度為何？
- **研究目標**：
  - 觀察並量化逐年新書資料對統計式推薦系統分類表現的影響。
  - 建立「出版年 vs 分類表現」的趨勢關係圖，為 LLM 模型建立基準與補強依據。

---

## 二、假設設計

- **假設 H₀**：每年的新書資料加入後，不會顯著提升分類推薦準確率。
- **假設 H₁**：逐年新書資料會顯著影響統計式推薦系統的分類準確率。
- **自變數**：訓練資料出版年度區間（2007 ~ 2016）
- **應變數**：統計式推薦系統對當年度新書的分類準確率（Micro-F1, Macro-recall, Jaccard）
- **控制變數**：斷詞策略 ( Bi-gram )、BM25 參數、推薦邏輯（再版規則、關聯規則）

---

## 三、資料來源與特性

- **來源**：國圖 Aleph MARC21 系統；資料欄位：書名、作者、出版社、分類號（089）、主題詞（699）、OWN（人工編目結果）
- **資料區間**：
    - 資料欄位上的年份取得方式 `唐仁壕論文` 特別說明「應使用 MARC 格式中固定欄位 008 的第 7–10 位來取代 260$c 欄位的出版年」，因為後者在舊書中格式不一，有時用民國年，有時有月份
  - 訓練集: 出版年份為 2007~2016（10年資料）
  - 測試集: 出版年份為 2017~2025 每年資料（逐年測試）
- **資料篩選條件**：
  - 語言為繁體中文（lang=chi）、英文 ( 以單字切分 )
  - 排除：學位論文、目錄、分篇等非一般圖書類型

---

## 四、實驗流程設計

1. **初始模型訓練**（2007~2016 為基底）  
   - 使用統計式推薦系統（BM25 + 關聯規則 + 再版比對）進行模型初始化。

2. **逐年分析迴圈（2017~2025）**
   - 對每一個年份 y：
     - 使用 2007~2016 + 2017~y−1 資料重新訓練模型
     - 使用該年度（y）的書籍作為測試集
     - 比對分類結果與人工編目，計算指標：
       - 分類號：Micro-F1
       - 主題詞：Macro-recall、Jaccard 指數
     - 記錄變化趨勢 ΔF1(y), ΔRecall(y), ΔJaccard(y)

3. **趨勢分析與視覺化**
   - 每年指標作為時間序列資料
   - 繪製：
     - 出版年 vs 分類號 F1 變化圖
     - 出版年 vs 主題詞 recall 變化圖
     - 出版年 vs Jaccard 一致性分析

---

## 五、實驗指標與檢定方法

| 分析目標 | 指標 | 檢定方法 |
|----------|------|----------|
| 分類精度變化 | Micro-F1 | ANOVA 或時間序列回歸 |
| 主題詞召回 | Macro-recall | Friedman test（多年份比較） |
| 類號與主題詞一致性 | Jaccard coefficient | 趨勢檢定（Spearman、Kendall） |

---

## 六、資料處理與重現性設計

- **資料處理程序**：
  - ISO2709 → MARC21 → MARCXML → PostgreSQL 清理匯入
  - 特徵擷取：書名（Bi-gram）、作者／出版社（整段比對）
- **記錄與重現性**：
  - 資料清洗腳本版本控管（Git）
  - 訓練流程參數化、紀錄各年份 F1 trend
  - 使用 DVC / MLflow 管理模型與結果
- **部署環境**：
  - Docker 容器封裝推薦系統（Ubuntu + Flask）

---

## 七、預期結果與應用

- 預期觀察逐年新增資料是否穩定提升推薦效能（若呈現邊際效益遞減，表示模型已飽和）
- 提供後續 LLM 替代設計基準：哪些年份資料有關鍵性貢獻
- 輔助 LLM 訓練資料切分與加權設計（如 fine-tune 階段的樣本選擇）


---

## 📘 子實驗：主題詞下各分類號分類準確度分析（逐年）

### 一、研究目的與問題

- **研究問題**：針對某一主題詞（如：`小說`），不同分類號在統計式推薦系統的準確度是否存在顯著落差？其潛在原因是什麼？
- **研究目標**：
  - 在每個年度，選定主題詞（或數個代表性主題詞），觀察其下各分類號的準確率。
  - 分析造成分類準確率差異的潛在因素，如樣本數不足、分類定義模糊、書籍資料特徵不足等。

### 二、分析對象與資料範圍

- **主題詞選擇策略**：
  - 優先選擇年度內樣本數量最多的前N大主題詞（如：科學、歷史、宗教、語言）
  - 或手動選定特定主題領域（如社會科學、生命科學）

- **資料來源**：
  - 含有 `OWN` 欄位（人工編目結果）與統計式推薦分類結果的書目資料
  - 以 `年` 為單位進行分析，使用2017~2025年逐年出版資料

### 三、實驗流程設計

1. **主題詞 → 書目過濾**
   - 以主題詞為查詢條件，篩出每年度中屬於該主題詞的書目（可依 `699$a` 欄位匹配）

2. **分類號群組**
   - 統計該主題詞下所出現的分類號（如 857.7、861.57、863.57...）
   - 記錄每個分類號：
     - 書籍總數（人工編目判定）
     - 推薦命中數（統計式系統預測正確）

3. **準確率計算**
   ```
   準確率 = 正確分類數 / 該分類號在主題詞下的總書籍數
   ```

4. **結果記錄與排序**
   - 每年、每個主題詞，輸出：
     - 分類號、總書數、命中數、準確率、樣本是否充足（如<3本記為低信度）
     - 可產出熱力圖或條形圖觀察準確度分布

5. **誤分類書籍樣本分析**
   - 挑選準確度偏低的分類號（如低於50%）
   - 抽樣該分類號的誤分類樣本，觀察其：
     - 書名是否過於模糊
     - 作者或出版社資訊是否不具辨識力
     - 書籍是否與其他分類號高混淆（可參考分類號混淆矩陣）

### 四、結果呈現與詮釋

- **視覺化**：
  - 年度 × 主題詞 × 分類號為三維觀察點
  - 使用：
    - 條形圖顯示每分類號準確率
    - 熱力圖或色階地圖標示準確率落差
    - 混淆矩陣輔助解釋誤分類趨勢

- **詮釋方向**：
  - 是否為樣本數太小導致不穩定？
  - 該分類號是否與其他分類號內容上容易混淆？
  - 書籍標題是否未出現關鍵詞？
  - 作者或出版社是否在資料庫中出現次數過少？

### 五、應用與後續延伸

- 可以針對準確率偏低的分類號，嘗試引入額外特徵或對LLM設計提示（如加上內容摘要）
- 為後續 fine-tune 階段設計新的加權策略（針對誤分類率高的主題詞與類號給予強化訓練）
- 作為 LLM prompt tuning 評估的偏誤監控依據
