# 開發紀錄: NCKU 研究人員資料庫檢查工具

本文檔記錄了「NCKU 研究人員資料庫存在性檢查工具」的開發歷程、遇到的挑戰，以及採用的核心技術。旨在為未來的維護者或對此專案感興趣的開發者提供深入的技術參考。

## 專案演進摘要

此工具從一個簡單的手動輸入腳本，逐步演化為一個自動化、更具彈性的解決方案。

1.  **初期版本**: 最初的版本要求使用者在 Jupyter Notebook 中手動維護一個 Python 列表 `input_list`。腳本會遍歷此列表，並對每個姓名進行查詢。

2.  **遭遇 403 Forbidden**: 在初次嘗試爬取網站時，遭遇了 `HTTP 403 Forbidden` 錯誤。這表明目標網站有基礎的反爬蟲機制。為了解決此問題，引入了 `User-Agent` 偽裝技術。

3.  **自動化輸入流程**: 為了提升效率並減少手動操作的錯誤，專案需求變更為從 PDF 檔案自動讀取姓名。為此，我們引入了 `pdfplumber` 套件。

4.  **優化輸出結構**: 根據使用者需求，將原本合併的 CSV 報告，改為分別輸出「存在」與「不存在」兩個獨立的檔案，並調整了檔案的儲存路徑至 `results` 資料夾。

5.  **新增統計功能**: 在 Notebook 的結尾處增加了最終的統計摘要，讓使用者可以一目了然地看到本次任務的執行成果。

---

## 核心技術深度解析

### 1. HTTP User-Agent 偽裝技術

**問題背景**:
在網路爬蟲的開發中，`HTTP 403 Forbidden` 是一個常見的錯誤。它意味著伺服器理解了我們的請求，但拒絕授權。在本次專案中，這是因為 `requests` 函式庫在發送請求時，會使用一個預設的 `User-Agent`，例如 `python-requests/2.28.1`。網站管理員可以輕易地設定防火牆規則，阻擋所有來自這個 `User-Agent` 的請求，從而達到反爬蟲的目的。

**解決方案**:
解決方法是模擬一個真實的、由主流瀏覽器發出的請求。我們透過在 HTTP 請求的標頭 (Headers) 中，手動指定 `User-Agent` 字串來實現這一點。

**程式碼實作**:
```python
# 偽裝成一個在 Windows 上運行的 Chrome 瀏覽器
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
}

# 在發送請求時，將此標頭附加進去
response = requests.get(search_url, headers=headers, timeout=15)
```

透過這個簡單的修改，我們的爬蟲請求在伺服器看來，就像是來自一個普通的 Chrome 瀏覽器用戶，因此能夠成功繞過基礎的存取限制。

### 2. 使用 `pdfplumber` 進行 PDF 表格解析

[![pdfplumber repo card](https://gh-card.dev/repos/jsvine/pdfplumber.svg)](https://github.com/jsvine/pdfplumber)

**問題背景**:
專案的後期需求是從 PDF 檔案中自動提取姓名列表。PDF 檔案本質上是為「列印」而非「資料交換」設計的，其內部結構複雜，直接讀取文字往往會得到混亂且難以解析的結果，特別是當內容包含表格時。

**解決方案**:
`pdfplumber` 是一個專為此類任務設計的 Python 套件。它的核心優勢在於能夠精準地識別 PDF 頁面中的線條和邊界，從而完美地重建表格的結構，並將其提取為可供程式處理的格式。

**程式碼實作**:
在 Notebook 中，我們設計了一個 `get_names_from_pdf` 函式來封裝這個過程。

```python
import pdfplumber
import pandas as pd

def get_names_from_pdf(pdf_path):
    names = []
    # 1. 使用 pdfplumber.open() 開啟 PDF 檔案
    with pdfplumber.open(pdf_path) as pdf:
        # 2. 遍歷每一頁
        for page in pdf.pages:
            # 3. 核心功能：使用 .extract_tables() 提取頁面中的所有表格
            tables = page.extract_tables()
            for table in tables:
                # 4. 將提取的表格資料轉換為 pandas DataFrame，方便後續操作
                #    我們將表格的第一行作為欄位名稱 (columns)
                df = pd.DataFrame(table[1:], columns=table[0])
                
                # 5. 檢查 '姓名' 欄位是否存在
                if '姓名' in df.columns:
                    # 6. 進行資料清理：移除 "教授" 字串，並過濾掉任何空值
                    cleaned_names = df['姓名'].str.replace('教授', '', regex=False).dropna()
                    names.extend(cleaned_names.tolist())
    return names
```

透過 `pdfplumber`，我們成功地將一個原本需要大量手動複製貼上的繁瑣任務，轉化為一個全自動、穩定可靠的程式化流程，極大地提升了此工具的實用性與效率。
