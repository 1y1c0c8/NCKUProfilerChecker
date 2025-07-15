# Gemini 專案日誌: NCKU 研究人員資料庫檢查工具

## 專案目標

此專案的目標是建立一個 Jupyter Notebook (`.ipynb`) 工具，用於自動化檢查一份研究人員名單，確認他們是否存在於成功大學(NCKU)的研究成果網站中。判斷的依據是，在搜尋結果頁面中是否存在一個標題為 "Profiles" 的特定 HTML 區塊。

最終，工具需要將檢查結果分類，並將「存在於系統中」和「不存在於系統中」的兩份名單分別匯出為獨立的 CSV 檔案。

## 核心功能

- **輸入**: 一個在 Notebook 中手動設定的 Python 列表 `input_list`，包含待查詢的人員姓名。
- **處理流程**:
  1. 遍歷 `input_list` 中的每一個姓名。
  2. 為每個姓名建立一個符合 NCKU 網站格式的查詢 URL。
  3. 發送一個帶有偽裝 `User-Agent` 的 HTTP GET 請求，以模擬真實瀏覽器，避免被伺服器以 403 Forbidden 錯誤阻擋。
  4. 使用 `BeautifulSoup` 解析回傳的 HTML 內容。
  5. 檢查頁面中是否存在 `<h2 class="section-title">` 且內容包含 "Profiles" 的元素。
  6. 根據檢查結果，將姓名分別加入 `found_list` 或 `not_found_list`。
- **輸出**:
  1. 一個合併的報告 `ncku_profile_check_results.csv`。
  2. 一個名為 `存在於系統中的人員.csv` 的獨立檔案。
  3. 一個名為 `不存在於系統中的人員.csv` 的獨立檔案。

## 開發歷程

1.  **初版建立 (2025-07-15)**: 根據使用者需求，產生了 `ncku_profile_checker.ipynb` 的第一個版本。
2.  **JSON 格式錯誤**: 使用者回報無法開啟 `.ipynb` 檔案，錯誤訊息為 `Expected double-quoted property name`。這表明我產生的 JSON 字串中存在語法錯誤 (很可能是引號轉義問題)。經過兩次嘗試，最終透過簡化 Python 字串的表示方式 (使用 `.format()` 而非 f-string) 修正了此問題，成功產生了可讀的檔案。
3.  **環境依賴問題 (`ImportError`)**: 使用者在執行 Notebook 時遇到 `ImportError: IProgress not found`。這是因為程式碼中使用了 `from tqdm.notebook import tqdm`，它依賴於 `ipywidgets` 套件，而使用者的環境中並未安裝。在建議使用者安裝套件但發現其環境 `PATH` 有問題後，最終決定修改程式碼本身。
4.  **移除 `ipywidgets` 依賴**: 為了讓工具更具通用性，將 `from tqdm.notebook import tqdm` 修改為 `from tqdm import tqdm`。這使得進度條變為純文字模式，不再需要 `ipywidgets` 即可執行。
5.  **檔案遺失與重建**: 在修改過程中，工具回報找不到檔案。經過 `list_directory` 確認後，發現檔案確實遺失。因此，我根據記憶中的最終版本，重新建立了包含所有修正的 `ncku_profile_checker.ipynb`。
6.  **HTTP 403 Forbidden 錯誤**: 使用者回報在存取網站時遇到 403 錯誤。我判斷這是由於網站的反爬蟲機制偵測到 `requests` 函式庫的預設 User-Agent 所致。為了解決這個問題，我在 `requests.get()` 中加入了一個偽裝成 Chrome 瀏覽器的 `User-Agent` header。
7.  **新增功能 (分別匯出 CSV)**: 使用者要求新增一個功能，將「存在」和「不存在」的兩個列表分別儲存為獨立的、以中文命名的 CSV 檔案。我為此在 Notebook 末尾新增了一個新的程式碼區塊，完成了此功能。

## 主要檔案

- `ncku_profile_checker.ipynb`: 核心的 Jupyter Notebook 工具。
- `ncku_profile_check_results.csv`: (舊的) 包含所有人員及其狀態的合併 CSV 報告。
- `存在於系統中的人員.csv`: 最終產出的報告之一，僅包含成功找到的人員。
- `不存在於系統中的人員.csv`: 最終產出的報告之二，僅包含未找到的人員。

## 如何使用

1.  使用 Jupyter Notebook 或相容的編輯器 (如 VS Code) 開啟 `ncku_profile_checker.ipynb`。
2.  找到標示為 `TODO` 的第二個程式碼區塊，修改 `input_list` 的內容，填入您想查詢的姓名。
3.  從頭到尾依序執行 Notebook 中的所有儲存格 (Cells)。
4.  執行完畢後，會在同一個目錄下找到產生的三個 CSV 報告檔案。
