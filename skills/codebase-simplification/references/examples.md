# 整理候補の例と負例

## 目次

1. 採用しやすい例
2. 挙動変更を含む例
3. 見送る例
4. PRの切り方

## 1. 採用しやすい例

### 判別子を入口へ集める

複数consumerが補助fieldの有無から種別を推測し、fallbackを持つ状態を疑う。
正本のdiscriminatorが存在するなら、入口でvalidateし、それだけで分岐する。

- 主操作: 移動。推測分岐とfallbackの削除は、その結果として数える。
- 品質軸: SSoT、不正状態の拒否、変更局所性
- 消えるもの: 推測分岐、fallback、leaf guard、補助field依存
- guard: 不正なdiscriminatorを入口でerrorにする型またはvalidation

### pass-through wrapperを畳む

引数を詰め替えるだけのwrapper、同じerrorをcatchして返すだけのhelper、1対1のDTO変換を疑う。

- 操作: 畳む
- deletion test: 削除時に知識がcallerへ再出現せず、経路自体が消えることを確認する。
- 測定: wrapper数、helper数、try/catch数、hop、net LOC

### wide DTOを能力へ狭める

consumerが巨大objectから1 fieldしか使わない場合、必要な値または限定したquery能力だけを渡す。

- 操作: 狭める
- 品質軸: 最小capability、変更局所性
- 消えるもの: unused field、prop drilling、不要mock setup、不要な変更波及

### 並行構造の突き合わせを上流で畳む

内訳ツリーを持つdomainが、金額などの派生値を「IDだけで並行する別ツリー」として返し、複数のconsumerが `find(id) + throw` のlookup helperで2つのツリーを突き合わせている状態を疑う。
このhelperが複数fileへ重複していても、指摘の語彙(重複)に従って共有helperへ統合するのは1段低い操作で止まっている。
生成元のadapterがノード参照または`Map`で金額を添えた形を返せば、ID結合という操作ごと消え、「対応する値が見つからない」という不正状態を型で作れなくなる。

- 操作: 畳む(生成元の出力形を変え、consumer側の突き合わせ経路を消す)
- 品質軸: 不正状態の拒否、変更局所性
- 消えるもの: lookup helper、call siteごとのID結合、実行時throw、並行構造という概念
- 負例: helperを共有moduleへ新設して2箇所から参照する統合。重複は消えるがID結合・throw・並行構造は残り、working setが純減しない。

### error handlingを根元へ移す

同じerror変換やretry判断をleafごとに持つ場合、意味を決められる入口またはownerへ寄せる。

- 操作: 移動
- 品質軸: SSoT、変更局所性
- 注意: errorを握り潰す共通handlerへ統合しない。callerが知るべきerror contractを保つ。

## 2. 挙動変更を含む例

### placeholder機能をsurfaceごと削除する

UIは存在するが保存できず、業務正本にもroadmapにもなく、利用者へ誤った期待だけを与える場合を考える。

- 操作: 削除
- 判断: business
- 証拠: 実装参照、到達性、業務正本の三点測量
- 変わる挙動: placeholder画面へ到達できなくなる。
- 維持する契約: 関連しない主要導線と既存データ。
- guard: route、navigation、schema、test、docsを同時に除去する。

「いつか実装する」というroadmapが確認できる場合は削除せず、statusとIssueを正本化して見送る。

### 不要な互換経路を打ち切る

移行期間が終わり、旧formatのproducerと保存データが存在しない場合、fallback parserを削除する。

- 操作: 削除
- 判断: designまたはbusiness
- 制約: 保存データと外部consumerを確認する。
- guard: 新formatを入口でfail-fastし、旧formatを新たに生成できないようにする。

## 3. 見送る例

### 行数だけ減る巨大関数

helperを全部inlineしてLOCを減らしても、複数の変更理由とerror contractが1関数へ集まるなら見送る。
working setと故障範囲が減っていない。

### 2回しか現れていない共通化

形が似ていても変更理由が違うなら統合しない。
3回という回数だけも根拠にせず、同じ業務意味と変更理由があるかを確認する。

### 形だけの重複への上位遡り

ほぼ同一のstyle定義や設定値の重複は、上流に生成元となるデータ形がない。
共通部分を1定義へ統合して止まるのが正しい。
selector方式の変更、markupへのclass付与、コンポーネント化へ遡ると、見た目は同じままworking setと概念が増える。
そうした構造変更に価値があるなら、統合とは独立した候補として分ける。

### 実装が1つのinterface削除

実装数だけを根拠に削除しない。
外部system隔離、依存方向、ownership、test seamの価値が実在する場合は残す。

### 高コストだが価値が実在する二重実装

real / mockの二重実装がpreview、VRT、offline開発を支える場合、コストだけで削除しない。
更新漏れを防ぐcontract testや生成へ問題を言い換える。

### 未完成機能を死仕様と誤認する

roadmapとIssueに実装予定があり、段階導入中なら削除候補にしない。
未実装範囲、status、完了条件を明示する。

## 4. PRの切り方

良い読み筋:

```text
正本のdiscriminatorを入口で必須化する
  → downstreamの推測分岐を削除する
  → fallback用testと型を削除する
  → 不正値の入口testを残す
```

分ける読み筋:

```text
PR 1: 新しい正本またはmigrationを導入する
PR 2: consumerを正本へ移す
PR 3: 旧経路・互換仕様・データ構造を削除する
```

PR 1がmergeされる前にPR 3を独立で出さない。
非競合の別moduleは並列で進める。
