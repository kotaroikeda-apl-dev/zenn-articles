---
title: "「Spanner EmulatorでSQLが落ちる」を1つの実装で潰す"
emoji: "🧪"
type: "tech"
topics: ["go", "spanner", "testing", "ci"]
published: false
---

ローカルでは動くのに、CI だけ落ちる。

しかも落ちているのはアプリ本体ではなく、Spanner Emulator に投げた SQL。
本番の Cloud Spanner では普通に通るのに、Emulator だと `null-filtered index` まわりで怒られたり、`FORCE_INDEX` 付きの SQL が `Internal` エラーでこけたりする。

自分が実際にこの事故を踏んだというより、現場の既存コードを読んでいて「なぜこんな補正が入っているんだろう」と気になったのが出発点でした。
追っていくと、そこには Emulator と本番の差異を吸収するための、かなり実務的な工夫がありました。
「じゃあ Emulator 用に SQL を別で持つか」とやり始めると、その瞬間からクエリの二重管理が始まります。
でも実務では、そこに保守コストを増やしたくありません。

この記事では、**Spanner Emulator と本番環境の SQL 差異をアプリ側で吸収して、1つの SQL を共通利用する実装パターン**を紹介します。
Go でそのまま使える形に寄せて書きます。

## 既存コードを読んで見えてきた問題

既存の Repository や SQL 補正ロジックを読んでいて、特に意図を感じたのが次の 2 パターンでした。

### 1. `null-filtered index` 関連のエラー

例えば、本番では問題なく使えているインデックス付きの検索が、Emulator だとこんな雰囲気のエラーで落ちます。

```text
The emulator is not able to determine whether the null filtered index ... can be used ...
```

要するに、**Emulator 側が「その index を使って大丈夫か」を安全に判断しきれず、クエリを拒否する**ケースです。

本番では通るので、最初は「クエリがおかしいのか、Emulator がおかしいのか」の切り分けがやりづらいです。

### 2. `FORCE_INDEX` 使用時のクラッシュ

もう 1 つつらかったのが、`FORCE_INDEX` を付けた SQL です。

本番では性能上の理由で index hint を付けているのに、Emulator では古いバージョンや実行条件によって `Internal` エラーで落ちることがありました。

```sql
UPDATE UserEvents@{FORCE_INDEX=UserEventsByUserID}
SET ProcessedAt = PENDING_COMMIT_TIMESTAMP()
WHERE UserID = @userID
```

この手の問題、体験としてはいちばん嫌です。

- ローカルでは動く
- 本番でも動く
- でも CI だけ落ちる

つまり、**コードの品質というより、実行環境の差異でパイプラインが不安定になる**んですよね。

## なぜ起きるのか

深掘りしすぎる必要はありません。
実務で押さえておけば十分なのは次の理解です。

- Spanner Emulator は Cloud Spanner の完全一致実装ではない
- クエリプランナーや hint まわりは、本番との差が出やすい
- Emulator のバージョン差もあり、CI イメージが古いと再現性がぶれやすい

つまり、**SQL 自体が間違っているとは限らないが、Emulator ではそのまま通らないことがある**、ということです。

ここで大事なのは、「環境ごとに SQL を書き分ける」ではなく、**差異を吸収する薄いレイヤーを 1 枚置く**という発想でした。

## まずは Before: SQL を環境ごとに書き分けるアンチパターン

最初にやりがちなのはこれです。

```go
func (r *UserEventRepository) MarkProcessedQuery() string {
    if os.Getenv("SPANNER_EMULATOR_HOST") != "" {
        return `
UPDATE UserEvents
SET ProcessedAt = PENDING_COMMIT_TIMESTAMP()
WHERE UserID = @userID
`
    }

    return `
UPDATE UserEvents@{FORCE_INDEX=UserEventsByUserID}
SET ProcessedAt = PENDING_COMMIT_TIMESTAMP()
WHERE UserID = @userID
`
}
```

一見シンプルですが、すぐにつらくなります。

- 同じクエリの本質が 2 箇所に散る
- 条件追加のたびに両方直す必要がある
- 本番用 SQL と Emulator 用 SQL が微妙にズレる
- レビュー時に「どっちが正なのか」が分かりにくい

