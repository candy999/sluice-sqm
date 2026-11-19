# Sluice SQM 設計定案

2026-11-19 定案。之後的變更直接改這份文件，並在 commit 訊息說明原因。
介面樣式見 `docs/mockups/luci-ui.html`；跨平台的規則見 `docs/porting-lessons.md`。

## 1. 定位

- 開了就能用的 OpenWrt 流量塑型，**取代** sqm-scripts（不是分支或附加套件，兩者不可同時啟用）
- 一般人只要一個開關；進階使用者可以調優先順序與頻寬範圍；專家可以直接改 CAKE 參數
- 授權：**GPL-2.0-only**（daemon 與 luci-app-sluice 相同）。貢獻用 DCO（`Signed-off-by`），不用 CLA

## 2. 平台

| 項目 | 定案 |
|---|---|
| 目標系統 | OpenWrt 25.12 以上（apk 套件） |
| 首發架構 | x86_64、aarch64、armv7（musl） |
| MIPS | **不支援**。MT7621 這類晶片跑 CAKE 大約 150～400 Mbps 就到頂，跟不上現在的光纖速率 |
| 其他平台 | 不由本專案維護，但保持容易移植：所有平台只透過第 5 節的 API 與 `sluice` 指令整合（見 porting-lessons） |

## 3. 架構：Rust 做判斷，核心做轉送

```
          ┌──────────── Rust：sluice daemon（判斷） ────────────┐
          │ 線路偵測  校準  autorate  CPU 保護  裝置輪廓  P2P 偵測 │
          └──────┬───────────────┬──────────────────┬─────────┘
       改頻寬（netlink）  更新 nft 集合（裝置、網域、P2P）   讀 conntrack 事件、/proc、計數器
                 ▼               ▼                  ▲
          ┌──────────── Linux 核心：每個封包的工作 ─────────────┐
          │ CAKE（上傳 egress、下載 IFB）  nftables 分類  conntrack │
          │ act_ctinfo 還原 DSCP  flow offloading                   │
          └─────────────────────────────────────────────────────┘
```

- **每個封包的處理一律留在核心**（CAKE、nftables、conntrack）。原因：
  - 速度：核心路徑不用把封包送到使用者空間
  - 硬體：flow offloading、各家網卡驅動、核心版本差異都由核心處理，Sluice 不用跟著每種硬體改
  - 穩定：daemon 當掉時轉送照常，看門狗會把塑型拿掉（第 4.6 節），不會斷網
- **Rust 只做判斷與控制**：看延遲、看負載、看連線事件，然後調整核心的參數與集合。不在 Rust 裡逐一處理封包
- 不寫 eBPF。之後真的需要「每個封包都要算」的判斷時再評估（Aya 在沒有 BTF 的 OpenWrt 核心上可以用）

### 3.1 程式組成

單一執行檔 `sluice`，子指令：

| 子指令 | 用途 |
|---|---|
| `sluice daemon` | 常駐（procd） |
| `sluice rpcd` | rpcd exec plugin（LuCI 用） |
| `sluice api <GET\|POST> <路徑> [JSON]` | 所有平台共用的 API 橋接 |
| `sluice health` | 健康檢查，結束碼 0／1／2 |
| `sluice status --json` | 狀態 |
| `sluice notify wan-up\|wan-down` | 給 hotplug 用 |
| `sluice paths`、`sluice uci-migrate` | 整合與升級 |

Cargo workspace：

| crate | 內容 |
|---|---|
| `core` | 純邏輯：線路判斷、autorate、CPU 檔位、分類規則產生、裝置輪廓。可在電腦上用模擬曲線測試 |
| `sys` | 碰系統：netlink（改頻寬、conntrack 事件）、ICMP socket、/proc、UCI 文字解析、呼叫 `tc`／`nft` |
| `bin` | 子指令進入點 |

不用 tokio；單執行緒事件迴圈（mio）。體積目標 1 MB 以下（opt-level z、LTO、panic=abort、strip）。

## 4. 塑型

### 4.1 佇列
- 上傳：WAN egress 掛 CAKE。下載：WAN ingress → `act_ctinfo` 還原 DSCP → mirred 到 IFB → CAKE `ingress`
- `diffserv4`；上傳 `dual-srchost nat`、下載 `dual-dsthost nat`（同一台裝置開再多連線，也只拿到自己那一份）
- 有 `cake_mq`（25.12 已 backport）且實測比單佇列好時改用；先在目標硬體上跑分再決定預設
- 建立 qdisc 用 `tc` 指令；**改頻寬這個常用動作直接用 netlink**，不 fork `tc`
- reload 時只改變動的參數，不拆掉重建

### 4.2 Overhead
- 依連線方式推算，**寧可高估**：PPPoE over 乙太網路 46（38＋8），有 VLAN 再加 4；mpu 84
- ATM（ADSL）只提供手動選項

