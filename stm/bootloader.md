# ブートローダー（Main 基板）

Main 基板（STM32G474）は、User Flash の先頭に基板専用のアプリケーションブートローダー（`Bootloader/`）を置き、通常アプリを A/B の 2 スロットで持つ。CM4 からの UART 更新、PENDING と CONFIRM、起動に失敗したときのロールバックを担当する。更新手順の全体は [MCU ファームウェア更新](../common/firmware_update.md) を参照。

STM32 の System Memory bootloader、BOOT0 による ROM boot、ハードウェアのバンクスワップ（`BFB2`）は使わない。

## Flash 配置

値は `Bootloader/Inc/boot_config.h` にまとめてあり、通常アプリのリンカ設定と metadata 生成スクリプトも同じ値を使う。

| 領域 | アドレス | サイズ | 用途 |
|---|---|---:|---|
| Bootloader | `0x08000000`〜`0x08007FFF` | 32 KB | 安全 IO、起動判定、UART 更新 |
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

1. `board_io_init_safe()` で安全 IO を設定し、検証中 LED（PC13）を点ける。
2. バックアップレジスタ `TAMP->BKP0R/BKP1R` から要求を読み、消去する。
   - `BOOT_REQUEST_UPDATE`：UART 更新モードに入る（[UART 更新](#uart-更新ofw1)）。
   - `BOOT_REQUEST_CONFIRM`：指定スロットが PENDING なら CONFIRMED にし、control record の試行回数を 0 にする。
3. 起動するスロットを選ぶ（`boot_select_slot()`）。
   - control record が有効ならその優先スロット、無効なら有効なスロットのうち generation が新しいほうを選ぶ。選んだスロットが無効なら、もう一方を試す。
   - 選んだスロットが PENDING で、すでに 3 回以上起動を試していたら、もう一方の CONFIRMED のスロットへ戻す（ロールバック）。戻せるスロットがなければ起動しない。
4. PENDING のスロットを起動するときは、control record の試行回数を 1 増やす。
5. そのスロットへジャンプする。有効なスロットがなければエラー LED（PC14）を点けて停止する。

スロットが有効とみなされるのは、次をすべて満たすときである（`boot_slot_is_valid()`）。

- magic `OFW1`、format version、record size が一致する
- slot と image base がそのスロットのもので、image size が 8 バイト以上 224 KB 以下
- state が CONFIRMED（起動判定では PENDING も可）
- metadata の CRC32C が一致する
- 初期 MSP が SRAM1 か CCMRAM の中で、8 バイト境界に揃っている
- Reset_Handler が Thumb アドレスで、イメージの範囲内にある
- イメージ全体の CRC32C が一致する

### ジャンプ時の CPU 状態

bootloader はジャンプの準備中に全割り込みを禁止するが、その状態を通常アプリへ引き継いではならない。SysTick と NVIC の enable/pending を消去し、VTOR をスロットの先頭に設定したあと、`CONTROL`、`BASEPRI`、`FAULTMASK`、`PRIMASK` を 0 に戻し、スロットの MSP を設定して Reset_Handler へ分岐する（`boot_jump_to_slot()`）。`PRIMASK = 1` のまま引き継ぐと、TIM7、USART2 受信、UART の DMA 送信が動かない。

## UART 更新（OFW1）

通常アプリは CM4 から `FWUP` の更新要求を受けると、現在の稼働スロットを `BKP1R` に、`BOOT_REQUEST_UPDATE` を `BKP0R` に書いて再起動する。bootloader は USART2（PB3/PB4、pin swap、1 Mbps、HSI 16 MHz から生成）を自前で初期化し、**稼働していないほうのスロット**へ書き込む（`boot_uart_update.c`）。

- フレーム：magic `OFW1`、version、type、sequence、length のヘッダ（12 バイト、CRC16 付き）と payload。応答には CRC32C を付ける。
- メッセージ：`INFO`（両スロットの状態）、`BEGIN`（対象スロットとイメージサイズを指定し、metadata と必要なページを消去）、`CHUNK`（offset と chunk CRC32C 付き、最大 896 バイト。offset は連続していなければならない）、`FINALIZE`（ベクタと全体 CRC32C を検証し、PENDING の metadata を書いて control record の優先スロットを書込み先にする）、`REBOOT`。
- 再起動後、PENDING のスロットが起動する。CM4 は新しいアプリが動いていることを確認してから `FWUP` の CONFIRM 要求を送り、CONFIRMED にする。

CM4 側のツールは `cm4/firmware/main_ab_updater.py`。

## 安全 IO

`SystemInit()` から `board_io_init_safe()` を呼び、C ランタイムや HAL より前に設定する。各ポートのボンディング済み GPIO はすべて、出力、入力、analog、SWD 保持のいずれかに分類し、マスクの重複と分類漏れを `_Static_assert` で検出する。

| GPIO | bootloader での状態 | 理由 |
|---|---|---|
| PA4 | Output High | IMU CS を非選択にする |
| PA15 | Output Low | 用途未確定の既存出力の安全値 |
| PB2/PB10/PB14 | Output Low | 用途未確定の既存出力の安全値 |
| PB7 | Output Low | status LED を消灯 |
| PC0/PC5 | Output Low | 既存出力、IMU FSYNC |
| PC12 | Output Low | ブザー PWM を止めた状態に固定 |
| PC13/PC14 | Output Low のあと status 表示 | 検証中 LED / エラー LED |
| PA10/PB5/PB6/PC4/PD2 | Input no-pull | DIP、IMU INT。外部回路のレベルを保つ |
| PB8 | Input pull-down | BOOT0 兼用ピンを実行開始後も Low 側に固定 |
| PA13/PA14 | reset 状態を保持 | SWDIO/SWCLK |
| その他のボンディング済み GPIO | Analog no-pull | 使わない。浮いたデジタル入力を残さない |

- FDCAN（PA11/PA12、PB12/PB13）と USART2（PB3/PB4）も安全 IO では analog にする。UART 更新に入るときだけ、bootloader が PB3/PB4 を alternate function に切り替える。
- 更新を始めてから通常アプリへジャンプするまで、TIM5 は強制リセットしてクロックを止め、PC12 は GPIO Output Low を保つ。G474 自身の更新中だけでなく、G474 が F303 の更新を中継している間もブザー PWM を始めてはならない。
- PA15、PB2、PB10、PB14、PC0 は回路図で用途を確かめていないので、現行の `MX_GPIO_Init()` と同じ Low を暫定の安全値にしている。

## ビルド

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\Script\build_bootloader.ps1 -Rebuild
powershell -NoProfile -ExecutionPolicy Bypass -File .\Script\build_slot_a.ps1 -Configuration Debug -Rebuild
```

`build_slot_a.ps1` は Slot A のベクタと領域を検査し、`.bin` から CRC32C 付きの metadata を生成する。A/B 両方を作るときは `build_slot_b.ps1` を使う（[STM 概要](overview.md#ビルドと書込み)）。

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
