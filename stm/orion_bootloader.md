# Orion：ブートローダー（Main 基板）

Orion（4輪オムニホイール）の Main 基板（STM32G474）は、User Flash の先頭に基板専用のアプリケーションブートローダー（`Bootloader/`）を置き、通常アプリを A/B の 2 スロットで持つ。CM4 からの UART 更新、PENDING と CONFIRM、起動に失敗したときのロールバックを担当する。以下の Flash 配置、更新中の安全動作、ビルド・導入手順は Orion の実装である。更新手順の全体は [MCU ファームウェア更新](../common/firmware_update.md)、機種別の入口は [STM 概要](overview.md) を参照。

STM32 の System Memory bootloader、BOOT0 による ROM boot、ハードウェアのバンクスワップ（`BFB2`）は使わない。

## Flash 配置

値は `Bootloader/Inc/boot_config.h` にまとめてあり、通常アプリのリンカ設定と metadata 生成スクリプトも同じ値を使う。

| 領域 | アドレス | サイズ | 用途 |
|---|---|---:|---|
| Bootloader | `0x08000000`〜`0x08007FFF` | 32 KB | 更新中の安全動作、起動判定、UART 更新 |
| Slot A | `0x08008000`〜`0x0803FFFF` | 224 KB | 通常アプリ |
| Slot B | `0x08040000`〜`0x08077FFF` | 224 KB | 通常アプリ |
| Metadata | `0x08078000`〜`0x0807FFFF` | 32 KB | Slot A metadata（先頭）、Slot B metadata（`+0x800`）、control record（`+0x1000`） |

- 通常アプリのリンカ `STM32G474RETX_FLASH.ld` は Slot A（`0x08008000`、224 KB）を指す。Slot B 用には `APP_FLASH_ORIGIN` で origin を差し替える。
- 通常アプリは `SystemInit()` で `SCB->VTOR = &g_pfnVectors` とし、自分がリンクされたスロットのベクタを使う。
- アプリ先頭の `+0x400` には `.fw_version`（`FWVR` magic と build ID）を置く（[MCU ファームウェア更新](../common/firmware_update.md#バージョン確認fwvr)）。

### metadata と control record

- **metadata**（スロットごと）：magic `OFW1`、format version、record size、slot、image base、image size、image CRC32C、generation、state、record CRC32C。state は `EMPTY` / `RECEIVING` / `VERIFIED` / `PENDING` / `CONFIRMED` / `INVALID`。
- **control record**：magic、format version、generation、優先スロット（`preferred_slot`）、起動試行回数（`boot_attempts`）、record CRC32C。

## 起動判定

`Bootloader/Src/main.c` は次の順に処理する。

1. [更新中の安全動作](#更新中の安全動作) に従い、検証中 LED を点ける。
2. バックアップレジスタ `TAMP->BKP0R/BKP1R` から要求を読み、消去する。
   - `BOOT_REQUEST_UPDATE`：UART 更新モードに入る（[UART 更新](#uart-更新ofw1)）。
   - `BOOT_REQUEST_CONFIRM`：指定スロットが PENDING なら CONFIRMED にし、control record の試行回数を 0 にする。
3. 起動するスロットを選ぶ（`boot_select_slot()`）。
   - control record が有効ならその優先スロット、無効なら有効なスロットのうち generation が新しいほうを選ぶ。選んだスロットが無効なら、もう一方を試す。
   - 選んだスロットが PENDING で、すでに 3 回以上起動を試していたら、もう一方の CONFIRMED のスロットへ戻す（ロールバック）。戻せるスロットがなければ起動しない。
4. PENDING のスロットを起動するときは、control record の試行回数を 1 増やす。
5. そのスロットへジャンプする。有効なスロットがなければエラー LED を点けて停止する。

スロットが有効とみなされるのは、次をすべて満たすときである（`boot_slot_is_valid()`）。

- magic `OFW1`、format version、record size が一致する
- slot と image base がそのスロットのもので、image size が 8 バイト以上 224 KB 以下
- state が CONFIRMED（起動判定では PENDING も可）
- metadata の CRC32C が一致する
- 初期 MSP が SRAM1 か CCMRAM の中で、8 バイト境界に揃っている
- Reset_Handler が Thumb アドレスで、イメージの範囲内にある
- イメージ全体の CRC32C が一致する

## UART 更新（OFW1）

通常アプリは CM4 から `FWUP` の更新要求を受けると、現在の稼働スロットと要求を bootloader に引き渡して再起動する。更新には [CM4 との UART 通信](orion_board_spec.md#cm4-との通信) を使い、**稼働していないほうのスロット**へ書き込む。

- フレーム：magic `OFW1`、version、type、sequence、length のヘッダ（12 バイト、CRC16 付き）と payload。応答には CRC32C を付ける。
- メッセージ：`INFO`（両スロットの状態）、`BEGIN`（対象スロットとイメージサイズを指定し、metadata と必要なページを消去）、`CHUNK`（offset と chunk CRC32C 付き、最大 896 バイト。offset は連続していなければならない）、`FINALIZE`（ベクタと全体 CRC32C を検証し、PENDING の metadata を書いて control record の優先スロットを書込み先にする）、`REBOOT`。
- 再起動後、PENDING のスロットが起動する。CM4 は新しいアプリが動いていることを確認してから `FWUP` の CONFIRM 要求を送り、CONFIRMED にする。

CM4 側のツールは `cm4/firmware/main_ab_updater.py`。

## 更新中の安全動作

- bootloader では通常の走行制御を開始しない。
- 更新開始から通常アプリへの切り替えまで、ブザーを停止状態に保つ。Main 自身の更新中だけでなく、CAN ノードの更新を中継している間も同じである。
- SWD によるデバッグ接続を維持する。
- 起動するスロットの検証中と、起動できるスロットがないときは [起動判定](#起動判定) に従って LED で状態を示す。

## ビルド

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\Script\build_bootloader.ps1 -Rebuild
powershell -NoProfile -ExecutionPolicy Bypass -File .\Script\build_slot_a.ps1 -Configuration Debug -Rebuild
```

`build_slot_a.ps1` は Slot A のベクタと領域を検査し、`.bin` から CRC32C 付きの metadata を生成する。A/B 両方を作るときは `build_slot_b.ps1` を使う（[Orion：STM 概要](orion_overview.md#ビルドと書込み)）。

## 初回導入

最初に dry-run し、Flash 全体、CRC32C、Option Bytes を退避する。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\Script\install_main_bootloader.ps1 -Configuration Debug
```

出力された backup と IO 表を確認したあとでだけ、明示的に `-Execute` を付ける。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\Script\install_main_bootloader.ps1 -Configuration Debug -Execute
```

`-Execute` を付けなければ、Flash への書込みもリセットも行わない。初回導入が終わるまでは、既存の `flash.ps1` を使わない。

導入後に ST-Link で Slot A を更新するときは、先に `build_slot_a.ps1` を実行してから、明示的に次を使う。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\Script\flash.ps1 -Configuration Debug -BootloaderInstalled
```

bootloader 領域の書込み保護（WRP）はまだ設定していない。
