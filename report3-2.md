#レポート課題3-2

氏名: 阿部修也  

##課題内容
OpenFlow1.3 版スイッチ（learning_switch13.rb）の動作を説明しなさい．
スイッチ動作の各ステップについて，trema dump_flows の出力 (マルチプルテーブルの内容) を混じえながら動作を説明すること．

##課題解答
OpenFlow1.3 版スイッチ（learning_switch13.rb）では，マルチプルテーブルが実装されており，
初期状態から，パケットを流したあとのマルチプルテーブルの状態を比べることでその動作を確認することができる．

動作確認のためには，以下の各状態におけるマルチプルテーブルのエントリを確認すればよい．
ただし，"未知"とはコントローラが情報をフローデータベース（FDB）に持たないことを示し，
"既知"とはコントローラが情報をFDBに持つことを示している．

0. 初期状態
1. 未知の送信元から未知の宛先にパケットを送信
2. 未知の送信元から既知の宛先にパケットを送信
3. 既知の宛先から既知の宛先にパケットを送信

以下は，上記4状態を確認するために行った具体的な動作確認である．

###0. 初期状態
初期状態におけるスイッチのマルチプルテーブルは以下の通りである．
テーブルは2つに分かれており，table0において，宛先macアドレスによってフィルタリングを行い（当てはまるものはドロップ），
残りのパケットはtable1によって判別している．
table1においては，宛先がブロードキャストアドレスであればフラッディングを行い，
そうでなければコントローラにPacket-Inを送るという2つのエントリが格納されている．
OpenFlow1.3では，未知のパケットがきたときにデフォルトでPacket-Inが起こるわけではなく，
フローエントリとして明示的にPacket-Inを記述する必要がある．
```
cookie=0x0, duration=21.805s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=21.767s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=21.764s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
cookie=0x0, duration=21.767s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=21.766s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
```

###1. host1からhost2へパケットを送信
（ブロードキャストアドレス宛ではない）未知の宛先へとパケットが送信されることになるため，
Packet-Inがマルチプルテーブルの内容に従って実行される．

このときのマルチプルテーブルは以下のようになっている．
n_packetsの欄を見ることで，table0において "goto_table:1" が実行され，
table1において "CONTROLLER:65535" が実行されたことがわかる．
後者はいわゆるPacket-Inである．
しかし，宛先はコントローラにとって未知であるため，
コントローラはFD）に送信元（host1）のMACアドレスと入力ポートの情報を記録し，フラッディングを行う．
（フラッディングが行われていることはhost2の統計情報から確認できる）

このとき，フラッディングに関するテーブルのn_packetsの値は0であるが，
このエントリはコントローラ以外からのパケットの宛先がブロードキャストアドレスの場合に実行されるためである（今回はコントローラがフラッディングを起こしたためフローエントリとのマッチングは行われない）．
```
[trema dump_flows lsw]
cookie=0x0, duration=90.403s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=90.365s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=90.362s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
cookie=0x0, duration=90.365s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=90.364s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535

[trema show_stats host2]
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
```

###2. host2からhost1へパケットを送信
1. と同じく，フローエントリに従ってPacket-Inが起こり，コントローラにとって未知の送信元（host2）のMACアドレスとポートの対応がFDBに登録される．
これにより，送信元及び宛先はいずれもFDBに登録されていることになるため，
送信元と宛先の情報をもとに新たなフローエントリをマルチプルテーブルにFlowModメッセージで追加し，ユニキャストでPacket-Outを実行する．
これにより，以降にport2からスイッチに入力されたhost2（0f:33:ee:6a:af:11）からhost1（23:5e:72:ad:67）へのパケットは，
port1から出力される．
```
cookie=0x0, duration=117.809s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=117.771s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=117.768s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
cookie=0x0, duration=117.771s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=3.698s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=0f:33:ee:6a:af:11,dl_dst=23:5e:72:ad:67:bb actions=output:1
cookie=0x0, duration=117.77s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535

※追加されたエントリ
cookie=0x0, duration=3.698s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=0f:33:ee:6a:af:11,dl_dst=23:5e:72:ad:67:bb actions=output:1
```

###3. host1からhost2へパケットを送信
フローエントリに従ってPacket-Inが発生する．
コントローラにとって送信元及び宛先は既知であるので，
送信元と宛先の情報をもとに新たなフローエントリをマルチプルテーブルにFlowModメッセージで追加し，ユニキャストでPacket-Outを実行する．
これにより，以降にport1からスイッチに入力されたhost1（23:5e:72:ad:67）からhost2（0f:33:ee:6a:af:11）へのパケットは，
port2から出力される．

```
cookie=0x0, duration=149.213s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=149.175s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=149.172s, table=0, n_packets=3, n_bytes=126, priority=1 actions=goto_table:1
cookie=0x0, duration=149.175s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=35.102s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=0f:33:ee:6a:af:11,dl_dst=23:5e:72:ad:67:bb actions=output:1
cookie=0x0, duration=3.016s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=1,dl_src=23:5e:72:ad:67:bb,dl_dst=0f:33:ee:6a:af:11 actions=output:2
cookie=0x0, duration=149.174s, table=1, n_packets=3, n_bytes=126, priority=1 actions=CONTROLLER:65535
```
