# CLAUDE.md：Sluice SQM 開發交接

每次開工先讀這份，再讀 `docs/design.md`。**設計以 `docs/design.md` 為唯一依據**；這份只放開發規範與現況，兩者衝突時以 design.md 為準並回頭修正這份。

## 溝通

- 回覆用繁體中文（台灣用語）。程式碼、識別字、commit message 用英文；README 以繁體中文為主（`README.md`），另附英文版 `README.en.md`。
- 使用者有電子工程背景，網路與 Linux 概念不用從頭解釋。
- 用語照 `docs/porting-lessons.md` 第 5.3 節。

## 專案一句話

OpenWrt 專用、開了就能用的流量塑型，體驗類似 cFosSpeed。大多數參數自動判斷，一般人只要開一個開關，進階使用者與專家可以細調。這是**取代** sqm-scripts 的獨立實作，不是它的分支或附加套件。

## 文件地圖

| 檔案 | 用途 |
|---|---|
| `docs/design.md` | 設計定案：架構、塑型、CPU 保護、能力偵測、分類、API、測試、打包、開發順序 |
| `docs/porting-lessons.md` | 從 kuroshio-dns／padavan-tw 學到的守則：API 約定、UCI 文字解析、Flash 寫入、健康檢查、介面規則 |
| `docs/mockups/luci-ui.html` | LuCI 介面草稿，四個分頁的樣式與用語 |
| `README.md`（繁中，首頁預設）、`README.en.md`（英文） | 對外介紹，以繁中為主；設計改了兩份要一起更新 |

## 已定案（摘要，細節見 design.md）

| 項目 | 決定 |
|---|---|
| 名稱 | 顯示 Sluice SQM；repo `sluice-sqm`；執行檔與套件 `sluice`、`luci-app-sluice`；設定 `/etc/config/sluice` |
| 授權 | GPL-2.0-only，貢獻用 DCO（`Signed-off-by`），不用 CLA |
| 平台 | OpenWrt 25.12 以上；x86_64、aarch64、armv7（musl）；**不支援 MIPS** |
| 分工 | Rust 只做判斷與控制；每個封包的處理留在核心（CAKE、nftables、conntrack、act_ctinfo） |
| 不做 | eBPF（暫不需要）、out-of-tree 核心模組、qdisc 與腳本自由組合、多 WAN（v1） |
| 實測環境 | Intel N100＋光世代 500/500 FTTH（PPPoE） |

## 程式結構（尚未建立）

```
sluice-sqm/
├── Cargo.toml                    # workspace
├── crates/
│   ├── core/                     # 純邏輯，不得有任何 I/O
│   ├── sys/                      # netlink、ICMP、/proc、UCI 文字解析、呼叫 tc／nft
│   └── bin/                      # 單一執行檔 sluice 的子指令
├── sim/                          # 光纖、PPPoE 晚間壅塞、行動網路、Starlink 的模擬曲線
├── data/shared/sluice-ui-core.js # 介面共用純函式，附測試資料
├── luci-app-sluice/              # htdocs view、menu.d、acl.d、po/zh_Hant
├── openwrt/                      # feed：sluice、luci-app-sluice 的套件 Makefile
└── docs/
```

子指令：`daemon`、`rpcd`、`api`、`health`、`status --json`、`notify wan-up|wan-down`、`paths`、`uci-migrate`（見 design.md 3.1）。

## 程式規範

- **`core` 不碰系統**，只透過 trait 取得時間、延遲、計數器與套用設定，才能在電腦上用 `sim/` 做決定性測試。這是選 Rust 的主要理由，不要破壞這個邊界。
- 不用 tokio；單執行緒事件迴圈（mio）。
- 函式庫用 `thiserror`，執行檔用 `anyhow`。
- UCI 自己解析文字，不依賴 `uci` 指令或 libuci。
- 建立 qdisc 用 `tc`；改頻寬用 netlink，不 fork `tc`。reload 只改有變動的參數。
- 學到的資料一天最多寫一次 Flash；執行狀態放記憶體。
- 測點、測速端點一律可在 UCI 設定，不寫死。
- Release profile：`opt-level="z"`、`lto=true`、`panic="abort"`、`codegen-units=1`、`strip=true`，stripped binary 目標小於 1 MB。新增依賴前先評估體積。
- `cargo fmt --check`、`cargo clippy -- -D warnings`、`cargo test` 必須通過。
- commit 加 `Signed-off-by`（`git commit -s`）。

## 現況

- **已完成**：命名、定位、授權、完整設計（design.md）、移植守則、LuCI 介面草稿、README。
- **還沒有任何程式碼。**
- **下一步（design.md 第 10 節第 1 項）**：
  1. 建立 Cargo workspace 骨架與 GitHub Actions CI（fmt、clippy、test、三個 target 交叉編譯、回報 binary 大小）
  2. `core`：autorate、CPU 檔位、學習模式，加上 `sim/` 模擬測試

## 待確認（design.md 第 11 節）

- `cake_mq` 在目標硬體上是否比單佇列好（實機跑分）
- `tc-tiny` 能否設定 CAKE 並輸出 `-j` JSON，不行就依賴 `tc-full`
- dnsmasq 的 dhcp-script 拿得到哪些 DHCP 指紋欄位
- P2P 判斷門檻，且不能誤判遊戲主機與視訊會議
