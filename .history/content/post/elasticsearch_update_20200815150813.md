---
title: "Elasticsearch 5.6.3を7.4に上げた"
date: 2020-08-15T14:12:54+09:00
draft: false
---
  
最近会社の検索に曖昧検索を入れたいとの声があり、ベクトル検索できるようにする必要があった。  
ベクトル検索はElasticsearchの7.3系以上からのみ利用可能であり、既存が5系だったのでメジャーバージョンアップデートを2つ行う必要があり、色々苦労した。  

# 苦労したところ
1. synonymの読み込み方法の変更  
Elasticsearchの5系から6系にアップデートされた時点で劇的な変更が...
synonymファイルをkuromojiのアナライザのfilterに入れるとトークン化されてしまう。  
つまり、「資生堂,SHISEIDO,しせいどう」などがsynonymファイルに定義されている場合、  
「し,せい,どう」のように分割されてしまい、6系以上では検索時のノイズの原因になってしまう。  
参考: https://qiita.com/imura81gt/items/aba56c682aea600d5f9b

2. typelessにAPIが変更されていた  
5系まではindexを作成する際にtypeを指定していた。  
しかし7系からそれは非推奨になっており、既存のまま叩くと例外が発生する始末。  
[タイプの廃止！](https://www.elastic.co/jp/blog/moving-from-types-to-typeless-apis-in-elasticsearch-7-0)

3. 辞書とsynonymは運用時に随時追加しているのでテストの構成を変えた  
AWSのElasticsearchでは2020年の春まで個別でsynonymファイル、辞書ファイルを定義して、  
必要な時に随時更新することができなった。そのため2017年に作成された現システムではEC2にElasticsearchをインストールし、そこにsynonymや辞書ファイルを持つことで新しい商品追加時などの辞書登録に対応している。    
辞書登録のリポジトリは本体のAPIとは別で置いているため、辞書登録ごとに他の登録済みの単語などに影響が出ないかを随時テストコードを追加することで性能を担保していた。(ここは各社色々運用ありそうなので気になる。)  
このテストコードがシノニム定義ワードがトークン化される事件の影響を受けて落ちまくった。  

4. Elasticsearch7系でkuromojiのsearchモードで例外発生
kuromojiはnormalモードとsearchモードとcustomモードがあり何かを設定する。  
searchモードを指定すると辞書で定めたトークンに単語を分割するが、7系でsynonymファイルを読み込むとまさかの例外が発生するようになっていた。（地獄）  
7.9系では改善されるらしい。  
参考資料：[第36回Elasticsearch勉強会](https://noti.st/johtani/CEIYbT)  

5. 既存環境との併用  
いきなり7系にするのは危険。5系と併用してABテストを行い、実際に検索に新検索サーバのクラスタが悪影響を出さないかを検証しなければならず、インフラ環境を2つ用意し、  
また、バックエンドのAPIはRailsで動いているので request_store というライブラリを導入してスレッドごとのローカル変数を準備し、既存クラスタか新クラスタにユーザーを振り分ける必要があった。  
ABテストは[グノシーのこのブログの手法](https://data.gunosy.io/entry/ab_testing_assignment)を利用した。  
request_storeのリポジトリ: https://github.com/steveklabnik/request_store

# 上記の項目の対応  
1についてはまずsynonymを読み込む専用のAnalyzerを定義しなおしてやった.  
既存は以下のような形でfilterでsynonymを読み込むように定義していた。(synonym部分は割愛) 
{{< highlight html >}}
tokenizer: {
              japanese_search: {
                  mode: 'search',
                  type: 'kuromoji_tokenizer',
                  user_dictionary: "user_dictionary.txt",
                  discard_punctuation: false
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
        # keywordタイプの場合はhtml_stripが上手く動かないので自前で作成
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
[助けられたブログ](http://chie8842.hatenablog.com/entry/2019/09/29/124500)