### 4.3 頻寬
- **校準**：首次啟用時跑約 3 分鐘（閒置延遲 → 下載滿載 → 上傳滿載 → 雙向同時 → 計算與 CPU 檢查）。用系統的 `uclient-fetch` 下載與上傳，daemon 只讀介面計數與量延遲，不自帶 TLS。**系統時間還沒對時就不校準**
- **不校準（自動學習）**：從線路的最高速度開始；只在有負載時往下試探；降速後延遲變好就停，最多降 30% 都沒改善就判定瓶頸不在這段線路，恢復原速並暫停試探
- **要不要塑型**：看校準時「滿載比閒置多出的延遲」。上下行都小於 5 ms 就不塑型；只有一個方向變差就只塑那個方向

### 4.4 Autorate
- 多個測點（168.95.1.1、1.1.1.1、8.8.8.8、9.9.9.9 等）的 RTT，綁定 WAN 介面發送；測點會回 ICMP timestamp 時用單向延遲
- 每個測點各自維護基線（下降快跟、上升慢跟）；取各測點延遲增量的中位數
- 門檻依「取捨」：延遲優先 8 ms、平衡 15 ms、頻寬優先 30 ms
- 判斷方向：延遲超標時，只壓**負載高的那個方向**；兩邊都高就兩邊都壓
- 超標且負載高 → 降到實際速率 × 0.9，冷卻 2～3 秒；沒超標且負載 > 75% → 小步往上；負載低 → 不動
- 首頁分數用被動資料（有負載時延遲增量的 p95），不另外打滿線路測試

### 4.5 CPU 保護
| 檔位 | 做法 |
|---|---|
| L0 完整 | 上下行塑型、四級優先、全部分類規則 |
| L1 停用動態分類 | 關掉大流量與 P2P 偵測、服務規則；保留裝置規則 |
| L2 簡化佇列 | `besteffort`、關掉每台裝置平分 |
| L3 只管上傳 | 停掉下載方向（IFB） |
| L4 暫停塑型 | 全部不塑型，硬體加速維持關閉；10、20、40 分鐘重試 |
| L5 交還硬體加速 | L4 時 CPU 仍滿載，或一天內第 3 次進 L4：還原使用者原本的 offload 設定，每天凌晨離峰重試一次 |

- 降檔：任一核心 softirq 超過 90% 持續 5 秒
- 升檔：回穩 10 分鐘後升一檔；升上去又馬上觸發，下次等待加倍

### 4.6 衝突與安全
- 偵測並提示：sqm-scripts、qosify、nft-qos 已啟用；WAN 上已有其他 qdisc；IFB 名稱衝突
- **硬體 offload**：記下原本的設定再關閉（它會讓封包完全繞過塑型）；L5 時還原
- **軟體 offload**：保留開啟。CAKE 與 `act_ctinfo` 照常作用，分類在連線開頭就完成，不受影響
- **看門狗**：daemon 停止超過 30 秒，就拿掉 qdisc 讓網路回到不塑型；停止服務時清乾淨
- VPN：v1 只偵測並說明（路由器上的 WireGuard 會讓每台裝置平分失效），之後處理 fwmark

## 5. 流量分類

### 5.1 四級
| 順序 | 類別 | CAKE tin | DSCP |
|---|---|---|---|
| 1 | 電話與視訊 | Voice | EF |
| 2 | 線上遊戲 | Video | CS4 |
| 3 | 影片串流（含認不出來的） | Best Effort | CS0 |
| 4 | 檔案下載（含 P2P） | Bulk | CS1 |

CAKE 的 tin 不是嚴格排隊：前三級各有上限（1/4、1/2、全部），超過就照下一級處理；Bulk 至少保留 1/16。

### 5.2 做法：每條連線只判斷一次
- nftables 只在連線的**前幾個封包**比對（`ct packets` 很小時），結果寫進 conntrack mark；之後的封包只讀標記（上傳 `meta dscp set` 從 ct mark 取，下載由 `act_ctinfo` 還原）
- 比對全部用 nft **集合／vmap** 查表，不寫一長串規則
- daemon 只負責更新集合內容，不碰封包

### 5.3 判斷依據（由準到不準）
1. **使用者指定**：裝置設成「全部當線上遊戲／全部當檔案下載」
2. **目的網域**：dnsmasq `nftset` 把網域解析結果放進集合
   - 影片串流：googlevideo.com、nflxvideo.net、Disney+、Twitch、巴哈姆特動畫瘋的 CDN 網域
   - 檔案下載：Steam、PlayStation、Xbox、Nintendo 的下載網域，Windows Update、Apple 軟體更新
   - 電話與視訊：Meet、Teams、Zoom、LINE、FaceTime、Discord 語音的網域與已公開的 UDP 埠
3. **裝置輪廓**（daemon 在 Rust 裡建立，結果寫進 nft 集合）
   - 來源：MAC 廠牌（OUI）、DHCP 指紋（要求的選項清單、vendor class、主機名稱，經 dnsmasq 的 dhcp-script 取得）、mDNS／SSDP 廣播（`_googlecast`、`_airplay`、DLNA 代表電視或串流盒）
   - 類型：電視／串流盒、遊戲主機、電腦、手機、NAS、IoT
   - 用法：**認不出網域的大流量**，電視／串流盒歸影片串流；電腦、NAS 歸檔案下載；遊戲主機的小封包 UDP 歸線上遊戲
   - 隨機 MAC 的手機只用 DHCP 指紋與行為，認不出來就照一般流量
