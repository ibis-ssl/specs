# STM（Main 基板）概要

G474_Orion_main リポジトリ（Main 基板 STM32G474RETx のファームウェア、STM32CubeIDE プロジェクト）の使い方をまとめる。

| 文書 | 内容 |
|---|---|
| [ハードウェア仕様](board_spec.md) | MCU、クロック、ピン、ペリフェラル、CAN ID、NVIC |
| [速度制御](speed_control.md) | `MAIN_MODE_FULL_AI_CONTROL` の速度制御 |
| [ブートローダー](bootloader.md) | Flash 配置、A/B 起動判定、安全 IO |
| [CM4 との UART](uart_cm4.md) | USART2 の受信・送信の実装と診断 |
| [診断機能](diagnostics.md) | DRIVE_LOG、エンコーダーノイズ、静止時の角度誤差クリア |

上位からの指令の意味は [制御パケット](../common/control_packet.md)、テレメトリは [フィードバックパケット](../common/feedback_packet.md)、停止条件は [停止判定](../common/safety_stop.md) を参照。

## モード切り替え（ロータリースイッチ）

スイッチの値がそのまま `main_mode`（`management.h` の列挙）になる。エラー時は `MAIN_MODE_ERROR` に移る。

| 値 | `main_mode` | モード |
|---|---|---|
| 0 | `MAIN_MODE_FULL_AI_CONTROL` | 通常動作モード |
| 1 | `MAIN_MODE_MANUAL_CONTROL` | デバッグ用マニュアルモード（Communication Tester で動かすためのモード）。加速度をロボット側で適用する。ローカルカメラモードのテスト用 |
| 2 | `MAIN_MODE_CMD_DEBUG_MODE` | デバッグ用（走行出力なし） |
| 3 | `MAIN_MODE_MOTOR_TEST` | モーター動作テスト。UI のボタンの方向へ移動する。真ん中は旋回 |
| 4 | `MAIN_MODE_DRIBBLER_TEST` | ドリブラーモーター動作テスト |
| 5 | `MAIN_MODE_KICKER_AUTO_TEST` | キッカー動作テスト。ドリブラーとボールセンサーあり（ボールセンサーが反応したらキック） |
| 6 | `MAIN_MODE_KICKER_MANUAL` | キッカー動作テスト。ボールセンサーなし（ボタンを押したらキック） |
| 7 | `MAIN_MODE_LATENCY_CHECK` | レイテンシ確認 |
| 8 | `MAIN_MODE_MOTOR_CALIBRATION` | モーターキャリブレーション。左右同時に実行してはいけない。左右どちらかのボタンを長押しで開始 |
| 9 | `MAIN_MODE_ERROR` | エラーモード（モード遷移のデバッグ用） |

