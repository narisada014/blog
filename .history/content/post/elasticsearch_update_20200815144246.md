---
title: "Elasticsearch 5.6.3を7.4に上げた"
date: 2020-08-15T14:12:54+09:00
draft: false
---

# 苦労したところ
1. synonymの読み込み方法の変更  
Elasticsearchの5系から6系にアップデートされた時点で劇的な変更が...
synonymファイルをfilterに入れるとトークン化されてしまう。  
つまり、「資生堂,SHISEIDO,しせいどう」などがsynonymファイルに定義されている場合、  
「し,せい,どう」のように分割されてしまい、6系以上では検索時のノイズの原因になってしまう。  

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



[助けられたブログ](http://chie8842.hatenablog.com/entry/2019/09/29/124500)