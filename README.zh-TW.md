# Sluice SQM

> 開了就能用的 OpenWrt 智慧流量塑型，可直接取代 sqm-scripts。

[English](README.md)

家裡有人在下載、上傳或掛 BT 的時候，打遊戲、開視訊會議、滑網頁照樣順暢，這就是 Sluice SQM 要做的事。

一般的 SQM 要你自己測頻寬、選 overhead、猜 link layer，還要在一堆 qdisc 和腳本之間做選擇。Sluice SQM 會自己判斷：

- **辨識線路**：讀 WAN 的連線方式（PPPoE、DHCP、VLAN 等），自動帶入 overhead，寧可高估也不低估
- **校準頻寬**：第一次啟用時花約 3 分鐘量測。也可以不校準，讓它在實際使用中自己學
- **即時調整**：持續看多個測點的延遲，只壓真正塞住的那個方向
- **不需要就不塑型**：線路滿載時延遲幾乎沒增加，就保留全速；只有一個方向變差，就只塑那個方向
- **認得硬體**：實際試用這份韌體的核心能做什麼（CAKE、cake_mq、IFB、ctinfo、HTB），缺什麼就降到做得到的最好模式
- **保護 CPU**：路由器跟不上時，先調整（分散中斷、多佇列 CAKE、降低上限），不夠再依序拿掉功能，從影響最小的開始
- **依流量種類排順序**：電話與視訊 → 線上遊戲 → 影片串流 → 檔案下載，依網域、裝置類型與連線行為判斷，加密的 P2P 也認得出來
- **不會讓你斷網**：每個封包的處理都留在核心。daemon 停止時，看門狗會把塑型拿掉，不會留下壞掉的佇列

> Sluice SQM 是獨立實作，不是 sqm-scripts 的分支。
> 請勿與 sqm-scripts、qosify、nft-qos 同時啟用，偵測到時 Sluice 會提醒你。

## 四個分頁，三種控制程度

| 分頁 | 適合 | 你會看到 |
|---|---|---|
| 總覽 | 所有人 | 延遲等級、目前速率、偵測到什麼以及為什麼這樣設 |
| 一般 | 所有人 | 一個開關、取捨（延遲優先 ↔ 頻寬優先） |
| 進階 | 玩家與愛好者 | 每台裝置的優先順序、頻寬上下限、調整靈敏度 |
| 專家 | 網路老手 | 直接改 CAKE 參數、HTB＋fq_codel 備案、實驗性的 L4S |

## 運作方式

Rust 只負責判斷與調整，每個封包的處理一律交給 Linux 核心（CAKE、nftables、conntrack）。這樣速度快，各家網卡與硬體加速由核心處理，daemon 出問題也不影響轉送。完整設計見 [docs/design.md](docs/design.md)。

## 開發狀態

🚧 **設計已定案，開始實作。** 目前還沒有可安裝的套件。

| | |
|---|---|
| 目標系統 | OpenWrt 25.12 以上（apk 套件） |
| 支援架構 | x86_64、aarch64、armv7（musl） |
| 不支援 | MIPS（例如 MT7621），這類晶片跑 CAKE 約 150～400 Mbps 就到頂 |
| 實測硬體 | Intel N100＋光世代 500/500 FTTH（PPPoE） |

## 文件

| 檔案 | 內容 |
|---|---|
| [docs/design.md](docs/design.md) | 設計定案 |
| [docs/porting-lessons.md](docs/porting-lessons.md) | 從黑潮 DNS 移植到 Padavan 學到、套用在本專案的守則 |
| [docs/mockups/luci-ui.html](docs/mockups/luci-ui.html) | LuCI 介面草稿（用瀏覽器開啟） |
| [CLAUDE.md](CLAUDE.md) | 開發交接與程式規範 |

## 參與貢獻

採用 [DCO](https://developercertificate.org/)，commit 時加上 `git commit -s` 簽署即可，不需要 CLA。

## 授權

[GPL-2.0-only](LICENSE)，`sluice` 與 `luci-app-sluice` 相同。
