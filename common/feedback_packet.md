# Main共通フィードバックパケット

MainからCM4、CM4からcrane・PCの生データデバッグツールへ渡す128バイトの契約を定義する。Orion向けのCM4送受信とPCデコーダがこの配置を使う。4WSは同じ配置を採用する設計で、MainのSPI通信とCM4からの配信は未実装である。

## 送信元と経路

OrionMain（STM32G474）のUARTからCM4へ送る。通信設定は[基板の対外仕様](../stm/orion_board_spec.md#cm4-との通信)、ネットワーク宛先は[システム構成](system_overview.md)を参照する。

CM4は検証済みフレームを生データmulticastとCM4内の位置制御用loopbackへ渡す。UARTの読み手とソケット構成は[CM4ブリッジ実装](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/bridge.md)に記載する。

## パケット仕様

パケット長は 128 バイト固定です。

### ヘッダ

- `0`: 同期バイト `0xAB`
- `1`: 同期バイト `0xEA`
- `2`: CRC-8/ATM（byte 3..127 を対象）
- `3`: `check_counter`（Mainが受けた指令の `check_counter` をそのまま返す）

CRC-8/ATM のパラメータは多項式 `0x07`、初期値 `0x00`、入力・出力とも反転なし、最終 XOR `0x00` です。
計算範囲は **byte 3..127 の125バイト**で、同期バイトとbyte 2は含めません。
標準検査値は `"123456789" → 0xF4` です。送信側は全ペイロードの確定後にbyte 2を書きます。
CM4は長さ・同期バイト・CRCを確認し、不正なパケットを位置制御にも再配信にも渡しません。

Main→CM4ではOrionMainと4WS Mainのどちらも同期値`0xAB 0xEA`を使います。CM4からPCへUDP multicastで送る際、4WSに限り転送コピーのbyte 1を`0xEB`に変更します。OrionのUDPコピーは`0xAB 0xEA`のままです。PCのデバッグツールはこの値で機体を判別し、同じペイロードデコーダを使います。同期バイトはCRC対象外なのでbyte 1の変更後もCRC値は有効です。4WSのUDP配信は未実装です。

byte 3 は指令の `check_counter` の反射です。mode 4 の位置制御経路では **CM4 が `check_counter` を採番する**
ので、ここを見れば「CM4 が出した指令がどこまで OrionMain に届いたか」が分かります
（詳細は [制御パケット](control_packet.md#check_counter) を参照）。
**これは実機だけの性質です。** シミュレータは自走カウンタを送ります
（[シミュレータとの接続](#シミュレータとの接続)を参照）。

### ペイロード
ヘッダの後、byte 4から下表の記述順に隙間なく配置します。各項目は表のバイト位置で扱います。

#### コア機能

| バイト | 項目 | 形式・意味 |
| --- | --- | --- |
| `4` | `tx_cycle_count` | 送信ごとに1増える`uint8_t` |
| `5..6` | `current_error_id` | little-endian `uint16_t` |
| `7..8` | `current_error_info` | little-endian `uint16_t` |
| `9..12` | `current_error_value` | little-endian IEEE754 float |

#### メイン基板

| バイト | 項目 | 形式 |
| --- | --- | --- |
| `13..16` | `imu_yaw_deg` | little-endian IEEE754 float |
| `17..18` | `ball_detection[2]` | 各1バイト |
| `19` | `ball_detection_extra` | 1バイト |
| `20..23` | `diff_angle_deg` | little-endian IEEE754 float |

#### 電源基板

| バイト | 項目 | 形式 |
| --- | --- | --- |
| `24..27` | `battery_voltage` | little-endian IEEE754 float |
| `28` | `kick_state_div10` | 1バイト |
| `29` | `temp_fet` | 1バイト |
| `30..31` | `temp_coil[2]` | 各1バイト |
| `32..35` | `capacitor_boost_voltage` | little-endian IEEE754 float |
| `36..39` | `mouse_odom_x` | little-endian IEEE754 float |
| `40..43` | `mouse_odom_y` | little-endian IEEE754 float |
| `44..47` | `mouse_global_vel_x` | little-endian IEEE754 float |
| `48..51` | `mouse_global_vel_y` | little-endian IEEE754 float |
| `52..55` | `mouse_quality` | little-endian IEEE754 float |

#### モーター基板

| バイト | 項目 | 形式 |
| --- | --- | --- |
| `56..59` | `motor_current_x10[4]` | 各1バイト、電流の10倍 |
| `60..63` | `temp_motor[4]` | 各1バイト、駆動モーター温度 [°C] |
| `64..67` | `output_vel_x` | little-endian IEEE754 float |
| `68..71` | `output_vel_y` | little-endian IEEE754 float |
| `72..75` | `motor_feedback_0` | little-endian IEEE754 float、回転数 [rps] |
| `76..79` | `motor_feedback_1` | little-endian IEEE754 float、回転数 [rps] |
| `80..83` | `motor_feedback_2` | little-endian IEEE754 float、回転数 [rps] |
| `84..87` | `motor_feedback_3` | little-endian IEEE754 float、回転数 [rps] |
| `88..91` | `local_odom_speed_mvf_x` | little-endian IEEE754 float |
| `92..95` | `local_odom_speed_mvf_y` | little-endian IEEE754 float |
| `96..99` | `local_odom_speed_mvf_w` | little-endian IEEE754 float |
| `100..107` | `steering_angle[4]` | 各2バイト、ステア現在角度 [rad] |
| `108..111` | `temp_steering_motor[4]` | 各1バイト、ステアモーター温度 [°C] |

`steering_angle[0..3]`は輪番号順に各2バイトを上位バイトから格納します。
符号化範囲は±10π radで、[mode 5の設計案](../proposals/control_modes.md)の操舵角と同じ2バイト表現です。
`temp_steering_motor[0..3]`の輪番号も操舵角と対応させます。
OrionMainはステアを搭載しないため、各角度を0 radの符号化値`0x7FFF`、各ステアモーター温度を0として送ります。

#### 制御

| バイト | 項目 | 形式 |
| --- | --- | --- |
| `112..115` | `vision_based_position_x` | little-endian IEEE754 float |
| `116..119` | `vision_based_position_y` | little-endian IEEE754 float |
| `120..123` | `global_odom_speed_x` | little-endian IEEE754 float |
| `124..127` | `global_odom_speed_y` | little-endian IEEE754 float |

byte 4..127はすべて上表のフィールドに割り当てます。
FW更新ゲートウェイ動作中は通常のfeedback送信を停止し、独立した16バイトの`FWRP`応答フレームを送ります。

CM4のローカルカメラ情報は、このfeedbackパケットとは別の経路で受けます。形式は[CM4カメラ実装](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/camera.md)を参照してください。


## 受信側の契約

- CM4の位置制御と生データ再配信は、長さ・同期・CRCが正しいフレームだけを採用する。位置制御は位置のX・Yが有限であることも確認する。不正フレームで受信鮮度を更新しない。
- PCデバッグツールはOrion/4WSのUDP同期値を判別して共通配置を復号する。CRC不一致は診断結果として表示し、CM4の制御経路のように一律破棄する扱いではない。
- 操舵角の各2バイトは上位バイト先行で、`0xFFFF`を無効値として拒否する。他の整数・floatのlittle-endianと区別する。
- ボールセンサは`ball_detection[2]`と`ball_detection_extra`である。送信サイクルカウンタをセンサ値として読まない。
- 稼働中のFWゲートウェイは通常のfeedbackを停止する。独立した`FWRP`応答はこの128バイト配置として復号しない。

## シミュレータとの接続

simulator-cliもこの配置とCRCを生成する必要がある。CRCを定数で代用した出力はCM4で受け付けない。シミュレータのbyte 3は自走カウンタであり、指令到達の反射として使わない。位置・速度の更新源の違いと接続条件は[シミュレーション連携](../pc/simulation.md)を参照する。

## 実装と運用の参照先

- CM4：`cm4/bridge/robot_feedback_packet.h`、`forward_robot_feedback.cpp`、`forward_ai_cmd_v2.cpp`。
- PC：`host/lib/feedback/packet.py`、`receiver.py`。
- [受信・表示ツール](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/host_tools.md)
- [非走行での確認手順](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/bridge.md)
- [CM4共通状態パケット案（未実装）](../proposals/cm4_status_packet.md)
