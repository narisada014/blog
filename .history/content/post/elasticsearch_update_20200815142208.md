---
title: "Elasticsearch 5.6.3を7.4に上げた"
date: 2020-08-15T14:12:54+09:00
draft: false
---

# 苦労したところ
Elasticsearchの5系から6系にアップデートされた時点で劇的な変更が...
synonymファイルをfilterに入れるとトークン化されてしまう。  
つまり、「資生堂,SHISEIDO,しせいどう」などがシノニムファイルに定義されている場合、  
し,せい
