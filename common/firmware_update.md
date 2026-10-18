# MCU ファームウェア更新

CM4 から、UART でつながった Main（STM32G474）を更新する。CAN 上の Sub、左右 BLDC、Power（STM32F303）は、Main をゲートウェイにして更新する。Main は A/B の 2 スロットを持ち、CAN ノードはアプリ領域が 1 つだけである。転送したデータと書込み結果は CRC32C で確認する。

## 対象とノード

| 対象 | CM4 側ツール | 経路 | CAN バス / node ID |
|---|---|---|---|
| Main | `cm4/firmware/main_ab_updater.py` | UART（Main の bootloader と OFW1 で通信） | — |
| Sub | `cm4/firmware/sub_can_updater_v2.py` | UART（OFW2）→ Main ゲートウェイ → CAN | CAN1 / 4 |
| 右 BLDC | 同上 | 同上 | CAN1 / 16 |
| 左 BLDC | 同上 | 同上 | CAN2 / 17 |
| Power | 同上 | 同上 | 100（ツールの指定は CAN1。ゲートウェイは安全停止のときに応答があったバスへ切り替える） |
| バージョン確認 | `cm4/firmware/fw_version_reader.py` | UART（FWVR） | 全ノード |

- `sub_can_updater_v2.py` は 1 回の更新で CAN1 と CAN2 のノードを 1 つずつ選べる（`--node-can1` 既定 4、`--node-can2` 既定 `0xFF` = 使わない）。左右の BLDC には同じイメージを両バスへ並列に送れる。
- ノード番号とバスは、実機の配線に合わせて指定する。

## 更新順序

1. 対象、イメージ、給電状態を確認する。UART の取り合いを避けるため `control_server.service` を止める。
2. Sub と BLDC のイメージを更新し、確定する。
3. Power を安全停止してから更新する。Power は Sub と BLDC への給電を制御しているので、この順序を守る。
4. Main の稼働していないスロットを更新する。Main は CAN 更新のゲートウェイなので最後にする。
5. `fw_version_reader.py` で全対象の build ID と CRC32C を確認し、Main の稼働スロットが CONFIRMED であることを確認する。
6. `control_server.service` を戻し、制御 API と必要な周辺機能が起動していることを確認する。

`all_can_updater.py` は Power→Sub→BLDC の順にすべての対象を bootloader へ移してから転送する。Power を先に bootloader へ移すと Sub と BLDC への給電が切れるので、この給電構成では全ノードの一括更新に使わない。更新中の通信エラーや、再起動後の不一致は成功として扱わない。対象と状態を確認してから再開する。

## 実行例

引数の形式を示す。実行前に、対象の機体番号、バス、node ID、イメージを照らし合わせる。

```bash
python3 cm4/firmware/sub_can_updater_v2.py sub.bin --port /dev/serial0 --node-can1 4
python3 cm4/firmware/sub_can_updater_v2.py bldc.bin --port /dev/serial0 --node-can1 16 --node-can2 17
python3 cm4/firmware/sub_can_updater_v2.py power.bin --port /dev/serial0 --node-can1 100
python3 cm4/firmware/main_ab_updater.py --slot-a main_a.bin --slot-b main_b.bin --port /dev/serial0
python3 cm4/firmware/fw_version_reader.py --port /dev/serial0 \
  --main-a main_a.bin --main-b main_b.bin --sub sub.bin \
  --bldc-can1 bldc.bin --bldc-can2 bldc.bin --power power.bin
```

各コマンドの引数の詳細は `--help` で確認する。

## 仕組み

### 更新モードへの切り替え

CM4 は通常の 72 B UART フレーム（[制御パケット](control_packet.md#uart-フレームcm4--g47472-バイト)）の byte 1..4 に `FWUP` を入れ、byte 5 のコマンドで Main に指示する。

| byte 5 | 動作 |
|---|---|
| `1` | CAN 更新ゲートウェイを開始する |
| `2` | ゲートウェイ動作中に、指定した CAN フレームを CAN1 へ送る |
| `3` | ゲートウェイを終了し、Main を再起動する |
| `4` | Main bootloader の UART 更新へ移る（Main 自身の更新） |
| `5` | Main bootloader で PENDING のスロットを CONFIRMED にする |

`4` と `5` では、現在の稼働スロットと要求をバックアップレジスタ（`TAMP->BKP0R/BKP1R`）に書いて再起動し、bootloader が受け取る。

### Main の A/B 更新

`main_ab_updater.py` は、Main bootloader の OFW1 UART（12 バイトのヘッダ、chunk 896 バイト）で稼働していないスロットへ書き込む。書き終えたスロットは PENDING として起動し、CM4 が CONFIRM を送ると CONFIRMED になる。PENDING のスロットの起動が 3 回確定しなかった場合、bootloader はもう一方の CONFIRMED のスロットへ戻す。Flash 配置と起動判定は [ブートローダー](../stm/bootloader.md) を参照。

### CAN ノードの更新（Main ゲートウェイ）

Main のアプリ内にある `fw_update_gateway.c` が、CM4 から受けた OFW2 フレームを CAN へ展開する。

- UART フレーム：magic `OFW2`、version、type、sequence、length、ヘッダ CRC16、payload（最大 907 バイト）、CRC32C。同じ sequence を再び受けたら、前回の応答を返す。
- CAN：データは `0x480`〜`0x4FF`（chunk 896 バイトを 7 バイトずつ 128 フレームに分ける）、ノードへのコマンドは `0x610`、応答は `0x650 + node ID`。
- `ENTER` で CAN1 と CAN2 に別々の node ID を指定できる。使わないバスは `0xFF`。左右の BLDC には同じデータフレームを両方の FDCAN から並列に送り、応答はバスごとに確認する。status と確定 offset が両方で一致したときだけ成功とする。
- FDCAN の送信 FIFO に空きができるまで待ち、更新フレームは捨てない。CAN の応答がタイムアウトしたら HELLO で確定 offset を問い合わせ、書込み済みの chunk を二重に書かない。
- Power を更新するときだけ、8 CAN フレームごとに 1 ms 待つ。更新前に CAN ID `0x110` でキッカーを安全状態にし、Power の状態を 3 回続けて確認する。
- ゲートウェイの動作中は、TIM7 の制御周期、ブザー PWM、通常のテレメトリ送信を止める。

### バージョン確認（FWVR）

Main の各スロットのイメージ先頭 `+0x400` には、`FWVR` magic と Unix 秒の build ID が置かれる（CAN ノード側のファームウェアの配置は、この文書群では確認していない）。

1. CM4 は 72 B UART フレームの byte 1..4 に `FWVR` を入れて送る。
2. Main は自身の両スロットを直接読み、CAN ID `0x611` で Sub（CAN1 node 4）、右 BLDC（CAN1 node 16）、左 BLDC（CAN2 node 17）、Power（node 100、両バス）に照会する。
3. 応答のないノードにだけ 40 ms 間隔で再照会し、250 ms 後に 6 対象の build ID と image CRC32C を 60 バイトの応答にまとめて CM4 へ返す。

`fw_version_reader.py` は期待するバイナリを指定すると、実機の内容と比べた結果も表示する。Main のビルドスクリプトは、ビルドごとに Git hash と dirty 状態を `Script/Logs/Build/` に JSON で保存する（[STM 概要](../stm/overview.md#ビルドと書込み)）。
