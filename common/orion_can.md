# Orion：MainとCANノードの通信

Mainと左右BLDC・Sub・電源基板の通信の正本。対象はOrionの現行Main送受信実装であり、Atlasの対応を示さない。

- フレーム形式: Classic CAN
- ID: Standard ID
- Bit rate: 約 1 Mbps

## バスの使い分け

- FDCAN1: 右側モータ系、ドリブラー、電源 / キッカー系を送信対象に含む。
- FDCAN2: 左側モータ系、電源 / キッカー系を送信対象に含む。
- Mainは両バスで下表の受信IDを同じ形式として解釈する。Sub・センサの物理的な結線先は未確認である。

以下のID表は用途の一覧で、バイト配置は後述の[ペイロード](#ペイロードの配置)に定義する。各基板の全パラメーターの許容範囲と物理配線の照合は含まない。

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


## ペイロードの配置

以下はOrionMainの送信・受信処理と4WS_MainFWの互換処理で照合した配置である。整数はlittle-endian、floatはIEEE 754 binary32のlittle-endianとする。角度の機体補正、輪の配置、受信値の有効性判定はこのバイト配置とは別に扱う。

### 電源指令

| 指令 | ID | byte配置 |
|---|---|---|
| 電源ON/OFF | 0x010 | byte 0=0、byte 1=OFF 0 / ON 1、byte 2..7=0 |
| 保護パラメーター | 0x010 | byte 0=パラメーターID、byte 1..4=float値、byte 5..7=0 |
| 電源リセット | 0x001 | byte 0=0x5A、byte 1=0xA5、byte 2..7=0 |

これらの指令は8バイトで、両バスへ送る。保護パラメーターIDは1=最低電圧、2=最高電圧、3=最大電流、4=FET温度上限、5=コイル温度上限。機器の許容範囲・値間の整合、両バスで受信した指令の重複処理、再送時の扱いは相手側との確認が必要である。Main側の送信キューへの登録成功だけで機器への反映を確認したことにはならない。

### テレメトリー

| ID | byte配置と値 |
|---|---|
| 0x000–001 | byte 0..1=エラーID uint16、2..3=補足情報 uint16、4..7=値 float |
| 0x200–204 | byte 0..3=回転数 rps float、4..7=エンコーダー角度 rad float |
| 0x210–216 | byte 0..3=電圧 V float |
| 0x220–223 | byte 0..3=モーター温度 float、4..7=ドライバー温度 float |
| 0x224 | byte 0=FET温度、1..2=コイル2個の温度（各uint8） |
| 0x230–234 | byte 0..3=電流 float |
| 0x240 | byte 0..1=ボール検出2個（各uint8） |
| 0x241 | byte 0..1=マウスX int16、2..3=Y int16、4..5=品質 uint16 |
| 0x500–503 | byte 0..3=モーターパラメーター応答 float |

表にない末尾領域は、この照合では意味を定義していない。表の使用バイト数を各IDのDLCや送信周期の定義とみなさない。特に電源電圧のDLCは機器側との照合が必要で、4WS側の8バイト限定デコーダーを全機器の共通要件にしない。4WSの受信観測・破棄・非対応形式は[FWの対応状況](https://github.com/ibis-ssl/4WS_MainFW/blob/main/doc/peripherals.md#can受信)で管理する。

### 配置の照合範囲

OrionMainの`actuatorPower_ONOFF()`、`actuator_power_param()`、`resetPowerBoard()`、`parseCanCmd()`と、4WS_MainFWの`power_packet_*()`、`orion_telemetry_decode()`を参照して配置を照合した。OrionMainはコミット`500855ee2a2fa3e99715a6b71a97bf17d0c2f71c`、4WS_MainFWは`6f1d5f9ebae95e606414236afa25452d8f224ecb`時点。これは配置を確認した範囲であり、CANノード側FW・物理配線・全DLCの一致や4WS機体への適合を保証しない。

## 更新・照会

更新用CANはデータID `0x480`～`0x4FF`、コマンドID `0x610`、応答ID `0x650 + node ID`を使う。各データフレームのimageデータは7バイトとする。chunk上限と再送条件は[MCU更新](firmware_update.md#can-ノードの更新main-ゲートウェイ)を参照する。FWバージョン照会は `0x611`。更新対象・ノード番号と給電順序は[MCU更新](firmware_update.md)を参照する。

物理配線と回路図は未照合であり、バスの割当はMainの送受信実装から確認した論理構成である。FIFO実装とピン・割り込み設定は[Main内部のハードウェア実装](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/board_spec.md)を参照する。
