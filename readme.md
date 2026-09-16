# TDA1543 Online Player

把 Raspberry Pi Zero W 上的音樂檔，經 I²S 與 TDA1543 16-bit DAC 轉成類比音訊的網頁控制播放器原型。適合想把「瀏覽器點歌」一路接到實體電路的電子與程式實作者。

目前可查看電路實作、網頁與播放紀錄，並在具備相容音效硬體的 Raspberry Pi 上嘗試播放。這裡的 Online 指透過網頁控制本機曲庫，沒有串流音樂服務整合。

![Raspberry Pi 與 TDA1543 的實體製作](content/tda1543.jpg)

## 從點選曲目到聲音輸出

沒有螢幕的 Raspberry Pi 不方便逐次下指令選歌；這個專案用簡單網頁列出 `music/` 第一層檔案，點選曲目後由 Pi 執行解碼與音效輸出。

- **曲庫介面**：[Flask 主程式](app.py) 取得檔名，交給 [HTML 模板](templates/index.html) 顯示。
- **播放路徑**：`GET /play` 啟動 FFmpeg，輸出至 ALSA 裝置 `hw:0,0`，聲音由 Pi 的音效硬體播放，而不是瀏覽器。
- **硬體製作紀錄**：保留 [系統架構圖](content/Architecture.jpg)、[網頁畫面](content/tda1543_webui.jpg) 與 [FFmpeg 主控台紀錄](content/console.jpg)。這些是歷史實作截圖，不代表本次已重新接線量測。

```text
瀏覽器選取檔名 → Flask /play → FFmpeg 解碼 → ALSA hw:0,0 → I²S → TDA1543 DAC
```

播放採同步 `subprocess.run()`，程式短、容易追蹤，但 HTTP 請求會等到播放結束。沒有播放佇列、暫停／停止、音量控制或多人播放協調。模板上的資料夾樹與工具列大多是樣板，不能當成已實作的檔案管理器。

## 在 Raspberry Pi 嘗試

需要 Raspberry Pi Zero W／相容 Raspberry Pi、TDA1543 電路、類比音訊接收設備，以及 Python 3、FFmpeg、ALSA。硬體請先對照原有 [電路參考](https://aroundwaves.wordpress.com/2015/01/23/malinowy-dac-dla-i2s-na-tda1543-lub-1541-cz-3/)。Repo 沒有完整 BOM、接線驗收或音質量測報告。

### 1. 設定 I²S 與 ALSA

原始實作使用 Raspbian，編輯 `/boot/config.txt`（其他系統映像的設定位置與 overlay 相容性需另確認）：

```ini
dtparam=i2s=on
#dtparam=audio=on
dtoverlay=hifiberry-dac
```

建立或調整 `/etc/asound.conf`，保留原本以 card 0 輸出的設定：

```text
pcm.!default {
 type hw card 0
}
ctl.!default {
 type hw card 0
}
```

重新開機後，用 `aplay -l` 確認裝置。程式直接指定 `hw:0,0`；若 DAC 不是 card 0、device 0，只改 `asound.conf` 並不會改變程式的輸出目標，需要同步調整 `app.py`。

### 2. 準備程式與自己的音樂檔

在 Debian／Raspbian 類系統，從 repo 根目錄操作：

```bash
git clone https://github.com/KarlSideProjects/TDA1543_Online_Player.git
cd TDA1543_Online_Player
sudo apt install python3-venv ffmpeg alsa-utils
python3 -m venv .venv
. .venv/bin/activate
python3 -m pip install Flask Flask-RESTful
mkdir -p music
```

將有權使用的音樂檔放入 `music/`。Repo 已移除媒體檔，也沒有鎖定 Python 套件版本；首次重現需確認環境相容性。先分開驗證音訊路徑：

```bash
ffmpeg -i music/example.flac -f alsa hw:0,0
```

把 `example.flac` 換成實際檔名；預期由接在 DAC 後端的設備發聲。

### 3. 啟動與操作

以僅限本機的方式啟動開發伺服器：

```bash
python3 -m flask --app app run --host 127.0.0.1 --port 5000
```

在 Pi 本機瀏覽器開啟 `http://127.0.0.1:5000/`，可看到曲目清單。需要同一個受信任區域網路的其他裝置操作時，將 `--host` 改為 `0.0.0.0`，使用 Pi 的實際 IP；模板的 IP 欄位是固定示意值，不會自動反映主機位址。

網頁載入舊版外部 Shield UI 與 PrepBootstrap 腳本；若載入失敗，初始化可能中斷，連帶讓曲目點選失效。可另開終端直接驗證播放端點：

```bash
curl --get --data-urlencode 'name= example.flac ' http://127.0.0.1:5000/play
```

保留檔名前後各一個空白：目前後端以 `name[1:-1]` 去掉模板產生的兩側字元。回應 `singing` 不等於播放成功，因為程式沒有檢查 FFmpeg 的回傳碼，仍需看終端錯誤及實際音訊。

原有 `python3 app.py` 會開啟 debug 並監聽所有介面；此原型沒有登入、檔名驗證或完整錯誤處理，只適合隔離或受信任環境的實驗，不適合直接暴露到網際網路。

## 可以深入追問的設計

這個作品把軟體事件與實體訊號串在一起：網頁請求成功、FFmpeg 解碼成功、ALSA 接受資料，以及 DAC 正確發聲，是四個不同的檢查點。逐層排除問題，比只看「按鈕能不能按」更能說明工程除錯的過程。

可延伸為數位取樣、位元深度、數位／類比轉換與系統整合的探究活動，例如比較不同音訊格式是否能沿同一條輸出路徑播放。這是可進行的學習設計，尚無已完成的教學成效、失真或訊噪比實驗數據。

## 驗證與來源界線

本次文件依據主程式、模板與現存圖片整理；repo 沒有自動測試或 CI，未重新驗證 Raspberry Pi 接線、聲音輸出或舊版外部前端腳本。先完成單獨的 FFmpeg 播放，再驗證 HTTP 控制，能區分硬體與網頁問題。

原 README 標示 MIT，但 repo 沒有對應的專案 LICENSE 檔，因此不能只憑徽章推定整份作品的授權。第三方來源仍各自保留：

- 電路：上方連結的 AroundWaves 文章。
- UI 模板：`templates/index.html` 標示 PrepBootstrap，並引用 Shield UI。
- Bootstrap、jQuery：見所附檔頭的來源與授權聲明。
- Font Awesome 4.7：所附檔頭標示字型 SIL OFL 1.1、CSS MIT。
- 音訊工具：[FFmpeg](https://www.ffmpeg.org/) 與 [ALSA](https://www.alsa-project.org/wiki/Main_Page)。

這份 README 描述 repo 中可查驗的整合成果，不把第三方電路、模板或工具宣稱為本專案獨立原創。