スイッチ値の読み方は [ハードウェア仕様](board_spec.md#dip-スイッチ--アナログスイッチ) を参照。

## LED 表示

| 色 | 意味 |
|---|---|
| 白 | IMU キャリブレーション中 |
| 緑 | PC（AI）との通信確立中 |
| 赤 | エラー状態 |
| 青 | メインルーチンの動作確認用の点滅（TIM7 割り込み） |

## ビルドと書込み

前提：STM32CubeIDE 1.17.0 と GNU Tools for STM32、STM32CubeCLT 1.21.0（STM32_Programmer_CLI）。

`Script/` に、ビルドと書込みをまとめた PowerShell スクリプトがある。

```powershell
# ビルド
powershell -ExecutionPolicy Bypass -File .\Script\build.ps1 -Configuration Debug
powershell -ExecutionPolicy Bypass -File .\Script\build.ps1 -Configuration Release -Rebuild
powershell -ExecutionPolicy Bypass -File .\Script\build_bootloader.ps1 -Rebuild
powershell -ExecutionPolicy Bypass -File .\Script\build_slot_a.ps1 -Configuration Debug -Rebuild

# 接続確認
powershell -ExecutionPolicy Bypass -File .\Script\flash.ps1 -List
powershell -ExecutionPolicy Bypass -File .\Script\flash.ps1 -ConnectOnly

# ビルドしてから書込み（bootloader 導入済みの基板）
powershell -ExecutionPolicy Bypass -File .\Script\build_and_flash.ps1 -Configuration Debug -BootloaderInstalled
powershell -ExecutionPolicy Bypass -File .\Script\build_and_flash.ps1 -Configuration Release -Rebuild -BootloaderInstalled

# UART ログ受信
powershell -ExecutionPolicy Bypass -File .\Script\monitor_uart.ps1 -Port COM60 -BaudRate 2000000 -DurationSec 5
powershell -ExecutionPolicy Bypass -File .\Script\monitor_uart.ps1 -Port COM60 -BaudRate 2000000 -MaxLines 100 -LogPath .\uart_log.txt
```

- 通常アプリは Slot A（`0x08008000`）にリンクされる。**通常アプリだけを CLI で書き込むと metadata と食い違うので禁止**。書込みは上のスクリプトを使う。
- `build_and_flash.ps1` は、bootloader を導入済みの基板でだけ `-BootloaderInstalled` を付けて使う。初回は必ず `install_main_bootloader.ps1` を使う（[ブートローダー](bootloader.md#初回導入)）。
- bootloader、Debug 版 Slot A アプリ、metadata をまとめて再ビルドし、Flash 全体を退避してから一括で書き込むときは `flash_all.ps1` を使う。確認オプションなしで直ちに書き込むので、対象の基板と安全な状態を確認してから起動する。
- `build_slot_b.ps1` は A/B を同じ build ID で生成し、両スロットの metadata も必ず作り直す。ビルドごとに Git hash と dirty 状態が `Script/Logs/Build/` に JSON で保存される。
- 成果物は `G474_Orion_main.elf` / `.map` / `.list`。リンカの RWX 警告は現行のリンカ設定によるもの。

STM32CubeIDE の make を直接呼ぶ場合（Release も同様に `cd Release`）:

```powershell
cd Debug
& "C:\ST\STM32CubeIDE_1.17.0\STM32CubeIDE\plugins\com.st.stm32cube.ide.mcu.externaltools.make.win32_2.2.0.202409170845\tools\bin\make.exe" -B -j4 all
```

ST-Link の接続確認:

```powershell
& "C:\ST\STM32CubeCLT_1.21.0\STM32CubeProgrammer\bin\STM32_Programmer_CLI.exe" -l stlink
& "C:\ST\STM32CubeCLT_1.21.0\STM32CubeProgrammer\bin\STM32_Programmer_CLI.exe" -c port=SWD mode=UR -rst
```

`mode=UR` は Under Reset で接続する。起動直後に SWD が不安定なときに使う。

実機評価では、起動シーケンス、校正シーケンス、保護動作、通信周期を必ず回帰確認する。

## デバッグ UART（LPUART1、2 Mbps）

起動ログ、状態表示、表示ページの切り替え、ゲイン調整に使う。表示ページの名前と周期は `Core/Src/main.c` の `print_page_config[]` にまとめている。

| キー | 動作 |
|---|---|
| `0`〜`9` | 対応するページを直接選ぶ |
| Enter | 次のページ |
| Delete | `AI_CMD` ページ |
| `g` | `DRIVE_LOG` ページ（[診断機能](diagnostics.md#直進加速診断ログdrive_log)） |
| `t` | CM4 への通常テレメトリ送信を一時停止・再開する（[CM4 との UART](uart_cm4.md#テレメトリ送信)） |
| `q` / `a` | オムニ角度制御の kp を 1.1 倍 / 1/1.1 倍 |
| `w` / `s` | 同 kd を 1.1 倍 / 1/1.1 倍 |
| その他 | 前のページ |
