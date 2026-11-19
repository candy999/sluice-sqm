# 移植守則：從黑潮 DNS 移植到 Padavan 學到的事

Sluice 的目標平台是 OpenWrt，之後也要能放進其他路由器韌體（Padavan、Asuswrt-Merlin、一般 Linux）。
這份文件抄錄 [黑潮 DNS](https://github.com/candy999/kuroshio-dns) 移植到 [padavan-tw](https://github.com/candy999/padavan-tw) 時踩過的坑與定下的規則，並寫明 Sluice 對應的做法。

來源（抄錄時的版本）：

| 文件 | 位置 |
|---|---|
| 黑潮 DNS 移植指南 | kuroshio-dns `PORTING.md` |
| 黑潮 DNS 整合介面（穩定約定） | kuroshio-dns `docs/integration.md` |
| padavan-tw 後台介面交接單（第 4、5、6、12、13 節） | padavan-tw 分支 `claude/zealous-feynman-bhwrbf` 的 `docs/HANDOFF-UI.md` |
| padavan-tw 完整交接單（第 2 節：踩過的坑） | 同分支 `docs/HANDOFF-LOCAL.md` |
| padavan-tw 路線圖（E 章：硬體 QoS 捨棄） | 同分支 `docs/ROADMAP.md` |

每一條的格式：**學到的事**（出處）→ **Sluice 的做法**。

> **範圍（2026-11-19 定案）**：Sluice 不支援 MIPS，所以 Padavan 這類 MIPS 韌體不是移植目標。專案只維護 OpenWrt；
> 這份文件的目的是讓第三方作者容易把 Sluice 移植到其他平台（OpenWrt 衍生版、ARM／x86 的路由器韌體、一般 Linux）。
> 下面提到 Padavan 的條目是當時的實例，原則一樣適用。

---

## 1. 核心與介面分開：用穩定的 API 溝通

### 1.1 核心提供「穩定約定」，平台整合只依賴約定
- 黑潮把對外約定寫成一份文件：指令、參數、結束碼、JSON 欄位、訊號、環境變數。**JSON 只加欄位，不改名、不刪、不改型別**；給人看的文字不是約定。行為不相容時把整數 `api` 加 1，第三方用它判斷，不比對版本字串。（integration.md 開頭）
- 第三方韌體沒有實機，就不寫各家專屬腳本，而是提供穩定介面＋通用 Linux 包，由平台維護者整合。（PORTING.md）

→ **Sluice**：第一版就寫 `docs/integration.md`，約定 `sluice` 的子指令、結束碼、`status --json` 欄位、訊號、環境變數。`/status` 帶整數 `api`。所有平台（OpenWrt 的 rpcd、Padavan 的 CGI、Linux 的 systemd）都只透過這份約定溝通。

### 1.2 一個核心 API，各平台只寫「轉接」
- OpenWrt 用 `rpcd.sh`、Padavan 用 `/kuroshio_api.cgi` 轉接同一個管理 API；兩邊的**路徑白名單相同**，查看類與動作類分開，回應包裝一致（`{"ok":true,"result":…}` / `{"ok":false,"error":"…"}`）。（HANDOFF-UI 13.3）

→ **Sluice**：daemon 開 Unix socket（`/var/run/sluice/api.sock`，不佔 TCP 埠、不需要登入 token，靠檔案權限保護）。對外只有一個橋接指令 `sluice api <GET|POST> <路徑> [JSON]`，所有平台的轉接都只是呼叫它：
- OpenWrt：`sluice rpcd`（rpcd exec plugin）把 ubus 方法對應到 API 路徑
- Padavan：一支 CGI 殼呼叫 `sluice api`
- 一般 Linux：直接用指令，或自己包一層 HTTP

白名單寫在核心裡（`sluice api --list` 印出來），轉接層照抄，不自己維護。

### 1.3 轉接層不能吞掉錯誤
- `curl -f` 把管理 API 的 `HTTP 400 {"error":"DDNS 未啟用"}` 吃掉，畫面只看到「沒有回應」。（HANDOFF-UI 13.9 第 1 點）

→ **Sluice**：`sluice api` 的結束碼與輸出約定：成功輸出 `{"ok":true,"result":…}`；API 回錯誤時輸出 `{"ok":false,"error":"<核心的原因>"}`；只有連不上 daemon 才說「沒有回應」。轉接層一律原樣轉送。

### 1.4 耗時的動作一律背景執行
- 「立即更新過濾清單」在 D3 要 68 秒，轉接的 60 秒逾時先到，畫面說失敗但其實成功。黑潮 3.3.0 改成立刻回 `runId`，進度在 `/api/status` 查；重複呼叫回同一個 `runId`，不重複執行。（HANDOFF-UI 13.9 第 2 點、integration.md）

→ **Sluice**：**頻寬校準約 3 分鐘**，一定照這個模式：`POST /calibrate` 立刻回 `{started, runId}`，進度與結果在 `/status` 的 `calibration`（`state`、`phase`、`progress_pct`、`runId`、`result`、`error`）；跑的中途再按回同一個 `runId`。自測、CPU 跑分也一樣。

### 1.5 介面共用的純函式放進核心的 data/
- 黑潮把跟介面框架無關的純函式（時間格式、位元組格式、比對邏輯）放在 `data/shared/ui-core.js`，LuCI、Windows 設定頁、Padavan 都載入同一份；每個函式的輸入與預期輸出寫在 `ui-core.tests.json`，CI 逐一檢查。語法用 ES5，`require()` 也能用。（integration.md「網頁共用檔」）
- 儀表板 `dashboard.js` 從「轉接器」拿資料，各平台只寫轉接器。Padavan 直接把韌體裡黑潮附的 dashboard.js 連結進網頁目錄，不另外複製。（HANDOFF-UI 13.8）

→ **Sluice**：`data/shared/sluice-ui-core.js`：分數與白話說明（「延遲只多 3 ms」）、檔位說明（L0～L6）、單位格式、SVG 圖表繪製，加上 `sluice-ui-core.tests.json`。LuCI view 與 Padavan 頁面都只負責排版與取資料。這樣「延遲品質分數怎麼說」不會在兩個平台各寫一份而說法不同。

### 1.6 資料介面的共同格式（沿用 padavan-tw 的約定）
padavan-tw 的後台已經定好這套格式（HANDOFF-UI 第 6 節），Sluice 直接照用，之後放進 padavan-tw 的總覽頁不用轉換：

- 每個回應都有 `ok`、`ts`（Unix 秒）、`available`（這個功能目前韌體有沒有提供）、`warming_up`（資料收集中）
- 時間序列：`{"step": 秒, "start": Unix 秒, "values": [...]}`，缺資料用 `null`
- **單位寫在欄位名稱**：`_ms`、`_kbps`、`_pct`、`_bytes`、`_s`、`_ts`
- 狀態值固定五種：`ok` / `warn` / `bad` / `unknown` / `off`
- 事件：`events[{ts, type, level, text}]`，Sluice 用 `type: "qos"`（padavan-tw 的時間軸已預留這個類型）

---

## 2. 設定：不綁 UCI 指令

### 2.1 UCI 文字模式
- 沒有 `uci` 指令的韌體，把自己的設定（nvram）寫成 UCI 文字檔再啟動黑潮。Padavan 就是這樣做的。（PORTING.md 第 3 節）
- 另外支援讀 `ubus call uci get` 的 JSON 輸出（`KUROSHIO_UCI_JSON`）。（integration.md）

→ **Sluice**：核心自己解析 UCI 文字格式（不呼叫 `uci`、不連結 libuci）。`SLUICE_UCI_FILE` 指定檔案位置，預設 `/etc/config/sluice`。

### 2.2 nvram 蓋過 UCI 的對照表
- Padavan 用一張表列出「哪些 nvram 蓋過哪些 UCI 選項」，改 UCI 檔裡的同名選項沒用（會被蓋掉）；其他設定都寫在 UCI 檔。（HANDOFF-UI 13.4）

→ **Sluice**：設定欄位分兩群並在文件標明：**平台會接管的**（介面名稱、開關）與**只在 Sluice 裡的**（演算法參數、規則）。前者在 Padavan 由 nvram 產生。

### 2.3 寫設定的介面要保留看不懂的內容
- 介面讀寫 UCI 檔時，**沒改的行原樣輸出，保留註解與不認得的區段／選項**，不能因為介面沒有那一項就刪掉。（HANDOFF-UI 13.4、13.8 驗收）

→ **Sluice**：LuCI 走 UCI 本來就會保留；Padavan 版的頁面若自己解析 UCI 文字，要照這條做，並把「不認得的區段存檔後仍在」列入驗收。

### 2.4 升級遷移
- 預設設定有變時，**沒改過的整份換新，改過的只補新項目、不動使用者改的**；看不懂的版本標記就不動，只記一行說明。（HANDOFF-UI 13.12「升級遷移」）
- 設定欄位只新增；改名或淘汰時舊名稱照樣讀得進來，啟動日誌說明。（PORTING.md）

→ **Sluice**：`sluice uci-migrate <檔案>`，規則同上。設定檔裡放 `option config_version`。

### 2.5 重新載入：失敗時繼續用上一份
- 收到 SIGHUP 在同一個行程裡重新載入；**新設定讀不進來時不結束，繼續用上一份**，原因放在 `/status` 的 `warnings` 與 `reload.usingPrevious`。（integration.md「訊號」）
- 黑潮 3.3.2 SIGHUP 後約 4 秒不回答，整合腳本只好等 8 秒；3.3.3 縮到 0.3 秒內，等待時間跟著縮短。（HANDOFF-UI 13.12、13.13）

→ **Sluice**：reload 時**不拆掉 qdisc 重建**，只對變動的參數下 `tc qdisc change`（或 netlink），塑型不中斷；學到的頻寬、基線延遲留著。新設定不合法就保留舊的，並回報 `reload.usingPrevious`。

---

## 3. 儲存：路由器的 Flash 很脆弱

### 3.1 寫入頻率依檔案系統決定
- jffs2／ubifs／overlay 視為 flash，6～24 小時寫一次；tmpfs 視為 ram；內容沒變不寫；SIGTERM 時全部寫出；SIGUSR1 立即寫出。（PORTING.md 第 4 節）

→ **Sluice**：要保存的只有校準結果、學到的頻寬範圍、CPU 跑分，量很小。**只在校準完成或學習結果明顯改變時寫**，平常一天最多一次；執行狀態與圖表資料一律放記憶體。

### 3.2 設定放在記憶體的韌體要有「寫入後執行」的掛鉤
- Padavan 的 `/etc/storage` 是記憶體，要執行 `mtd_storage.sh save` 才寫進 Flash。沒做的結果：**斷電後在後台改的設定全部不見**（D3 實際發生過）。（HANDOFF-UI 13.8、13.9 第 8 點）
- 黑潮加了 `KUROSHIO_ON_CONFIG_WRITE`：寫入設定目錄後合併 3 秒執行一次，結束服務前也會先跑完；**只能用環境變數設定**，管理介面改不了（避免被拿來執行任意指令）。第一版監看的資料夾漏掉 `KUROSHIO_ETC`，Padavan 上不會觸發。（integration.md、HANDOFF-UI 13.10）

→ **Sluice**：`SLUICE_ON_STATE_WRITE`，規則照抄；**每一個會寫檔的位置都要列進觸發範圍**，並寫測試。

### 3.3 Padavan 的 Storage 分割區只有 512 KB
- 一個 303 KB 的舊備份佔掉大半，再大就**所有設定都存不進去**（存檔失敗只記 log）。（HANDOFF-UI 13.9 第 4 點）

→ **Sluice**：放在設定目錄的檔案要小（目標總共 < 16 KB），不在設定目錄放備份或歷史資料。

### 3.4 不碰任何儲存裝置
- 黑潮不會格式化或抹除任何儲存裝置，只用設定給它的目錄；準備可寫空間是平台的責任。`kuroshio paths` 列出所有實際位置。（PORTING.md、integration.md）

→ **Sluice**：同樣原則，加 `sluice paths`。

---

## 4. 服務管理與健康檢查

### 4.1 健康檢查要便宜、要看「程式本身活著」
- `kuroshio health` 問自己回答的名稱，不經過上游，WAN 斷線不會誤判；結束碼 0／1／2。（integration.md）
- 健康檢查讀**只有埠號的小設定檔**：使用者存了一份壞設定時，用正式設定會讀不到、保護程式以為服務掛了。（HANDOFF-UI 13.12「檢查用小設定檔」）

→ **Sluice**：`sluice health`：連 Unix socket 問 daemon，結束碼 0＝正常、1＝沒回應、2＝設定讀不到。socket 路徑寫死或由環境變數給，不依賴使用者設定檔。

### 4.2 故障時的退回
- 黑潮停擺時把 DNS 交回業者，不讓網路斷掉。（PORTING.md 第 6 節）

→ **Sluice**：daemon 掛掉時 qdisc 還留在介面上，頻寬卡在最後一個值。**看門狗**（procd 的 respawn 以外）：daemon 停止超過 N 秒，就把 qdisc 拿掉、讓網路回到不塑型（等同 L5）。這比「卡在一個可能過低的頻寬」安全。關機或停止服務時也要清乾淨。

### 4.3 精準辨識主程式
- `pidof kuroshio` 會把 `kuroshio health`、`export` 這些子指令也算進去，`stop` 曾經連每日匯出一起殺掉。（HANDOFF-UI 13.12「程式辨識」）

→ **Sluice**：daemon 寫 pidfile；停止服務只對 pidfile 的 PID 送 SIGTERM。

### 4.4 WAN 變動要通知
- WAN 重新撥號、換 IP 時執行 `kuroshio notify wan-changed`；Padavan 放在 `post_wan_script.sh`，OpenWrt 放 hotplug。（integration.md）

→ **Sluice**：`sluice notify wan-up` / `wan-down`。PPPoE 重撥會重建 `pppoe-wan`，qdisc 跟著消失，一定要接這個事件；daemon 自己也每輪檢查 ifindex 當保險。

### 4.5 系統時間還沒對時
- 沒有 RTC 的路由器開機是 1970 年，TLS 驗證全部失敗。黑潮以「早於建置時間 − 2 天」判斷沒對時，期間延後需要 TLS 的工作，**不記成失敗**。（integration.md）

→ **Sluice**：校準要連 HTTPS 測速伺服器，**對時前不校準**，先用上一次的結果或保守值；狀態顯示「等待系統對時」。

---

## 5. 網頁介面的規則（padavan-tw 後台定下的）

### 5.1 硬性限制（HANDOFF-UI 第 4 節）
1. **不載入任何外部資源**（CDN、Google Fonts 都不行）：路由器常常還沒連上網就要用後台
2. **不用前端建置工具**：純 HTML／CSS／JS
3. **路由器資源有限**：請求要少而小；**輪詢間隔至少 2 秒，頁面不在前景時停止輪詢**（`document.visibilityState`）
4. **手機**：可點區域至少 44 px；不能橫向捲動；**表格在手機改成卡片**
5. 亮色／暗色都要正確
6. 不疊特效：沒有漸層、厚陰影、裝飾動畫

### 5.2 設計原則（HANDOFF-UI 5.3）
1. **先回答問題，再給數字**：每個區塊先用一句話說結論，數字放下面當證據
2. **狀態 → 原因 → 行動**：有問題時附上為什麼、可以怎麼做，以及每個做法的**代價**
3. **漸進揭露**：預設只顯示多數人需要的設定
4. **套用前說清楚影響**：會不會斷線、要多久
5. **用台灣日常用語**，技術名詞第一次出現附白話說明
6. **每個數字都有參考**：「延遲 18 ms（平常 12 ms）」
7. **不嚇人**：警告分等級，只有真的影響使用才用紅色
8. **功能還沒好時誠實顯示**：「需要更新韌體後才能使用」「資料收集中（約需 10 分鐘）」，**不要顯示假資料**（模擬資料只用在預覽工具）

### 5.3 統一用語（HANDOFF-UI 5.7）
| 用 | 不用 |
|---|---|
| 網際網路 | 外網、WAN（進階頁可以括號附註） |
| 區域網路 | 內網、LAN |
| 裝置 | 用戶端、客戶端、終端 |
| 業者 | ISP、運營商 |
| 套用 | 應用、提交 |
| 路由器 | 主機、設備 |

### 5.4 Padavan httpd 的特殊限制
- 它會解析 `.js` 檔裡的模板標記，所以共用的 `.js`、`.css` **不能出現 `<%`、`%>`、`<#`、`#>`**（註解、字串、正規表示式都算）。黑潮用 CI 腳本檢查。（integration.md）
- httpd 的表單緩衝只有 **64 KB**，超過的部分直接丟掉、照樣存檔。前端要算「編碼後」的大小（換行是 CRLF、中文一個字 9 bytes）。（HANDOFF-UI 13.12）
- httpd 是單一程序：同時最多 2 個請求。（HANDOFF-UI 13.8）
- 載入共用檔時網址帶 `?v=版本`，升級後瀏覽器不會拿到舊快取。（integration.md）

→ **Sluice**：`tools/check-ui-assets.js` 進 CI，檢查上面四種標記。Sluice 的設定量小，但規則清單可能變大，Padavan 版的頁面要做大小檢查。

### 5.5 安全
- 會改變狀態的動作**只收 POST**，而且要從區網登入。（HANDOFF-UI 13.3）
- 跨站請求：只看 `Sec-Fetch-Site` 在 HTTP 後台擋不住（瀏覽器只在 HTTPS 才送）；改成要求自訂標頭 `X-TW-Request`（跨來源 fetch 帶自訂標頭要先預檢，httpd 不支援，所以擋得住）。（HANDOFF-UI 13.12）

→ **Sluice**：rpcd 有 LuCI 的 ACL；Padavan 版轉接照 padavan-tw 的做法（POST、login_safe、`X-TW-Request`）。核心的 API 路徑分「查看」與「動作」兩類，轉接層依此決定要不要 POST。

### 5.6 預覽工具與模擬資料
- 預覽工具在電腦上模擬 API，網址參數 `?scenario=` 切換 `normal`／`problem`／`empty`／`none`（沒有這個 API 的舊韌體）。模擬資料從實機抓真實回應，**遮掉 Token、網域、IP、MAC**。（HANDOFF-UI 2.1、6、13.5）

→ **Sluice**：`tools/ui-preview/`：一支小伺服器，回 `mock/<路徑>.<情境>.json`；情境至少要有 `normal`、`degraded`（CPU 降檔）、`calibrating`、`learning`（不校準）、`off`。

---

## 6. 硬體與平台的事實（直接影響 Sluice 的設計）

| 事實 | 出處 | 對 Sluice 的影響 |
|---|---|---|
| **MT7621 跑 CAKE，PPPoE 大約 400 Mbps 就到頂** | ROADMAP E 章 | 500/500 線路在 MT7621 一定會降檔（L4 或 L6）。CPU 保護是必要功能，不是附加功能 |
| **硬體 QoS（QDMA）讓 CPU 平均從 17% 升到 57%，限速 250／200、150／100 時 Wi-Fi 延遲都沒改善**；Wi-Fi 滿載延遲是驅動佇列與空中時間不公平造成的，不在 WAN | ROADMAP E 章、WIFI-STABILITY.md | Sluice 只解決**網際網路這一段**的延遲。介面不能暗示能改善 Wi-Fi 延遲；診斷要分開說明是 Wi-Fi 慢還是網際網路慢 |
| **硬體 NAT 會繞過流量計數**；padavan-tw 的 WAN 速率取「WAN 介面」與「交換器埠 `ESW_P0`」兩者中大的 | HANDOFF-UI 第 4 節、第 6 節 `wan` | L6（交還硬體加速）時，介面計數不準，負載判斷要換資料來源或標示「硬體加速中，數字僅供參考」。**不做每台裝置的流量排行** |
| Padavan 沒有 nftables，用 iptables＋ipset | ROADMAP G3 | 分類要有兩個後端：nftables（OpenWrt）、iptables（Padavan） |
| Padavan 是 Linux 4.4 | HANDOFF-LOCAL 第 2 節 | `act_ctinfo`（5.3 起才有）不能用，下載方向的分類要另外想辦法（需確認：4.4 上能用的 `act_connmark`＋CAKE 的 `fwmark`、或只分類上傳）。4.4 也沒有 PSI |
| 上游 TurBoTse 有 SQM／cake（含 IFB 模組）；padavan-tw 因 CPU 負擔已把軟體 SQM 移除，並列「Rust 版 SQM＋cFosSpeed 類套件整合：未來另外評估」 | PATCH-REVIEW、ROADMAP「不做」 | Sluice 就是這個「未來評估」的候選。進 Padavan 時，CPU 保護＋只塑上傳（L4）是主要賣點 |
| D3 根目錄 squashfs 已滿（13 MB 用到 100%），黑潮執行檔 9.5 MB | HANDOFF-UI 12.1 | Sluice 的 1 MB 目標對 Padavan 很重要 |
| busybox 沒有 `cmp`（每次比對都當成不同，導致每次都重啟 dnsmasq）；busybox `find` 不跟進捷徑（`/proc/device-tree/` 一定要加斜線） | HANDOFF-UI 13.12、HANDOFF-LOCAL 第 2 節 | 整合腳本只用 busybox 確定有的功能；能在 Rust 裡做的就不要交給 shell |
| 中華電信 MOD（IPTV）流量不應經過排程 | ROADMAP E11 | Sluice 要辨識並排除 MOD 的 VLAN／介面（待查 OpenWrt 上的常見接法） |

---

## 7. 打包與發布

- **通用 Linux 包**：每個 CPU 一個 tar.gz，執行檔＋同版 `data/`＋預設設定；`data/manifest.json` 記錄格式版本，不一致時警告但照常執行。（PORTING.md 第 2 節、integration.md）
- **musl 靜態連結**，不依賴系統函式庫；Rust 標準函式庫要求 Linux 3.2 以上；已知在 4.4（MT7621）可以執行。MIPS 用 nightly＋`-Zbuild-std`，軟體浮點。（PORTING.md 第 1 節）
- **GPL 原始碼**：每個 Release 附 `<名稱>-<版本>-source.tar.xz`（原始碼＋`cargo vendor` 的全部依賴，可離線編譯），雜湊列在 `SHA256SUMS`。GitHub 自動產生的原始碼壓縮檔不保證位元組相同，不適合核對。padavan-tw 的 CI 就是抓這個檔附進自己的 Release。（PORTING.md「授權」、HANDOFF-UI 13.11）

→ **Sluice**：照抄。第一版就產 `sluice-linux-<target>.tar.gz`、`sluice-<版本>-source.tar.xz`、`SHA256SUMS`。

---

## 8. 跟其他專案協作

- 回報問題的格式：環境（機型、韌體 commit、版本）、現象、指令輸出、推測、建議。（HANDOFF-LOCAL 第 6 節）
- 需要轉給別的專案的內容，放在可以一鍵複製的區塊。（HANDOFF-LOCAL 第 7 節）
- 每個平台的整合由那個平台的維護者寫，寫進 PORTING 的「已知平台」表。（PORTING.md 第 9 節）
