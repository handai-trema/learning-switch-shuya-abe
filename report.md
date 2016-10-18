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
今回利用するネットワークは，
サンプルとして課題リポジトリ内に入っていたtrema.multi.confによって作られる．
以下にtrema.multi.confを示す．

```
vswitch('lsw1') { datapath_id 0x1 }
vswitch('lsw2') { datapath_id 0x2 }
vswitch('lsw3') { datapath_id 0x3 }
vswitch('lsw4') { datapath_id 0x4 }

vhost('host1')
vhost('host2')
vhost('host3')
vhost('host4')

link 'lsw1', 'host1'
link 'lsw2', 'host2'
link 'lsw3', 'host3'
link 'lsw4', 'host4'
link 'lsw1', 'lsw2'
link 'lsw2', 'lsw3'
link 'lsw3', 'lsw4'
```

これによって構築されるネットワークのモデル図は以下のようになる．
図の通り，スイッチが4つ，ホストが4つあり，スイッチとホストが1対1で接続される．
また，スイッチ同士はlsw1-lsw2-lsw3-lsw4というように接続される．

![](img/multi-switch-model.png)

####シナリオ
今回の動作説明では，以下のようにパケットの送受信が行われた際のフローテーブルや統計情報を確認する．

```
1. host1からhost4へパケットを送信する
2. host4からhost1へパケットを送信する
3. host1からhost4へパケットを送信する
```

1. の前後ではフローテーブルは書き込まれず，
2. の終了後にはhost4からhost1へのパケットについて，
各スイッチのフローテーブルに情報が追加される．
最後に，3. の終了後にはhost1からhost4に対するパケットについて
各スイッチのフローテーブルに情報が追加される．

####実際の動作
まず，host1からhost4にパケットを送る．これにより，FDBにlsw1について，host1とlswのポートとの対応が登録される．
```
trema send_packets --source host1 --dest host4
```

統計情報を確認すると，確かにhost1からhost4にパケットを送信できていることがわかる．
```
trema show_stats host1
---
Packets sent:
  192.168.0.1 -> 192.168.0.4 = 1 packet
```

同じく，host4はhost1からのパケットを受け取っている．
```
trema show_stats host4
---
Packets received:
  192.168.0.1 -> 192.168.0.4 = 1 packet
```

ここで，host4からhost1にパケットを送信する．これにより，FDBにはhost4とポート番号の対応が登録される．
また，宛先であるhost1に関する情報がすでに登録されているため，flowmodメッセージによってhost4からhost1へ送られるパケットに関する情報がフローテーブルに追加される．
```
trema send_packets --source host4 --dest host1
```

統計情報を確認すると，確かにhost4からhost1にパケットを送信できていることがわかる．
```
trema show_stats host1
---
Packets sent:
  192.168.0.1 -> 192.168.0.4 = 1 packet
Packets received:
  192.168.0.4 -> 192.168.0.1 = 1 packet
```

同じく，host4の統計情報は以下の通り．
```
trema show_stats host4
---
Packets sent:
  192.168.0.4 -> 192.168.0.1 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.4 = 1 packet
```

このとき，lsw1のフローテーブルの中身は以下のようになっている．
確かに，host4からhost1へのパケットに関するエントリのみが保存されている．
```
trema dump_flows lsw1
---
cookie=0x0, duration=29.616s, table=0, n_packets=0, n_bytes=0, idle_age=29, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=97:f6:8a:65:ed:a9,dl_dst=37:84:12:a7:26:85,nw_src=192.168.0.4,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
```

また，経路であるlsw2やlsw3も同じようにエントリが追加されている．
```
[lsw2]
cookie=0x0, duration=41.181s, table=0, n_packets=0, n_bytes=0, idle_age=41, priority=65535,udp,in_port=3,vlan_tci=0x0000,dl_src=97:f6:8a:65:ed:a9,dl_dst=37:84:12:a7:26:85,nw_src=192.168.0.4,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
---
[lsw3]
cookie=0x0, duration=51.877s, table=0, n_packets=0, n_bytes=0, idle_age=51, priority=65535,udp,in_port=3,vlan_tci=0x0000,dl_src=97:f6:8a:65:ed:a9,dl_dst=37:84:12:a7:26:85,nw_src=192.168.0.4,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
```

ここで，host1からhost4へのパケットを送信する．
これにより，lsw1のフローテーブルにはhost1からhost4に対するパケットのエントリが保存される．
```
trema send_packets --source host1 --dest host4
```

統計情報を確認すると，確かにhost1からhost4への送信回数が増えている．
```
trema show_stats host1
---
Packets sent:
  192.168.0.1 -> 192.168.0.4 = 2 packet
Packets received:
  192.168.0.4 -> 192.168.0.1 = 1 packet
```

host4も正しくパケットを受信できていることがわかる．
```
trema show_stats host4
---
Packets sent:
  192.168.0.4 -> 192.168.0.1 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.4 = 2 packet
```

ここで，lsw1のフローテーブルを確認すると以下の通り．確かに，host1からhost4へのパケットのエントリが追加されている．
```
trema dump_flows lsw1
---
cookie=0x0, duration=127.136s, table=0, n_packets=0, n_bytes=0, idle_age=127, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=97:f6:8a:65:ed:a9,dl_dst=37:84:12:a7:26:85,nw_src=192.168.0.4,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=28.013s, table=0, n_packets=0, n_bytes=0, idle_age=28, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=37:84:12:a7:26:85,dl_dst=97:f6:8a:65:ed:a9,nw_src=192.168.0.1,nw_dst=192.168.0.4,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
```

