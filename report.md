#レポート課題2-2

氏名: 阿部修也  

##課題内容
複数スイッチに対応したラーニングスイッチ (multi_learning_switch.rb) の動作を説明しなさい。

##課題解答
###コード解説
本説では、課題として与えられたmulti_learning_switch.rbのソースコードについて説明する。

####timer_event
```ruby
  timer_event :age_fdbs, interval: 5.sec
```
Tremaのタイマー機能を用いて後述するage_fdbsメソッドを定期実行している。間隔は5秒。

####startハンドラ
```ruby
  def start(_argv)
    @fdbs = {}
    logger.info 'MultiLearningSwitch started.'
  end
```
FDB（フローデータベース）を初期化している。また、infoレベルでプログラムがスタートしたことを出力する。

####switch_readyハンドラ
```ruby
  def switch_ready(datapath_id)
    @fdbs[datapath_id] = FDB.new
  end
```
switchのdatapath_idを引数に取り、連想配列FDBにdatapath_idをキーとして新たなインスタンス変数を作成、登録している。

####packet_inハンドラ
```ruby
  def packet_in(datapath_id, message)
    return if message.destination_mac.reserved?
    @fdbs.fetch(datapath_id).learn(message.source_mac, message.in_port)
    flow_mod_and_packet_out message
  end
```
packet_inメッセージに含まれる宛先MACアドレスが、予約済み（ネットワークのプロトコルによる予約）のものであれば処理を終了してパケットを破棄する。

予約済みでない場合、packet_inメッセージを送信したスイッチのdatapath_idで連想配列FDBからインスタンスを呼び出し、fdb.rb内で定義されたlearnメソッドを実行する。
learnメソッドは、
送信元MACアドレスとスイッチにパケットが入ってきたポート番号を取得し、
これを対応付けてエントリを追加（学習）するメソッドである。

その後、flow_mod_and_packet_outメソッドを呼び出し、FlowModやPacket-Outメッセージの生成、送信を行っている。

####age_fdbs
```ruby
  def age_fdbs
    @fdbs.each_value(&:age)
  end
```
本メソッドは5秒ごとに定期実行され、寿命を超えたFDBエントリを削除するfdb.rbのageメソッドを呼び出す（デフォルトでは、寿命は300）。

####flow_mod_and_packet_out
```ruby
  def flow_mod_and_packet_out(message)
    port_no = @fdbs.fetch(message.dpid).lookup(message.destination_mac)
    flow_mod(message, port_no) if port_no
    packet_out(message, port_no || :flood)
  end
```
FDBからスイッチのdatapath_idと送信先MACアドレスをもとに送信ポート番号を取得し、
これを引数としてFlowMod、Packet-Outを行うメソッドをそれぞれ呼び出す。
ただし送信ポート番号がFDBから得られない場合にはFlowModは送信されず、パケットはフラッディングされる。

####flow_mod
```ruby
  def flow_mod(message, port_no)
    send_flow_mod_add(
      message.datapath_id,
      match: ExactMatch.new(message),
      actions: SendOutPort.new(port_no)
    )
  end
```
引数としてPacket-Inメッセージと送信先ポート番号を受け取り、FlowModメッセージとしてスイッチに送信して学習させる。


####packet_out
```ruby
  def packet_out(message, port_no)
    send_packet_out(
      message.datapath_id,
      packet_in: message,
      actions: SendOutPort.new(port_no)
    )
  end
```
引数としてPacket-Inメッセージと送信先ポート番号を受け取り,
Packet-Outメッセージを送信する。
このメソッドをpacket_inメソッドが呼び出すことにより，Packet-Inを引き起こしたパケットを転送することができる．


###動作解説
本節では，以下のようなモデル，シナリオを設定し，動作結果とともにmulti_learning_switch.rbの動作説明を行う．

####ネットワークモデル


