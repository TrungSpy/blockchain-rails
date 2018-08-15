# はじめに
blockchainの仕組みを学んでいたときに読んだ[こちら](https://qiita.com/hidehiro98/items/841ece65d896aeaa8a2a)の記事がとてもわかりやすかったので、この記事を参考にしてRailsでblockchainを実装してみました。

# 作成する機能
今回は参照元の記事にある

- transactionの追加
- blockの追加(マイニング)
- blockchainの取得
- nodeの登録
- blockchainの更新(登録しているnodeの中で一番長いblockchainに更新)

を実装していきます。

# blockchain
models/以下にblockchain.rbを作ります。

```rb
class Blockchain
end
```

今回は通常のRailsアプリと違い、サーバーを起動している間はメモリにデータを持たせたままにして置くので、singletonモジュールを使ってシングルトン化して、リクエストをまたいでデータを持つようにしておきます。
また、スレッドが複数になると整合性がとれなくなるため、pumaのthread数は1にしておきます。
(このあたりはいまいち自信ないので間違っていたらご指摘お願いします。)

```rb
require 'singleton'

class Blockchain
  include Singleton
end
```

## transaction
transactionは取引の記録です。
今回は送信者、受信者、送信量を記録します。

```rb
def new_transaction(sender, recipient, amount)
    @current_transactions.append(
        {
            sender: sender,
            recipient: recipient,
            amount: amount,
        }
    )

    return last_block[:index] +1
end
```

transactionは内部でリストとして保持していて、次回block作成時にそれまで作られたtransactionのリストがblockに記録されます。

## block
blockは

- index(Block作成時に１ずつ増加する数値)
- timestamp
- transactionのリスト
- proof
- 直前のblockのhash値

を持ちます。

直前のblockのhash値を持つのは、以前のblockが改ざんされていないことを確認するためです。
リストを辿りながらhash値を検証することで、全てのblockが改ざんされていないことを確認することが出来ます。

proofはマイニングの成功判定に使うhash値を求めるために使用します、
具体的には直前のblockのproofと文字列連結したものにSHA256を適用した値を求めるために使います。

この値の先頭が「0000」で始まるときのみ正当なproofとして認められます。
「0000」以外の場合はproofをインクリメントして、再度SHA256を適用します。

```rb
        def valid_proof(last_proof, proof)
            guess = "#{last_proof}#{proof}"
            guess_hash = Digest::SHA256.hexdigest(guess)

            return guess_hash.first(4) == "0000"
        end
```

この条件に合致するproofが求められると、報酬を得ることができます。
報酬の取得は受信者を自分自身にしたtransactionをリストに加えることで実現します。
この報酬の取得が、マイニングを行う動機となります。

ちなみにblockを100個程作成したときのproofの平均は約65000でした。
判定条件の0の桁数を増やすことで、マイニングの難易度をあげることができます。

そして、これまでのtransactionのリストを持ったblockを作成します。

作成したblockはblockの配列の末尾に追加しておきます。

```rb
    def new_block(proof, previous_hash=nil)
        block = {
            index: @chain.size + 1,
            timestamp: Time.now.to_i,
            transactions: @current_transactions,
            proof: proof,
            previous_hash: previous_hash || self.class.hash(last_block),
        }

        @current_transactions = []
        @chain.append(block)
        return block
    end
```


なお、一番最初のblockはBlockchainクラスが最初に作られたときに作成します。

```rb
new_block(100,1)
```

# blockchainの正当性
blockchainは全てのサーバー間で共通である必要があります。
複数のサーバーでマイニングが成功した際には複数のblockchainが存在してしまうので、どのblockchainを正しいものとするかを決めるルールが必要です。
今回は最も長いチェインを持つものを正当なものとするルールにします。

その比較を行うため、別のサーバー(node)を追加できるようにします。

```rb
    def register_node(address)
        parsed_url = URI::parse(address)
        @nodes.add("#{parsed_url.host}:#{parsed_url.port}")
    end
```

次にチェインを比較するためのメソッドを作ります。

```rb
   def resolve_conflicts
        neighbours = @nodes
        new_chain = nil

        max_length = @chain.size
        @nodes.each do |node|
            io = OpenURI.open_uri("http://#{node}/blocks")
            response = JSON.parse(io.read).deep_symbolize_keys!
            status = io.status[0]

            if status == "200"
                length = response[:length]
                chain = response[:chain]

                if length > max_length && self.valid_chain(chain)
                    max_length = length
                    new_chain = chain
                end
            end
        end

        if new_chain
            @chain = new_chain
            return true
        end

        return false
    end
```

少し長いですが、やっていることは、

- 登録されたnodeからblockのチェイン(リスト)を取得
- そのチェインが正当なものかを検証(proofのhashとblockのhashの検証)
- 正しいものかつ、自分のチェインより長いものならそのチェインを採用

となります。

# Web APIの作成
ここまでの機能をAPI経由で行えるようにしていきます。

## Blockchainの取得
ルーティングはblockリストの取得ということで

```rb
get '/blocks', to: 'blocks#index'
```
としています。　

このAPIは単純に現在のブロックのリストを返却します。

```rb
class BlocksController < ApplicationController
    ...
    def index
        blockchain = Blockchain.instance
        chain = blockchain.chain
        response = {
            chain: chain,
            length: chain.size,
        }

        render json: response, status: :ok
    end
    ...
end

```

## transactionの追加
ルーティングは

```rb
post '/block/transactions', to: 'block/transactions#create'
```
としています。

現在のtransactionリストの最後にtransactionを追加します。

```rb
class Block::TransactionsController < ApplicationController
    def create
        sender = create_params[:sender]
        recipient = create_params[:recipient]
        amount = create_params[:amount]

        blockchain = Blockchain.instance
        index = blockchain.new_transaction(sender, recipient, amount)

        response = {'message': "Transaction will be added to Block #{index}"}
        render json: response, status: :created
    end

    def create_params
        params.permit(:sender, :recipient, :amount)
    end
end
```

## マイニング
マイニングは結果的にblockの作成を行うので、ルーティングは

```rb
post '/blocks', to: 'blocks#create'
```
としています。

まず、正当なproofを求め、見つかった後に自分へ報酬を与えるtransactionの追加とblockの作成を行っています。

```rb
class BlocksController < ApplicationController
    ...
    def create
        blockchain = Blockchain.instance

        last_block = blockchain.last_block
        last_proof = last_block[:proof]
        proof = blockchain.proof_of_work(last_proof)

        blockchain.new_transaction("0",blockchain.node_identifire, 1)

        block = blockchain.new_block(proof)

        response = {
            message: "new block",
            index: block[:index],
            transactions: block[:transactions],
            proof: block[:proof],
            previous_hash: block[:previous_hash],
        }

        render json: response, status: :ok
    end
    ...
end
```


## node(サーバー)の追加
nodeを追加します。ルーティングは

```rb
  post '/nodes', to: 'nodes#create'
```
としています。

nodeのリストはSet型なので重複して登録しても重複は排除されます。

```rb
class NodesController < ApplicationController
    def create
        nodes = create_params[:nodes]

        if nodes.nil?
            render json: {error: 'Please supply a valid list of nodes'}, status: :bad_request
            return
        end

        blockchain = Blockchain.instance

        nodes.each do |node|
            blockchain.register_node(node)
        end

        response = {
            message: "New nodes have been added",
            total_nodes: blockchain.nodes
        }

        render json: response, status: :created
    end

    def create_params
        params.permit(nodes:[])
    end
end
```

## blockchainの解決
登録しているノードと自分自身が持つblockchainの中から最長のものを採用します。
blockchainをアップデートするのでルーティングは

```rb
put '/blocks', to: 'blocks#update_all'
```
としています。

```rb
class BlocksController < ApplicationController
    ...
    def update_all
        blockchain = Blockchain.instance
        
        replaced = blockchain.resolve_conflicts

        if replaced
            response = {
                message: "Our chain was replaced",
                new_chain: blockchain.chain
            }
        else
            response = {
                message: "Our chain is authoritative",
                chain: blockchain.chain
            }
        end

        render json: response, status: :ok
    end
    ...
end
```

# docker-compose
blockchainの解決を確認するためには少なくともサーバーが２台必要なので、
dockerとdocker-composerを使って構築します。

```
version: '3'
services:
  rails1:
    build: .
    volumes:
      - ./:/var/www/blockchain
  rails2:
    build: .
    volumes:
      - ./:/var/www/blockchain
  curl:
    image: byrnedo/alpine-curl
```

これで

```bash
$ docker-compose up -d rails1 rails2
```
を実行するとサーバーが２台立ち上がります。


# 動作検証
サーバーを起動させた状態にしておきます。

APIはcurlコマンドで実行していきますが、rali1,rail2はdocker-composeネットワークの内部からしかアクセスできないため、docker-compose run curl を使ってcurlを実行するコンテナを立ち上げます。

オプションに「--rm」を付けて実行後にコンテナが削除されるようにしておきます。
また、結果はjson形式で渡ってきますが、見やすくするためjqコマンドで整形しています。

まず、rails1サーバーに対してマイニング(blockの追加)を行います。

```bash
$ docker-compose run --rm curl -X POST http://rails1/blocks | jq
```
```json
{
  "message": "New Block Forged",
  "index": 2,
  "transactions": [
    {
      "sender": "0",
      "recipient": "rails1",
      "amount": 100
    }
  ],
  "proof": 35293,
  "previous_hash": "2c6bc3ac80745d1b9df789379432fdb75da977b9940696507936c4f7cc11ee10"
}
```

この採掘でrails1に報酬として100coinが付与されました。
(便宜上扱う単位はcoinとします。)

今回はサーバーはユーザーが各１台たてているものとして、rails1はサーバー名かつユーザー名としています。

次にrails2側でもマイニングを行います。

```bash
$ docker-compose run --rm curl -X POST http://rails2/blocks | jq
```
```json
{
  "message": "New Block Forged",
  "index": 2,
  "transactions": [
    {
      "sender": "0",
      "recipient": "rails2",
      "amount": 100
    }
  ],
  "proof": 35293,
  "previous_hash": "715b974cfd57196c03dcfbbaa2d3eb23779296f751d39aedf6695e7fa9317f07"
}
```
このマイニングでrails2に報酬として100coinが付与されました。

rails2からrails1に10coinを渡してみます。
coinの移動はtransactionを作成することで行います。

```bash
$ docker-compose run --rm curl -X POST -H "Content-Type: application/json" -d '{
    "sender": "rail2",
    "recipient": "rails1",
    "amount": "10" 
}' "http://rails2/block/transactions" | jq
```
```json
{
  "message": "Transaction will be added to Block 3"
}
```
続いてrails2がもう一度マイニングを行います。

```bash
$ docker-compose run --rm curl -X POST http://rails2/blocks | jq
```
```json
{
  "message": "New Block Forged",
  "index": 3,
  "transactions": [
    {
      "sender": "rail2",
      "recipient": "rails1",
      "amount": "10"
    },
    {
      "sender": "0",
      "recipient": "rails2",
      "amount": 100
    }
  ],
  "proof": 35089,
  "previous_hash": "d1c26982d394e486cc21275838f24126bea3d9c6a3fe8a39b7341a7f65a171ce"
}
```
rail2からrails1へのtransactionと、rails2への報酬のtransactionを含んだblockが作成されました。

rails1とrails2のblockchainを取得してみます。

```bash
# rails1
$ docker-compose run --rm curl http://rails1/blocks
```
```json
{
  "chain": [
    {
      "index": 1,
      "timestamp": 1511929217,
      "transactions": [],
      "proof": 100,
      "previous_hash": 1
    },
    {
      "index": 2,
      "timestamp": 1511929217,
      "transactions": [
        {
          "sender": "0",
          "recipient": "rails1",
          "amount": 100
        }
      ],
      "proof": 35293,
      "previous_hash": "2c6bc3ac80745d1b9df789379432fdb75da977b9940696507936c4f7cc11ee10"
    }
  ],
  "length": 2
}
```

```bash
#rails2
$ docker-compose run --rm curl http://rails2/blocks
```
```json
{
  "chain": [
    {
      "index": 1,
      "timestamp": 1511929306,
      "transactions": [],
      "proof": 100,
      "previous_hash": 1
    },
    {
      "index": 2,
      "timestamp": 1511929306,
      "transactions": [
        {
          "sender": "0",
          "recipient": "rails2",
          "amount": 100
        }
      ],
      "proof": 35293,
      "previous_hash": "715b974cfd57196c03dcfbbaa2d3eb23779296f751d39aedf6695e7fa9317f07"
    },
    {
      "index": 3,
      "timestamp": 1511929526,
      "transactions": [
        {
          "sender": "rail2",
          "recipient": "rails1",
          "amount": "10"
        },
        {
          "sender": "0",
          "recipient": "rails2",
          "amount": 100
        }
      ],
      "proof": 35089,
      "previous_hash": "d1c26982d394e486cc21275838f24126bea3d9c6a3fe8a39b7341a7f65a171ce"
    }
  ],
  "length": 3
}
```

この時点でrails1のチェインよりrails2の方のチェインの方が長くなっています。
チェインは長いものが正当とするルールなので、rails2のチェインが正当なものになります。
rails1で正当なチェインを採用するように更新を行います。

rails1にrails2を登録します。

```bash
$ docker-compose run --rm curl -X POST -H "Content-Type: application/json" -d '{
    "nodes": ["http://rails2"]
}' http://rails1/nodes | jq
```
```json
{
  "message": "New nodes have been added",
  "total_nodes": [
    "rails2:80"
  ]
}
```

続いてrails1のblockchainの更新(解決)を行います。

```bash
$ docker-compose run --rm curl -X PUT http://rails1/blocks | jq
```
```json
{
  "message": "Our chain was replaced",
  "new_chain": [
    {
      "index": 1,
      "timestamp": 1511929306,
      "transactions": [],
      "proof": 100,
      "previous_hash": 1
    },
    {
      "index": 2,
      "timestamp": 1511929306,
      "transactions": [
        {
          "sender": "0",
          "recipient": "rails2",
          "amount": 100
        }
      ],
      "proof": 35293,
      "previous_hash": "715b974cfd57196c03dcfbbaa2d3eb23779296f751d39aedf6695e7fa9317f07"
    },
    {
      "index": 3,
      "timestamp": 1511929526,
      "transactions": [
        {
          "sender": "rail2",
          "recipient": "rails1",
          "amount": "10"
        },
        {
          "sender": "0",
          "recipient": "rails2",
          "amount": 100
        }
      ],
      "proof": 35089,
      "previous_hash": "d1c26982d394e486cc21275838f24126bea3d9c6a3fe8a39b7341a7f65a171ce"
    }
  ]
}
```

blockchainが更新されました。

チェインが更新されたので、rails1の採掘した100coinは消失しています。

# 最後に
以上、長くなってしまいましたがblockchainの実装をRailsでやってみました。

参考にさせて頂いた記事にあるように、実際にblockchainを作ってみると、抽象的だった内容がだいぶ理解できたように思います。

今回作成したソースは[こちら](https://github.com/g-ozawa/blockchain-rails)にあげてあります。

ご指摘などありましたら、コメントにてお願いします。
