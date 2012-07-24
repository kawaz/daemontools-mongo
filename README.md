daemontools-mongo
=================

daemontools で MongoDB を起動するスクリプト。
/service/* のディレクトリ名から勝手にオプションを判断して起動してくれます。

# チュートリアル
## 1台のサーバで全部入りの環境構築してみる例
↓こんな構成を作ってみる。

    mongos (27017)
      configsvr
        db.example.jp:27021
        db.example.jp:27022
        db.example.jp:27023
      shardsvr
        rs1 (レプリカセット)
          db.example.jp:27030 (arbiter)
          db.example.jp:27031
          db.example.jp:27032
        rs2 (レプリカセット)
          db.example.jp:27040 (arbiter)
          db.example.jp:27041
          db.example.jp:27042

### 作業手順 ***

    # 必要なプロセス数分 clone する
    cd /service
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-mongos
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-config-27021
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-config-27022
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-config-27023
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-replrs1-arbiter-27030
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-replrs1-shard-27031
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-replrs1-shard-27032
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-replrs2-arbiter-27040
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-replrs2-shard-27041
    git clone https://github.com/kawaz/daemontools-mongo.git mongo-replrs2-shard-27042

    # mongos の設定だけ追加で作成する
    echo "db.example.jp:27021,db.example.jp:27022,db.example.jp:27023" > mongo-mongos/configdb

    # rs1の設定
    mongo localhost:27031 # レプリカの1つに繋ぐ。こいつがこのレプリカセットの最初のPRIMARYサーバになる。
    > rs.initiate() //時間掛かる
    rs1:SECONDARY> rs.status() // <-初期化が終わるとプロンプトが変わる
    rs1:PRIMARY> rs.status() // <-初期化が終わるとプロンプトが変わる
    rs1:PRIMARY> rs.add("db.example.jp:27032") // <-add等の作業はPRIMARYで行う
    rs1:PRIMARY> rs.addArb("db.example.jp:27030")

    # rs2 の設定
    mongo localhost:27041 # レプリカの1つに繋ぐ。こいつがこのレプリカセットの最初のPRIMARYサーバになる。
    > rs.initiate() //時間掛かる
    rs2:PRIMARY> rs.status() // <-初期化が終わるとプロンプトが変わる
    rs2:PRIMARY> rs.add("db.example.jp:27042")
    rs2:PRIMARY> rs.addArb("db.example.jp:27040")

    # シャーディングの設定
    mongo admin # mongo localhost:27017/admin と同じ
    mongos> db.runCommand({listshards:1})
    mongos> db.runCommand({addshard:"rs1/db.example.jp:27031,db.example.jp:27032"}) // <-レプリカセットを使う場合はこんな記述をする(適当に書けばエラーメッセージで教えてくれる)
    mongos> db.runCommand({addshard:"rs2/db.example.jp:27041"}) // <-実はレプリカのホストを全部指定しなくても芋蔓式に他のレプリカサーバを見つけてくれる

### 動作確認 ###
あとは`mongo test`とかで`mongos`に繋いで使えばOK。
- `db.hoge.insert{d:new Date, r:Math.random()}` とかすると対象シャードの前レプリカの data ディレクトリに test.db ができることが確認できる。
- レプリカの1つを落とても問題なく動作するし、落としたレプリカのdataディレクトリを削除してから再起動すれば勝手にレプリカセットに参加してデータの同期も行われることが確認できる。
- レプリカのPRIMARYサーバを落とすとPRIMARYサーバが移動することがログで確認できるし、mongosを経由していれば障害には気づかない。
- configサーバを順番に落としても mongos 経由なら問題なく利用できる。

### メモ ###
- MongoDBのレプリケーションではPRIMARYサーバ以外でDB操作が出来ない。単純にレプリカセットにアクセスする場合はどれがPRIMARYサーバかを判断する責任はアプリケーションやドライバ側の仕事という設計思想らしい。
なのレプリケーションをする場合はシャードが1つでもいいので、mongos+config を間に挟んで使うのが普通。mongos経由ならアプリ側は何も考えずにすむ。
- レプリカセットは2台構成とかでも良いがARBITERも立てておくとPRIMARYサーバの選択がコンフリクトすることを避けられるので立てておくと良い。
