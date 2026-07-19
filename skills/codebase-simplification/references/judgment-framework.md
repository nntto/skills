# 整理候補の判定フレームワーク

## 目次

1. 用語
2. 制約と判断者
3. 証拠の強さ
4. 測定
5. 採用質問
6. interfaceとseam
7. 優先順位

## 1. 用語

- **正本**: その意味を変更するときに最初に更新すべき場所。業務仕様、ADR、DB制約、型、policyなど、意味ごとに所在は異なる。
- **working set**: 1つの変更を安全に理解・実装・検証するために読む、変える、壊しうる範囲。
- **module**: interfaceとimplementationを持つ単位。関数、class、package、業務sliceのどれでもよい。
- **interface**: signatureだけでなく、callerが知る必要のある型、invariant、ordering、error mode、設定、性能特性を含む。
- **depth**: 小さいinterfaceの背後に、callerへ価値のある能力を隠す度合い。
- **seam**: その場を直接編集せず、挙動を差し替えられる地点。
- **adapter**: seamのinterfaceを満たす具体実装。

depthはcallerへleverageを与え、保守者へlocalityを与える。
ただし、深いmoduleを作ること自体を目的にしない。

### 近接概念との違い

- **cohesion**: module内部が同じ理由で変わるかを問う。変更局所性は、1つの変更タスクがまたぐ範囲を問う。
- **coupling**: 依存の本数と強さを問う。変更局所性は、実際の変更経路の長さを問う。
- **DRY**: コードの形ではなく意味の重複を避ける。偶然似たコードは局所に残してよい。
- **depth**: 小さいinterfaceへ能力を隠す構造上の手段。正しいownerから遠い共通moduleへ隠すと、depthがあっても変更局所性は下がる。

## 2. 制約と判断者

対象を次へ分類する。

| 分類 | 例 | 変更条件 |
| --- | --- | --- |
| hard constraint | DB永続データ、公開API、認可、課金、監査、法令 | migration、互換性、承認、rollbackを明示する |
| soft convention | naming、directory慣例、推奨pattern | 改善効果が勝るなら更新可能。規約側も同期する |
| accidental structure | 歴史的配置、pass-through wrapper、不要DTO | 正本と利用実態を確認して削除・再配置する |

判断を次へ分類する。

- **business**: 利用者価値、提供機能、業務運用を変える。
- **design**: 正本の所有者、抽象化、境界、データ表現を変える。
- **mechanical**: 正本と挙動を変えず、明白な重複や中継を除く。

分類が混ざる場合は、PRを分けるか最も強い承認条件を採用する。

## 3. 証拠の強さ

削除は「価値が見えない」ではなく「価値の不在を確認した」と言える強さを求める。

### 三点測量

対象に応じた3つの独立ソースを使う。

1. 実装: 書き込み、読み取り、import graph、runtime call、test経路
2. 到達性: UI route、API consumer、job、CLI、外部integration
3. 正本: 業務仕様、ADR、PRD、運用手順、roadmap

1つでも価値が確認できた場合は、削除ではなく完成、明文化、狭小化、隔離を検討する。

### 証拠の注意

- grepは候補抽出に使い、一般語の件数を価値不在の証拠にしない。
- framework、生成物、DB view、間接利用、reflection、外部consumerを確認する。
- 未完成機能は、不要機能ではなくroadmap上の状態である可能性を確認する。
- 進行中Issue / PRに載る候補は新発見と数えず、完了条件の欠落だけを追う。

## 4. 測定

変更前後で、該当するものを実数または概算で比較する。

- production / test / config / docsのnet LOC
- 消える業務概念、型、状態、分岐、fallback、互換経路
- consumerへ渡すfield、parameter、callback
- import、依存、公開symbol
- 読むfile、呼び出しhop、layer
- 変更理由を共有するmodule
- migration対象件数、rollback難度

net LOCだけを最大化しない。
次を削減に数えない。

- 複数処理を1行へ圧縮する。
- 巨大関数へinlineして名前と検証可能性を失う。
- 型安全性、必要なguard、意味のあるtestを消す。
- production codeをtest helperや設定へ移しただけにする。
- renameや移動を削除として数える。

LOCが増えても、不正状態の排除、認可、データ保護、循環依存の解消によりworking setや故障範囲が大きく減る場合は採用できる。

## 5. 採用質問

候補ごとに答える。

1. 正本と業務価値はどこにあるか。
2. 整理対象の最上位階層はどこか。
3. 6操作のどれか1語で言えるか。
4. 4品質軸のどれを改善するか。
5. 消える概念、分岐、依存、hop、LOCを数えられるか。
6. 観察可能な挙動はどう変わるか。維持する契約は何か。
7. hard / soft / accidentalのどれか。
8. business / design / mechanicalのどの判断か。
9. 価値の不在を主張する場合、3つの独立ソースが揃うか。
10. migration、可逆性、rollback、データ影響を説明できるか。
11. 同じ負債の再発を止めるguardと逃げ道封鎖を同梱できるか。
12. やらない場合のfailure modeまたは近い将来のchurnがあるか。
13. replacement込みでworking setと概念が純減するか。
14. 1 PR 1読み筋に収まるか。依存する場合はstackを切れるか。

答えが不足する候補は、即座に捨てず `insufficient-evidence` または `要判断` として記録する。
品質軸を改善しても6操作に収まらず、消えるものがない候補は、bug、feature、guardrail改善として別の台帳へ移す。

## 6. interfaceとseam

`deletion test` でpass-throughを見つける。

- moduleを消すと、interface、変換、error処理ごと複雑さが消える: 畳む候補。
- moduleを消すと、同じ知識が複数callerへ再出現する: 正本として残す候補。

依存は次の違いを考慮する。

1. **in-process**: 純粋計算やmemory内状態。不要seamを作らず直接testする。
2. **local-substitutable**: local DBやfilesystem代替がある。外部interfaceへtest都合を漏らさない。
3. **remote-owned**: 自組織のremote service。transport seamとadapterが価値を持ちうる。
4. **true-external**: 管理外の第三者service。外部契約を隔離するadapterが価値を持ちうる。

実装が1つでも、依存方向や外部契約隔離という価値があれば残す。
testだけのために全consumerへportを露出しない。

## 7. 優先順位

優先度を次で決める。

```text
priority = future change frequency × failure impact × simplification confidence
```

- 将来変更頻度: churn、roadmap、次の実装依存
- failure impact: 本番データ、認可、課金、主要業務、故障範囲
- simplification confidence: 正本、利用実態、試作、検証方法の強さ

過去の汚れを均等に掃除せず、これから繰り返し通る変更経路を先に短くする。
