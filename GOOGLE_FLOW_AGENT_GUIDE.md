# Google Flow 影片製作與 Codex 操作手冊

最後核對：2026-10-24。適用於 SOIL《四神破幻錄》專案，以及接手本專案的其他 Codex／agent。這份手冊整理實際跑過的流程；Flow 畫面、模型、點數和 MCP 上游程式可能改變，每次生成仍以當下狀態為準。

## 核心做法

把工作分成三個可檢查的階段：先確認故事和影片規格，再把劇本拆成分鏡與逐鏡提示詞，最後才在取得單次授權後操作 Flow。生成成功只代表有輸出；技術、畫面、教學意義、跨鏡連續性和音訊要分別檢查。原片、失敗版本和已採用版本都要分開保存。

本手冊是操作流程，不取代根目錄 AGENTS.md 的規則，也不取代使用者對劇情、角色、鏡頭及點數的決定。缺少資訊時標示「待確認」，不要默默補定。

## 新 agent 接手時先讀什麼

在進行新工作或操作 Flow 前，先確認目前所在專案是 SOIL影片資料夾，再依序閱讀：

1. AGENTS.md：專案共同規則。
2. CODEX_HANDOFF.md：最精確的續作入口、最近鏡頭和授權邊界。
3. PROGRESS.md、TODO.md：已完成與待辦狀態。
4. outputs/fullfilm_v0.1/README.md：逐鏡原片、QA、候選及採用版索引。
5. outputs/storyboard_v0.1/README.md、shots.json 和拍攝劇本：理解總敘事、分鏡、時間及跨鏡空間。
6. 目前鏡頭的 preparation、prompt 和 QA 文件；至少重看前一個已採用鏡頭及其相關參考圖。

【公開版說明】下列路徑、鏡號和 QA 例子取自原 SOIL 專案；本 repo 只提供手冊，不包含該專案的劇本、素材、Flow 登入 profile 或本機工作檔。其他專案使用時，請以當前工作區的文件和素材取代範例路徑，並從當前 Flow 專案核實設定。

文件裡的點數是歷史記錄，不是即時餘額。要先用 Flow MCP 查現況；也不要把上一輪已使用或失敗的單次授權帶到下一次生成。

## Flow MCP 是什麼，以及安裝方式

