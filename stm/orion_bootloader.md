# Orion：ブートローダー（Main 基板）

Orion（4輪オムニホイール）の Main 基板（STM32G474）は、User Flash の先頭に基板専用のアプリケーションブートローダー（`Bootloader/`）を置き、通常アプリを A/B の 2 スロットで持つ。CM4 からの UART 更新、PENDING と CONFIRM、起動に失敗したときのロールバックを担当する。以下はOrionの更新ツールとMainが共有するイメージ配置・起動状態・更新中の安全動作の契約である。更新手順の全体は [MCU ファームウェア更新](../common/firmware_update.md)、機種別の入口は [STM 概要](../README.md#orionの対外仕様) を参照。

STM32 の System Memory bootloader、BOOT0 による ROM boot、ハードウェアのバンクスワップ（`BFB2`）は使わない。

## Flash 配置

値は `Bootloader/Inc/boot_config.h` にまとめてあり、通常アプリのリンカ設定と metadata 生成スクリプトも同じ値を使う。

| 領域 | アドレス | サイズ | 用途 |
|---|---|---:|---|
| Bootloader | `0x08000000`〜`0x08007FFF` | 32 KB | 更新中の安全動作、起動判定、UART 更新 |
| Slot A | `0x08008000`〜`0x0803FFFF` | 224 KB | 通常アプリ |
| Slot B | `0x08040000`〜`0x08077FFF` | 224 KB | 通常アプリ |
| Metadata | `0x08078000`〜`0x0807FFFF` | 32 KB | Slot A metadata（先頭）、Slot B metadata（`+0x800`）、control record（`+0x1000`） |

- 更新イメージは書込み先スロットのアドレスにリンクする。
- アプリ先頭の `+0x400` には `.fw_version`（`FWVR` magic と build ID）を置く（[MCU ファームウェア更新](../common/firmware_update.md#バージョン確認fwvr)）。

### metadata と control record

- **metadata**（スロットごと）：magic `OFW1`、format version、record size、slot、image base、image size、image CRC32C、generation、state、record CRC32C。state は `EMPTY` / `RECEIVING` / `VERIFIED` / `PENDING` / `CONFIRMED` / `INVALID`。
- **control record**：magic、format version、generation、優先スロット（`preferred_slot`）、起動試行回数（`boot_attempts`）、record CRC32C。

## 起動と確定の契約

イメージとmetadataを検証してから有効なスロットを起動する。PENDINGを起動するたびに試行回数を記録し、3回確定できなかった場合は、もう一方の有効なCONFIRMEDスロットへ戻す。起動できるスロットがなければ通常制御を開始せず停止する。

検証・スロット選択・CPU状態の引継ぎ・安全IOの実装は[Mainブートローダー実装](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/bootloader.md)で管理する。

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
- 起動するスロットの検証中と、起動できるスロットがないときは [起動と確定の契約](#起動と確定の契約) に従って LED で状態を示す。

## 開発・導入手順

ビルド、初回導入、ST-Linkによる復旧・書込み、WRPの運用は[Main側の導入手順](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/bootloader.md)を参照する。CM4からの更新は[MCU更新](../common/firmware_update.md)の給電順序と更新契約に従う。