数本ならまだしも、Repository が増えると地味に壊れやすいです。

## After: 1つの SQL を共通化し、実行前にだけ正規化する

やりたいことはシンプルです。

1. アプリケーション内では本番基準の SQL を 1 つだけ持つ
2. 実行直前に、Emulator なら必要な補正だけ入れる
3. 補正ロジックは共通関数に閉じ込める

イメージはこうです。

```go
baseSQL := `
UPDATE UserEvents@{FORCE_INDEX=UserEventsByUserID}
SET ProcessedAt = PENDING_COMMIT_TIMESTAMP()
WHERE UserID = @userID
`

query := NormalizeSQLForSpanner(baseSQL)
```

この `NormalizeSQLForSpanner()` の中で、今回の 2 つの差異を吸収します。

## 実装パターン

### 1. Emulator 判定

まずは環境判定です。
Cloud Spanner Emulator を使う場合、一般的には `SPANNER_EMULATOR_HOST` が設定されるので、これを見ます。

```go
package spannerutil

import "os"

func IsEmulator() bool {
    return os.Getenv("SPANNER_EMULATOR_HOST") != ""
}
```

### 2. `PrependEmulatorHint()`: Emulator 時にヒントを付与

`null-filtered index` 系のエラーを避けたい場合、Emulator では次の hint を先頭に付けます。

```go
@{spanner_emulator.disable_query_null_filtered_index_check=true}
```

これを毎回手で SQL に書くのではなく、関数化しておきます。

```go
package spannerutil

import "strings"

const emulatorNullFilteredIndexHint = "@{spanner_emulator.disable_query_null_filtered_index_check=true}"

func PrependEmulatorHint(sql string) string {
    trimmed := strings.TrimSpace(sql)
    if trimmed == "" {
        return sql
    }
    if strings.HasPrefix(trimmed, emulatorNullFilteredIndexHint) {
        return trimmed
    }
    return emulatorNullFilteredIndexHint + "\n" + trimmed
}
```

ポイントは、**SQL 定義側には環境差異を書かず、補正側に閉じ込める**ことです。

### 3. `StripForceIndexForEmulator()`: Emulator 時に `FORCE_INDEX` を削除

次に、Emulator で問題を起こしやすい `FORCE_INDEX` を落とします。

```go
package spannerutil

import "regexp"

var forceIndexRe = regexp.MustCompile(`@\{FORCE_INDEX=[^}]+\}`)

func StripForceIndexForEmulator(sql string) string {
    return forceIndexRe.ReplaceAllString(sql, "")
}
```

ここでは単純に `@{FORCE_INDEX=...}` を消しています。

厳密には、すべての `FORCE_INDEX` を一律除去していいかはプロジェクト次第です。
ただ、**Emulator を安定して動かすためのテスト用補正**としては、このくらい割り切る方が運用しやすいことが多いです。

### 4. まとめて適用する

実際には、実行前に 1 回だけ通す関数を用意しておくのが使いやすいです。

```go
package spannerutil

func NormalizeSQLForSpanner(sql string) string {
    if !IsEmulator() {
        return sql
    }

    normalized := StripForceIndexForEmulator(sql)
    normalized = PrependEmulatorHint(normalized)
    return normalized
}
```

これで呼び出し側はかなりシンプルになります。

```go
stmt := spanner.Statement{
    SQL: NormalizeSQLForSpanner(`
UPDATE UserEvents@{FORCE_INDEX=UserEventsByUserID}
SET ProcessedAt = PENDING_COMMIT_TIMESTAMP()
WHERE UserID = @userID
`),
    Params: map[string]any{
        "userID": userID,
    },
}
```

## Before / After 比較

見比べると、設計の差がかなりはっきりします。

### Before

```go
func buildFindUserSQL() string {
    if os.Getenv("SPANNER_EMULATOR_HOST") != "" {
        return `
SELECT *
FROM Users
WHERE TenantID = @tenantID AND Email = @email
`
    }

    return `
SELECT *
FROM Users@{FORCE_INDEX=UsersByTenantIDAndEmail}
WHERE TenantID = @tenantID AND Email = @email
`
}
```

### After