4. **行為特徵**
   - 大流量：單一連線超過約 10 MB 且平均封包很大 → 檔案下載；**已認出是影片串流的不降級**
   - 遊戲：小封包、穩定的雙向 UDP → 線上遊戲
   - **P2P（BitTorrent 等）**：同一台裝置在短時間內連到很多個不同的對外 IP、用非 80／443 的高埠號、TCP 與 UDP 都有（含 uTP、DHT）。daemon 從 conntrack 事件統計，超過門檻就把這台裝置的「非 80／443 高埠號連線」歸檔案下載，一段時間沒有這種行為就解除。加密的 P2P 也看得出來，因為看的是連線型態而不是內容
   - 就算沒認出 P2P，每台裝置平分頻寬（`dual-*host`）也會讓跑 P2P 的電腦只拿到自己那一份

### 5.4 之後再做
- 讀連線前幾個封包裡的 TLS SNI／QUIC 網域（`nfq`，只送前幾個封包到使用者空間），補上裝置用加密 DNS 時認不出網域的情況
- 中華電信 MOD 的 VLAN／介面不經過排程

## 6. API 與介面

- daemon 開 Unix socket；所有平台經 `sluice api` 呼叫，路徑白名單寫在核心裡（查看類、動作類分開，動作只收 POST）
- 約定寫在 `docs/integration.md`（之後建立）：JSON 只加欄位不改名；行為不相容時整數 `api` 加 1
- 回應格式沿用 padavan-tw：`ok`、`ts`、`available`、`warming_up`；單位寫在欄位名稱（`_ms`、`_kbps`、`_pct`）；狀態值 `ok`／`warn`／`bad`／`unknown`／`off`；時間序列 `{step, start, values}`
- 校準、自測、CPU 跑分：背景執行，立刻回 `runId`，進度在 `/status`
- 介面共用的純函式（分數說法、檔位說明、單位、SVG 圖表）放 `data/shared/sluice-ui-core.js`，附測試資料
- LuCI：client-side JS view，四個分頁（總覽、一般、進階、專家）；menu.d、acl.d 用 JSON；放在本 repo
- 介面規則：不載外部資源、輪詢至少 2 秒且頁面不在前景時停止、手機表格改卡片、不顯示假資料、用語照 porting-lessons 第 5.3 節

## 7. 設定與儲存

- `/etc/config/sluice`；核心自己解析 UCI 文字，不依賴 `uci` 指令或 libuci
- 學到的資料（校準結果、頻寬範圍、CPU 跑分）只在明顯改變時寫 Flash，一天最多一次；執行狀態放記憶體
- `SLUICE_ON_STATE_WRITE`：寫入後執行的指令（給設定放在記憶體的韌體），只能用環境變數設定

## 8. 測試

1. `core` 單元測試：模擬光纖、PPPoE 晚間壅塞、行動網路（Mahimahi 公開 trace）、Starlink 的延遲與容量曲線
2. CI 整合測試：network namespace＋netem 模擬瓶頸，跑真的 CAKE、IFB、nftables
3. 實機：光世代 500/500 FTTH（PPPoE）

## 9. 打包

- `openwrt/` feed 目錄：`sluice`、`luci-app-sluice` 兩個套件 Makefile，用 SDK 的 rust-package.mk
- 依賴：kmod-sched-cake、kmod-ifb、kmod-sched-ctinfo、tc、uclient-fetch
- 每個 Release 附通用 Linux 包、`sluice-<版本>-source.tar.xz`（含 `cargo vendor`）、`SHA256SUMS`

## 10. 開發順序

1. `core`＋模擬測試（autorate、CPU 檔位、學習模式）
2. daemon 用 `tc` 建立塑型、netlink 改頻寬、看門狗
3. API（Unix socket、`sluice api`）＋rpcd＋LuCI 總覽頁
4. 自動偵測：線路與 overhead、校準、要不要塑型、CPU 保護、offload 處理
5. 分類：nftables 集合與 ct mark、網域清單、裝置輪廓、P2P 與大流量偵測

## 11. 待確認

| 項目 | 怎麼確認 |
|---|---|
| 使用者的路由器型號 | 問使用者；決定 500/500 時 CPU 保護的預設門檻 |
| `cake_mq` 在目標硬體上是否比單佇列好 | 實機跑分 |
| `tc-tiny` 能不能設定 CAKE、輸出 `-j` JSON | 在 25.12 上試；不行就依賴 `tc-full` |
| dnsmasq 的 dhcp-script 拿得到哪些 DHCP 指紋欄位 | 讀 dnsmasq 文件並實測 |
| P2P 判斷門檻（多少個不同對外 IP、多長時間） | 用真實 BitTorrent 流量實測，並確認不會誤判遊戲主機與視訊會議 |
