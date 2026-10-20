# Orion：MainとCANノードの通信

Mainと左右BLDC・Sub・電源基板の通信の正本。対象はOrionの現行Main送受信実装であり、Atlasの対応を示さない。

- フレーム形式: Classic CAN
- ID: Standard ID
- Bit rate: 約 1 Mbps

## バスの使い分け

- FDCAN1: 右側モータ系、ドリブラー、電源 / キッカー系を送信対象に含む。
- FDCAN2: 左側モータ系、電源 / キッカー系を送信対象に含む。
- Mainは両バスで下表の受信IDを同じ形式として解釈する。Sub・センサの物理的な結線先は未確認である。

floatはlittle-endian IEEE 754 binary32である。以下のID表は用途の一覧で、各基板の全パラメータ定義と物理配線の照合は含まない。

## 送信 CAN ID

| ID | 送信先 / 指令 | バス | データ |
| --- | --- | --- | --- |
| 0x010 | 電源制御 / パラメータ | FDCAN1, FDCAN2 | byte0: param id、byte1 以降: 値 |
| 0x100 | Front Right モータ | FDCAN1 | float duty |
| 0x101 | Back Right モータ | FDCAN1 | float duty |
| 0x102 | Back Left モータ | FDCAN2 | float duty |
| 0x103 | Front Left モータ | FDCAN2 | float duty |
| 0x104 | ドリブラーモータ | FDCAN1 | float duty |
| 0x110 | キッカー | FDCAN1, FDCAN2 | 充電、電圧、ストレート / チップ、キック強度 |
| 0x300 - 0x304 | モータパラメータ | FDCAN1/FDCAN2 | param id + float |
| 0x310 | モータキャリブレーション | FDCAN1 or FDCAN2 | dummy 8 byte |
| 0x000 | エラー通知 | FDCAN1, FDCAN2 | 8 byte |
| 0x001 | 電源基板リセット | FDCAN1, FDCAN2 | `0x5A, 0xA5` |

## 受信 CAN ID

| ID | 内容 |
| --- | --- |
| 0x000 - 0x001 | エラー ID / info / value |
| 0x200 - 0x204 | モータ・ドリブラーの rps と角度 |
| 0x210 - 0x216 | 各基板の電圧、電源バッテリ、キャパシタ電圧 |
| 0x220 - 0x223 | モータ温度、ドライバ温度 |
| 0x224 | 電源系 FET 温度、コイル温度 |
| 0x230 - 0x234 | 各系統の電流 |
| 0x240 | ボール検出 |
| 0x241 | マウスセンサ raw X/Y と quality |
| 0x500 - 0x503 | モータパラメータ rps |


## 更新・照会

更新用CANはデータID `0x480`～`0x4FF`、コマンドID `0x610`、応答ID `0x650 + node ID`を使う。各データフレームのimageデータは7バイトとする。chunk上限と再送条件は[MCU更新](firmware_update.md#can-ノードの更新main-ゲートウェイ)を参照する。FWバージョン照会は `0x611`。更新対象・ノード番号と給電順序は[MCU更新](firmware_update.md)を参照する。

物理配線と回路図は未照合であり、バスの割当はMainの送受信実装から確認した論理構成である。FIFO実装とピン・割り込み設定は[Main内部のハードウェア実装](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/board_spec.md)を参照する。