本專案使用第三方 GTAI-1/flow-mcp。它在本機啟動專用 Chrome，透過瀏覽器操作 Flow 網頁，不是 Google 官方 Flow API。上游說明和程式在 [GTAI-1/flow-mcp](https://github.com/GTAI-1/flow-mcp)。Windows 上游支援狀況可能變動；本專案曾在 Windows 成功安裝、建置、連線、下載和生成，但其他電腦仍要自行驗證。

本專案已驗證的來源版本為 commit 404c4d3bef59ac110a1303b2c88c21856b0783dd；Node.js 24.16.0、npm 11.13.0。為了讓跨電腦結果一致，先用這個固定版本，不要直接 git pull 更新。更新前要先檢視差異、工具權限和參考圖處理，再測建置及唯讀連線。

### Windows 安裝

準備 Node.js 22 或更新版本、Git 和 Google Chrome。影片 QA 另需 FFmpeg／ffprobe；只有要做本機語音轉錄或字幕時才需要 whisper.cpp。建議把 MCP 原始碼放在不會被 OneDrive 設成僅線上的本機路徑，例如 C:\Users\<使用者>\flow-mcp；若 SOIL專案本身位於雲端同步資料夾，避免把 Chrome 登入 profile 或 node_modules 同步上雲端。

在 PowerShell 執行以下命令。若工具目錄已存在，先核對版本，不要重複 clone 或覆寫：

    $flowMcpDir = Join-Path $env:USERPROFILE 'flow-mcp'
    git clone https://github.com/GTAI-1/flow-mcp.git $flowMcpDir
    git -C $flowMcpDir checkout 404c4d3bef59ac110a1303b2c88c21856b0783dd
    Set-Location $flowMcpDir
    node --version
    npm --version
    npm ci --ignore-scripts --no-audit --no-fund
    npm run build

本專案實際使用 npm ci 和鎖定檔安裝，npm run build 成功。該固定版本的 npm test 是空的佔位指令，不能寫成測試通過。

Flow MCP 會用專用 Chrome profile，必須在安裝程式、登入和 Codex 設定中都指定同一個 FLOW_MCP_HOME。建議放在本機使用者資料夾：

    $env:FLOW_MCP_HOME = Join-Path $env:LOCALAPPDATA 'flow-mcp'
    npm run login

新的專用 Chrome 視窗開啟後，由使用者自行登入 Google、開啟正確 Flow 專案，並讓電腦保持喚醒。不要把密碼、驗證碼或 chrome-profile 複製到聊天、文件、Git 或另一台電腦。登入完的 profile 是該電腦自己的設定。

### Codex 專案連線

本專案使用根目錄 .codex/config.toml 的專案級 MCP 設定。新電腦要依實際路徑修改 node.exe、dist/index.js、cwd、FLOW_MCP_HOME 和 FLOW_MCP_OUTPUT；TOML 不會自動展開 %LOCALAPPDATA%，要填入實際絕對路徑。先用下列命令查 Node 路徑：

    (Get-Command node).Source

設定範例如下；將尖括號位置換成該電腦的真實完整路徑：

    [mcp_servers.flow_mcp]
    command = "C:\\Program Files\\nodejs\\node.exe"
    args = ["C:\\Users\\<使用者>\\flow-mcp\\dist\\index.js"]
    cwd = "C:\\Users\\<使用者>\\flow-mcp"
    enabled = true
    startup_timeout_sec = 20
    tool_timeout_sec = 60
    enabled_tools = ["flow_status", "flow_assets", "flow_download", "flow_generate", "flow_wait"]
    default_tools_approval_mode = "prompt"

    [mcp_servers.flow_mcp.env]
    FLOW_MCP_HOME = "C:\\Users\\<使用者>\\AppData\\Local\\flow-mcp"
    FLOW_MCP_OUTPUT = "C:\\Users\\<使用者>\\Desktop\\SOIL影片\\outputs\\fullfilm_v0.1\\flow-mcp-downloads"

Codex 若提示專案設定尚未信任，先檢查設定內容，再只對這個專案允許其 MCP 設定。重新開啟 Codex 專案／工作階段後，在 SOIL 專案對話使用 /mcp 確認伺服器和工具已載入。專案級伺服器不一定會出現在全域 MCP 清單。不要為了方便把登入 profile 或本機路徑寫進共享設定。

## 企劃、劇本和分鏡怎麼討論

每支影片開始前，先確認主題、目的、觀眾、播放場合、片長、比例、美術風格和聲音需求。已知資訊先整理，缺少的列為「待確認」。不要在使用者尚未決定時替他確定風格、內容或成片長度。

創作分成以下三層，避免劇本和提示詞互相混淆：

1. 劇情提案：說清楚影片要讓觀眾理解什麼、人物要經歷什麼轉變。
2. 拍攝劇本：用場次和時間安排畫面、旁白、對白、音效和因果；核對總時長及聲音是否有足夠演出時間。
3. 分鏡與 Flow 提示詞：每鏡一個主要行動，列出人物與道具、起始狀態、動作先後、鏡位、場景方位、光線、聲音、參考素材和預定剪用時間。

在本專案，完整劇本約5分30秒，分鏡包有53鏡，其中52鏡先生成8秒來源，最後一鏡是後製字卡；實際剪用秒數另依每鏡動作決定。不要將「來源8秒」誤當「成片每鏡都剪8秒」，也不要把文字腳本總長當作已驗證的配音或成片長度。

分鏡要寫得可觀察、可判斷。例如「支點滑移」要寫清楚是誰操作、哪個接觸點移動、負載和繩索狀態、何時看出失敗；不能只寫「看起來更穩」。涉及路徑、節點數、左右岸、橋柱、河流方向或固定岩石時，文字和參考圖要一致。複雜教學圖形優先先準備固定圖，再要求模型只做一個可見變化。

討論時先出文字方案與樣例，確認後再批次整理分鏡；先做代表性樣片，經過檢視才展開更多鏡頭。修改某鏡時只改必要內容，保留舊提示詞、素材及版本，並清楚註明新版本解決哪些已觀察到的問題。

### 單鏡工作卡建議欄位

- 鏡號、片中時間、故事／教學目的。
- 起始畫面與結束畫面；人物位置及場景方向。
- 一個主要動作，以及動作的先後順序。
- 必須保留的物件、拓樸、數量、接觸點或視線。
- 明確禁止的偏差，例如仿字、額外角色、錯誤分枝、方向翻轉。
- 景別、鏡頭運動、光線、美術風格和聲音。
- Flow 參考圖、角色圖、前後鏡連戲條件。
- Flow 來源秒數、預計剪用秒數和 QA 標準。

## Flow 操作與付費生成閘門

### 先確認即時狀態

1. 呼叫 flow_status：確認已登入、正確 Flow 專案開啟、方案和即時餘額、目前佇列，以及伺服器回傳的提示詞說明。
2. 呼叫 flow_assets：確認素材標題、類型、影片是否已存在，並避免重複生成或選錯附件。
3. 需要時由使用者在專用 Chrome 開啟本片 Flow 專案。專案 ID 應從當前 Flow 畫面或本機受控設定中確認；不要把帳戶專屬 ID 寫入公開文件，並核實畫面和 MCP 回報一致。

過去點數、畫面標題或本機準備稿不能取代即時核對。flow_download 是下載既有資產，不會重新生成；下載前用 flow_assets 確認正確標題，避免以相同前綴誤中其他版本。

### 付費前核對

使用者必須明確授權本次付費生成，並提供或確認次數和點數上限。單次授權只涵蓋一個生成操作，不能延伸為重試或下一鏡授權。付費生成前逐項確認：

- Flow 專案、鏡號、take 編號、提示詞全文和參考素材。
- 當下模型、比例、解析度、秒數、變體數及 Flow 顯示報價。
- 生成上限 max_credits 不高於使用者授權；variants 設 1，retries 設 0。
- 本次 scenes 只有一鏡，佇列沒有可能混淆的工作。
- 若價格高於上限、附件或實際提示詞錯誤、Flow 未登入、專案不明或報價未確認，就不送出。

本專案常用 Omni 1.1 Flash、16:9、720p、8秒、單支；曾有單支最多12點的明確授權例子。這不是固定價目，模型與點數每次以 Flow 即時報價為準。

flow_generate 會排入工作並回傳 job id；再用 flow_wait 等待，直到 finished=true。若仍在執行就按 job id 繼續等待。不要重複呼叫 generate 來「看看有沒有完成」。完成後重新查 flow_status 和 flow_assets，記下作業 id、場景標題、實扣點、前後餘額、實際提示詞、附件和下載位置。

flow_generate 的參考圖可使用本機絕對路徑或 asset:素材標題。Flow Composer 的影片設定不能同時使用 references 與 first/last frame；若填了首尾幀，工具會略過參考圖。提交前要核對實際畫面和工具回傳的提示詞，不要只看本機草稿。

## 生成後的保存、檢查和採用

先下載 original 原片，保存到 outputs/fullfilm_v0.1/raw/，不要覆寫舊 take。MCP 下載副本保留在 flow-mcp-downloads/；提示詞草稿和實際送出文字分開保存，QA 圖放 qa/，經剪輯的候選或採用版放 edits/。檔名包含鏡號、內容、take 編號和版本。

先做技術檢查，再做畫面判斷。Windows 上可用：

    ffprobe -v error -show_streams -show_format "outputs/fullfilm_v0.1/raw/S028_保留與改寫_take05.mp4"
    ffmpeg -v error -i "outputs/fullfilm_v0.1/raw/S028_保留與改寫_take05.mp4" -f null -
    Get-FileHash -Algorithm SHA256 "outputs/fullfilm_v0.1/raw/S028_保留與改寫_take05.mp4"
    ffmpeg -y -i "outputs/fullfilm_v0.1/raw/S028_保留與改寫_take05.mp4" -vf "fps=2,scale=480:-1,tile=4x4" -frames:v 1 "outputs/fullfilm_v0.1/qa/S028_take05_2fps_contactsheet.jpg"

對8秒影片，2fps接觸表可快速看16個時間點；另抽查轉折、關鍵接觸點、片尾和相鄰採用鏡頭。實際觀看／聽取媒體後再下結論。技術解碼成功不代表畫面符合；抽格可見也不必然代表動態因果成立。

每個 take 的狀態要明確區分：

| 狀態 | 代表意思 |
| --- | --- |
| 已生成 | Flow 有產生資產；仍待檢查 |
| 技術 QA 通過 | 解析度、codec、fps、秒數、聲軌及完整解碼已核對 |
| 畫面 QA 通過／不通過 | 實際影格符合或未符合該鏡的核心要求 |
| 候選 | 視覺初審可用，已另存規定秒數的編輯版，仍待使用者決定 |
| 正式採用 | 使用者明確採用；如需剪裁，另存版本並再核規格／雜湊 |
| 音訊已聽審 | AAC、旁白、對白或環境音已實際聽過；不能由 ffprobe 代替 |

只有使用者選擇候選後才標為採用。採用參考圖不等於採用影片；生成完成也不等於品質驗收。若核心構圖失敗，就保留原片和 QA，不要製作成假裝合格的候選。

## 已遇過的問題和處理方式

- Flow MCP 回報 fetch failed，或 Windows 網址安全檢查說無法判定目前瀏覽器網址：先停止付費操作，不要重複開頁、重送或猜網址。若 MCP 無法安全確認，將實際提示詞、模型、附件、點數上限交給使用者手動送出；取得 Flow 資產標題後，再做本機下載及 QA。
- Flow 顯示 reCAPTCHA／異常活動：不繞過驗證，也不盲目重試。記錄錯誤與前後點數；即使餘額未變，重試仍要新的明確授權。
- 固定 commit 404c4d3 的素材篩選曾把符合 ASCII 名稱的拓樸圖誤加成 CHARACTER LOCK；只調換附件順序無效。S028 take05 改用本機全頁 PNG 絕對路徑作第一張參考，才避免該錯誤。若實際提示詞出現不預期的 CHARACTER LOCK，停止提交並先檢查 MCP 程式與傳入素材；此為該版本觀察，不應推定所有版本都有相同行為。
- 參考圖不能保證影片照圖生成。S028 take05 的全頁固定參考已成功上傳、技術規格和解碼也通過，但左路仍變成十字、橢圓圈錯節點、繞石線變成連到陣列的大迴圈，因此未採用。對精確教學拓樸，要把數量和連接關係列成 QA 硬條件。
- Flow 曾因提示詞被判定涉及知名人士而拒絕生成。改用原創虛構人物的外觀描述，不沿用真實姓名；政策拒絕後不要在同一授權下改字重送，需重新確認新的付費授權。
- Windows Git 若出現 dubious ownership，不要直接執行 git config --global safe.directory。只在確有必要時用單次 git -c "safe.directory=C:\專案路徑" -C "C:\專案路徑" status --short 命令，並將範例路徑換成實際絕對路徑，或記錄該檢查不可用。
- 生成影片沒下載到本機，不代表應付費再生。先查 Flow 資產並用 flow_download 下載；此功能不消耗生成點數。

## 每次收工要更新什麼

將最新狀態寫回 PROGRESS.md、TODO.md 和 CODEX_HANDOFF.md；outputs/fullfilm_v0.1/README.md 維護逐鏡索引，該鏡 preparation 和 QA 文件記錄實際結果。交接須列明：

- 哪些素材實際生成、哪個檔案為原片、作業 id 和實扣點。
- 技術／畫面／音訊各自檢查到哪裡；未做的明確寫出。
- 候選、採用或不採用狀態及理由。
- 下一個具體工作點、生成前仍需的使用者授權，以及停止原因。

不要只寫「已完成」。如果生成失敗、影片不採用或授權未送出，照實記錄，並保留原始證據。

## 可貼給新 agent 的承接指令

    請在 SOIL影片專案接續 Google Flow 逐鏡製作。先讀根目錄 AGENTS.md、GOOGLE_FLOW_AGENT_GUIDE.md、CODEX_HANDOFF.md、PROGRESS.md、TODO.md，以及目前鏡頭的準備稿和 QA。先恢復最新狀態；未確認的內容標示待確認。

    目前要做的鏡頭／任務：<填鏡號和明確範圍>
    本次是否只做本機準備／QA：<填寫>
    付費生成授權：每次生成都需確認本次明確單次授權和點數上限；不要沿用舊授權。

    若有付費生成，先即時檢查 Flow 專案、登入、餘額、佇列、模型、報價、提示詞與附件；variants=1、retries=0，max_credits 不超過授權。生成後等到工作完成，下載原片、做技術與影格 QA，分開記錄候選／採用／音訊狀態，最後更新交接文件。

## 參考資料

- [Flow MCP 上游程式與說明](https://github.com/GTAI-1/flow-mcp)
- 本機安裝與唯讀試跑紀錄：outputs/fullfilm_v0.1/flow-mcp-downloads/mcp_trial_S025_take01_20261023/MCP試跑紀錄_20261023.md
- 本專案逐鏡工作索引：outputs/fullfilm_v0.1/README.md
- 分鏡製作包：outputs/storyboard_v0.1/README.md
- 成片規格與 Google Flow 製作流程：outputs/SOIL四神破幻錄_GoogleFlow製作流程_v0.1.md
- Google Flow 官方說明：[影片與參考輸入](https://support.google.com/flow/answer/16353334?hl=en)、[模型與支援功能](https://support.google.com/flow/answer/16352836)、[場景剪輯](https://support.google.com/flow/answer/16935718)
