+++
title =  "Elasticsearch 5.6.3を7.4にアップデートした"
date = 2020-08-15T14:12:54+09:00
draft = false
images = ["images/thumbnail.png"]
+++
  
最近検索に曖昧検索を入れたいとの声があり、ベクトル検索できるようにする必要があった。  
ベクトル検索はElasticsearchの7.3以上からのみ利用可能で、既存が5系だったのでメジャーバージョンアップデートを行う必要があり色々苦労したのでメモとして残しておく。 

# 環境  
サーバサイド： Rails  
Elasticsearch：5.6.3 -> 7.4.2  
インフラ： EC2のクラスタ(Ansibleで生成)

# 苦労したところ
1. synonymの読み込み方法の変更  
Elasticsearchの5系から6系にアップデートされた時点で劇的な変更が...
synonymファイルをkuromojiのアナライザのfilterに入れるとトークン化されてしまう。  
つまり、「資生堂,SHISEIDO,しせいどう」などがsynonymファイルに定義されている場合、  
「し,せい,どう」のように分割されてしまい6系以上では検索時のノイズの原因になってしまう。  
参考: https://qiita.com/imura81gt/items/aba56c682aea600d5f9b



2. typelessにAPIが変更されていた  
5系まではindexを作成する際にtypeを指定していた。  
しかし7系からそれは非推奨になっており、既存のまま叩くと例外が発生する始末。 
これはindexを作成する際にtypeキーをElasticsearchのクライアントから消してやればOK。  
`_doc`というtypeであらゆる項目が登録される仕様になっている。 
[タイプの廃止！](https://www.elastic.co/jp/blog/moving-from-types-to-typeless-apis-in-elasticsearch-7-0)



3. 辞書とsynonymは運用時に随時追加しているのでテストの構成を変えた  
AWSのElasticsearchでは2020年の春まで個別でsynonymファイル、辞書ファイルを定義して、  
必要な時に随時更新することができなかった。そのため2017年に作成された現システムではEC2にElasticsearchをインストールし、そこにsynonymや辞書ファイルを持つことで新しい商品追加時などの辞書登録に対応している。    
辞書登録のリポジトリは本体のAPIとは別で置いているため、辞書登録ごとに他の登録済みの単語などに影響が出ないかを随時テストコードを追加することで性能を担保していた。(ここは各社色々運用ありそうなので気になる。)  
このテストコードがシノニム定義ワードがトークン化される影響を受けて落ちまくった。  



4. Elasticsearch7系でkuromojiのsearchモードで例外発生  
kuromojiはnormalモードとsearchモードとcustomモードがあり何かを設定する。  
searchモードを指定すると辞書で定めたトークンに単語を分割するが、7系でsynonymファイルを読み込むとまさかの例外が発生するようになっていた。（地獄）  
ver 7.9では改善される。  
参考資料：[第36回Elasticsearch勉強会](https://noti.st/johtani/CEIYbT)  



5. 既存環境との併用  
いきなり7系にするのは危険。5系と併用してABテストを行い、実際に検索に新検索サーバのクラスタが悪影響を出さないかを検証しなければならずインフラ環境を2つ用意し、バックエンドのAPIはRailsで動いているので request_store というライブラリを導入してスレッドごとのローカル変数を準備し、既存クラスタか新クラスタにユーザーを振り分ける必要があった。  
ABテストは[グノシーのこのブログの手法](https://data.gunosy.io/entry/ab_testing_assignment)を利用した。設計力がやや試される部分だった。  
request_storeのリポジトリ: https://github.com/steveklabnik/request_store  


6. ESサーバのクラスタを作成する際の設定の大幅な変更  
ElasticsearchをEC2でクラスタ構成で利用する場合各ノードが同期する必要がある。  
その際に使用するミドルウェアが`discovery-es2`である。  
この設定が5系と7系で大きく違ったのでハマった。  
また、インスタンス内でのElasticsearchが書き込むlogのディレクトリに対する権限も以前とは異なっていたのでハマった。
  


# 対応の詳細  
## ~インフラ~

discovery-ec2の7系の設定の変更を行った。elasticsearch.ymlに定義してやる。  
5系と違い、master_nodeがどれであるかを予め指定してinitしてやる必要があった。  
下記構成の場合はmaster_nodeは全てのノードである。
{{< highlight html >}}
node.name: "your host name"
cluster.name: "your cluster name"
discovery.seed_providers: ec2 # es5からキー名が変更されている
discovery.seed_hosts:
  - "XXX.XX.XX.XX"
  - "XXX.XX.XX.XX"
  - "XXX.XX.XX.XX"
cluster.initial_master_nodes: # クラスタ初期構築時にどのノードがマスターかを投票で決定するので必要。インデックスなどのdataを吹っ飛ばす設定でもあるので必要な初期以外は外すこと。ここに記載したノードは全てmaster適格ノードとして登録される
  - "XXX.XX.XX.XX"
  - "XXX.XX.XX.XX"
  - "XXX.XX.XX.XX"
discovery.ec2.groups: "your security group id" # ノードが存在するセキュリティグループ
discovery.ec2.availability_zones: ["your zone 1", "your zone 2"]
{{< /highlight >}} 

logへの書き込み権限はとりあえず下記でOK。本質的じゃないのでスルー気味で行く。  
おそらくここが結構ハマりポイントな気がする。自分の場合はelasticsearchが吐くログを追って権限がないことに気づいた。  
**ログを探して読むのは大事(自戒)**
```
/etc/sysconfig/elasticsearch owner=root group=elasticsearch  
/etc/elasticsearch/elasticsearch.yml owner=root group=elasticsearch
```  
  　　
  
## ~シノニム周り~  

まずsynonymを読み込む専用のAnalyzerを定義しなおしてやった.  
既存は以下のような形でfilterでsynonymを読み込むように定義していた。(synonym部分は割愛) 
{{< highlight html >}}
tokenizer: {
    japanese_search: {
        mode: 'search',
        type: 'kuromoji_tokenizer',
        user_dictionary: "user_dictionary.txt",
        discard_punctuation: false
    },
},
analyzer: {
  ja_analyzer: {
      type: 'custom',
      # kuromojiでトークナイズ(単語分割)
      tokenizer: 'japanese_search',
      # 文字単位のフィルター。トークナイズ前に適用される。
      char_filter: [
          'icu_normalizer',
          'html_strip',
          'kuromoji_iteration_mark'
      ],
      # トークン単位のフィルター。トークナイズ後に実行される。
      filter: [
          "synonym",
          "kuromoji_baseform",
          "ja_stop",
          'kuromoji_part_of_speech',
          'kana_converter',
          "kuromoji_number",
          'katakana_stemmer'
      ],
  },....
{{< /highlight >}}  

これを以下のように変更してやった。
{{< highlight html >}}
tokenizer: {
    japanese_search: {
        mode: 'search',
        type: 'kuromoji_tokenizer',
        user_dictionary: "user_dictionary.txt",
        discard_punctuation: false
    },
    keyword_search: {
      type: "keyword",
      user_dictionary: "user_dictionary.txt",
      discard_punctuation: false
    }
},
char_filter: {
    symbol: {
        type: 'mapping',
            mappings: [
                "`=>"
            ]
        },
    # tokenizerがkeywordタイプの場合はhtml_stripが上手く動かないので自前で作成する必要あり
    remove_html: {
        type: "pattern_replace",
        pattern: "<(`.*?`|'.*?'|[^'`])*?>",
        replacement: ""
    }
},
analyzer: {
  ja_analyzer: {
      type: 'custom',
      # kuromojiでトークナイズ(単語分割)
      tokenizer: 'japanese_search',
      # 文字単位のフィルター。トークナイズ前に適用される。
      char_filter: [
          'icu_normalizer',
          'html_strip',
          'kuromoji_iteration_mark'
      ],
      # トークン単位のフィルター。トークナイズ後に実行される。
      filter: [
          "kuromoji_baseform",
          "ja_stop",
          'kuromoji_part_of_speech',
          'kana_converter',
          "kuromoji_number",
          'katakana_stemmer'
      ],
  },
  keyword_analyzer: {
      type: 'custom',
      tokenizer: 'keyword_search',
      char_filter: [
          'icu_normalizer',
          'kuromoji_iteration_mark',
          'symbol',
          'remove_html'
      ],
      filter: [
          "synonym",
          "kuromoji_baseform",
          'kuromoji_part_of_speech',
          "ja_stop",
          'kana_converter',
          "kuromoji_number",
          'katakana_stemmer',
      ]
  },...
{{< /highlight >}}  

この変更でsynonymをkeywordとしてElasticsearchのドキュメントに登録できるようになった。  
コメントにも書いたように、keywordタイプを指定した場合html_stripがうまく効かなくなるので、自前で正規表現を使用して<hogehoge>のようなhtmlタグを排除する仕組みを定義した。  
また、synonymの読み込みをkuromojiから切り離したおかげで`mode: search`が使用可能になった。(結構たまたまなので大声で叫びましたw)  

このようにアナライザを修正したら次はmappingの修正とクエリの修正が必要になる。  
今までは一つのアナライザで全て済んでいた。しかし、2つのアナライザを利用しなければならないためElasticsearchのmulti_matchクエリの`cross_field`を利用する必要がある。  
cross_fieldについては下記のブログに助けられた。(尚、こちらの方がElasticserachのIssueにも同様の内容を記載してくださっていたので参考にさせていただきました。)  
下記記事の通りなのでmappingもクエリについてもあまり言及しないが、multi matchクエリは以下のように使用した。  
[助けられたブログ](http://chie8842.hatenablog.com/entry/2019/09/29/124500)  

{{< highlight html >}}
{
  query: {
    bool: {
      must:
        {bool: {
          should: [
            {
              multi_match: {
                query: query,
                type: "cross_fields",
                # シノニム用とkuromoji用の2つのフィールド定義
                fields: [ "hogehoge.keyword_field", "hogehoge.japanese_field" ],
                # どっちも使いたいのでAND指定
                operator: "and"
              }
            },....
          ]
        }}
      }
  }
}
{{< /highlight >}}  

  
  　　
## ~API周り~   
2のtypelessに関してはひたすらtype指定を消すことで問題ない。  
5のスレッドローカル変数については設計上各コントローラは共通の継承元のコントローラから派生しているため、その継承元にRequestStoreを利用したユーザーの振り分けロジックを記載した。
RequestStoreの使用方法は以下の通り。  
{{< highlight html >}}
  RequestStore.store[:hogehoge] = "es7"
{{< /highlight >}} 
このように指定してやるだけでそのユーザーのそのリクエストに限ってはグローバル変数として有効化される。一時的に5系と7系を併用するので後々消すという意味であまり意識の高い実装を行っていない。 
グローバル変数は何かと問題にもなるので検証後は全てを消して新クラスタにむけたAPIの修正をいれる予定。   
下記記事も参考に目を通しておいた。  
[Railsの`CurrentAttributes`は有害である（翻訳）](https://techracho.bpsinc.jp/hachi8833/2017_08_01/43810)  

また、APIの修正に際して運用中のECアプリではElasticsearchはあらゆるロジックに影響を与えるため、バッチ処理においても同様の改修・追加を行った。基本的にRequestStoreを使用し、7系に向けたバッチなのか5系に向けたバッチなのかを切り分けて対応した。  
　　
　　　　  
## ~テスト~
既存のテストはkuromojiのアナライザを通った後に辞書ファイル・synonymファイルに定義した通りにアナライズされるかをテストするものだった。今回はアナライザを分けたので、
1. synonymファイルに定義した単語の通りにアウトプットされて欲しいテスト
2. 辞書ファイルに定義した単語の世折にトークン化されることを確認するテスト  

以上の二つのテストに分割した。  
  　　
## 終わりに
バックエンドの設計やインフラ力・検索エンジンの知識・辞書登録の運用など幅広く業務における力を試されるアップデートだったので非常に良い経験になった。
特にアナライザを分ける際や辞書のリポジトリのテストを変更する場合、普段の作業者のタスクの精度にも影響が出る可能性があったのでドキュメントの整備やなるべく既存の運用と変わらないようにShellスクリプトで作業のラッパーを書いて自動化できる部分は自動化するなどサービス運用観点でも大きく学びがあった。
周りをよく見ることができるエンジニアでありたいので今後も視野広めでやっていくつもりです！！！！