```go
const findUserSQL = `
SELECT *
FROM Users@{FORCE_INDEX=UsersByTenantIDAndEmail}
WHERE TenantID = @tenantID AND Email = @email
`

func buildFindUserStatement(tenantID, email string) spanner.Statement {
    return spanner.Statement{
        SQL: NormalizeSQLForSpanner(findUserSQL),
        Params: map[string]any{
            "tenantID": tenantID,
            "email":    email,
        },
    }
}
```

After の方が、クエリの責務が明確です。

- SQL 自体は 1 つだけ
- 環境差異は `NormalizeSQLForSpanner()` に集約
- Repository 側は純粋に業務クエリだけを持てる

現場だと、この差が後から効いてきます。

## 実運用での置き場所

この手の処理は、Repository ごとにバラバラに置かない方がいいです。
おすすめは、Spanner クライアントの薄いラッパーか、`spannerutil` のような共通パッケージに寄せることです。

例えば、読み取り・更新の入口を 1 箇所に寄せておくと、SQL の通し忘れを防ぎやすいです。

```go
package infra

type StatementBuilder struct{}

func (b StatementBuilder) New(sql string, params map[string]any) spanner.Statement {
    return spanner.Statement{
        SQL:    spannerutil.NormalizeSQLForSpanner(sql),
        Params: params,
    }
}
```

こうしておくと、各 Repository は SQL 本体に集中できます。

## このパターンのメリット

実際に運用して感じたメリットはかなり素直です。

- 同一コードで本番・ローカルの両方を動かせる
- CI だけ落ちる、ローカルだけ動く、というノイズが減る
- SQL の二重管理がなくなってメンテナンスしやすい
- 「どこで環境差異を吸収しているか」が明確になる

特に大きいのは、**テストが壊れにくくなること**です。

Emulator は便利ですが、本番と完全一致を期待しすぎると痛い目を見ます。
だからこそ、差異がある前提で受け止めて、アプリ側に薄い吸収レイヤーを置く方が安定します。

## CI/CD ではどう使うか

切り替えは環境変数ベースにしておくのが一番分かりやすいです。

アプリ側は `SPANNER_EMULATOR_HOST` を見て自動判定します。
つまり、CI では Emulator を起動してその環境変数を入れるだけで、同じコードがそのまま動きます。

GitHub Actions ならイメージはこんな感じです。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      spanner:
        image: gcr.io/cloud-spanner-emulator/emulator
        ports:
          - 9010:9010
          - 9020:9020

    env:
      SPANNER_EMULATOR_HOST: localhost:9010

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
      - run: go test ./...
```

この構成の良いところは、**アプリコード側に CI 専用分岐を増やさなくていい**ことです。

- ローカルで Emulator を使う
- CI でも Emulator を使う
- 本番では環境変数がないので補正が入らない

この流れにしておくと、「テストが壊れない仕組み」としてかなり扱いやすいです。

## 割り切りポイント

もちろん、このパターンは万能ではありません。

例えば、

- 本番でしか再現しない実行計画の問題
- Emulator ではまだ未対応の SQL 機能
- 本番の性能検証そのもの

このあたりは別途、本番相当環境での確認が必要です。

ただ、それでも日常の開発や CI の安定化という観点では、**差異を完全に消すより、差異を局所化する**方がコスパがいいです。

既存コードがここを割り切っていた理由も、読み進めるほどかなり納得感がありました。

## まとめ

Spanner Emulator と本番環境の差異は、想像以上に現場のノイズになります。

特に今回のように、

- `null-filtered index` で Emulator だけ落ちる
- `FORCE_INDEX` で一部環境だけ不安定になる

という問題は、SQL を環境ごとに書き分け始めると、あとで必ず保守が重くなります。

なので方針としてはシンプルです。

- SQL は 1 つだけ持つ
- 実行前にだけ環境差異を吸収する
- Emulator 判定は環境変数で行う

`PrependEmulatorHint()` と `StripForceIndexForEmulator()` を用意しておくと、この方針をかなり素直に実装できます。

**環境差異は、呼び出し側に漏らさず吸収する設計にした方が長く効く。**

既存コードの「なぜこの補正が必要なのか」が気になっているなら、まずは SQL を分岐させる前に、この薄い吸収レイヤーという見方で整理してみるのがおすすめです。
