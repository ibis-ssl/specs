# Orion：CM4とのUART契約

OrionMainとCM4の対外仕様を扱う。物理設定は[基板の対外仕様](orion_board_spec.md#cm4-との通信)、指令は[制御パケット](../common/control_packet.md#cm4main-パケット72-バイト)、状態は[共通feedback](../common/feedback_packet.md)、通信断と停止は[停止判定](../common/safety_stop.md)を正本とする。

## 受信・応答

ヘッダ・長さ・チェックサムが正しい制御指令だけを採用し、CM4接続の鮮度を更新する。`FWVR` / `FWUP`は通常制御指令と区別して[更新・照会要求](../common/firmware_update.md)として扱う。FWゲートウェイ中はOFW2を受信し、通常のfeedbackを停止する。

## UART 占有率

72バイトの指令は8N1・1 Mbpsで720 µsを占有する。指令100 Hzで7.2%、500 Hzで36%となる。feedbackの送信周期・鮮度の扱いは[feedback契約](../common/feedback_packet.md#送信元と経路)を参照する。

## Main内部の参照先

FIFO受信・DMA送信・レジスタ競合防止・診断・デバッグ操作は[MainのUART実装](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/uart_cm4.md)で管理する。byte 64..70を旧カメラ情報として読むMainの残存処理と影響も同文書で追跡する。共通契約は同領域を未定義・ゼロ送信とし、旧経路の互換性を保証しない。
