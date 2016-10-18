#レポート課題2-2

氏名: 阿部修也  

##課題内容
複数スイッチに対応したラーニングスイッチ (multi_learning_switch.rb) の動作を説明しなさい。

##課題解答
###コード解説
本説では、課題として与えられたmulti_learning_switch.rbのソースコードについて説明する。

####timer_event
Tremaのタイマー機能を用いて後述するage_fdbsメソッドを定期実行している。間隔は5秒。

####startハンドラ
FDB（フローデータベース）を初期化している。また、infoレベルでプログラムがスタートしたことを出力する。

####switch_readyハンドラ
switchのdatapath_idを引数に取り、連想配列FDBにdatapath_idをキーとして新たなインスタンス変数を作成、登録している。

####packet_inハンドラ
packet_inメッセージに含まれる宛先MACアドレスが、予約済み（ネットワークのプロトコルによる予約）のものであれば処理を終了してパケットを破棄する。

予約済みでない場合、packet_inメッセージを送信したスイッチのdatapath_idで連想配列FDBからインスタンスを呼び出し、fdb.rb内で定義されたlearnメソッドを実行する。
learnメソッドは、
送信元MACアドレスとスイッチにパケットが入ってきたポート番号を取得し、
これを対応付けてエントリを追加（学習）するメソッドである。

その後、flow_mod_and_packet_outメソッドを呼び出し、FlowModやPacket-Outメッセージの生成、送信を行っている。

####age_fdbs
本メソッドは5秒ごとに定期実行され、寿命を超えたFDBエントリを削除するfdb.rbのageメソッドを呼び出す（デフォルトでは、寿命は300）。

####flow_mod_and_packet_out
FDBからスイッチのdatapath_idと送信先MACアドレスをもとに送信ポート番号を取得し、
これを引数としてFlowMod、Packet-Outを行うメソッドをそれぞれ呼び出す。
ただし送信ポート番号がFDBから得られない場合にはFlowModは送信されず、パケットはフラッディングされる。

####flow_mod
引数としてPacket-Inメッセージと送信先ポート番号を受け取り、FlowModメッセージとしてコントローラに送信して学習させる。

####packet_out
引数としてPacket-Inメッセージと送信先ポート番号を受け取り,
Packet-Outメッセージを送信する。




