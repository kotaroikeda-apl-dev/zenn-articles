---
title: "重いI/Oを含むユースケースで、トランザクションを局所化する理由"
emoji: "🧭"
type: "tech"
topics: ["go", "ddd", "cleanarchitecture", "database"]
published: false
---

「トランザクションはユースケース単位で張るべき」という原則は、設計の出発点としてとても分かりやすいです。

クリーンアーキテクチャやDDDの文脈でも、アプリケーションサービスやユースケースをひとまとまりの整合性境界として捉える考え方には納得感があります。

ただ、実務ではそれが常に最適とは限りません。

特に、DBアクセスのあとに重いファイルダウンロードやZIP圧縮のような処理が続くユースケースでは、ユースケース全体を1トランザクションで囲うと、整合性のために必要な時間以上にDB資源を持ち続けてしまうことがあります。

この記事では、Goアプリケーションでのエクスポート処理を例にしながら、**トランザクション境界はユースケース単位で固定するものではなく、整合性要件・処理コスト・外部I/Oの有無で決めるべき**、という話をします。

原則を否定したいのではなく、原則を知ったうえで状況に応じて外せることが、設計の成熟だと思っています。

## まず前提: 「1ユースケース = 1トランザクション」は良い原則

最初に明確にしておきたいのは、ユースケース単位でトランザクションを張る考え方自体は間違いではない、ということです。

この設計には、少なくとも次のような利点があります。

- 整合性境界が分かりやすい
- 実装者ごとの判断ブレが減る
- 途中失敗時のロールバック方針を説明しやすい
- アプリケーションサービスの責務が見えやすい

一方で、分かりやすさはあくまで「原則として扱いやすい」という話です。

実行時間が長い処理や、DB以外の外部資源を巻き込む処理までまとめてしまうと、その分だけトランザクションは重くなります。

大事なのは、**何を守るためにトランザクションを張っているのか**を明確にすることです。

## 問題設定: エクスポート処理を考える

例えば、次のようなエクスポート用ユースケースを考えます。

1. DBからエクスポート対象のメタデータ一覧を取得する
2. 各ファイルをストレージや外部システムからダウンロードする
3. それらをZIPにまとめる
4. 呼び出し元へ返す

これを素直に見ると、「エクスポート」という1つのユースケースなので、全部を1トランザクションに入れたくなるかもしれません。

ただ、ここで冷静に分解すると、整合性を強く求めたいのは主に「どのファイルを対象として扱うかを確定するDB読み取り」の部分です。

その後のダウンロードやZIP圧縮は、時間もかかるし、DBの外で完結する処理です。

もしDB取得からZIP圧縮完了まで同一トランザクションで囲うと、次のようなつらさが出ます。

- ファイルダウンロード待ちのあいだもDB資源を保持し続ける
- ZIP圧縮でCPUを使っているあいだもトランザクションが開きっぱなしになる
- 並列ダウンロードしたくても、トランザクション境界との責務が混ざりやすい
- その結果、スループットやスケーラビリティが落ちやすい

## なぜユースケース全体をトランザクションで囲わないのか

### DBコネクション保持時間が長くなる

DBトランザクションのコストは、SQL実行時間だけではありません。

トランザクションを開いているあいだ、そのセッションや関連資源を保持し続けること自体がコストです。

重いI/OやCPU処理をトランザクション内に含めると、実際にはDBに触っていない時間までDB資源を専有する形になります。

これは単純にもったいないですし、同時実行数が増えるほど効いてきます。

### I/O待ちの時間までDB資源を持つのは無駄

外部ストレージからのダウンロードは、ネットワーク状況や相手先の負荷に影響されます。つまり、速いときもあれば遅いときもある処理です。

その不確実な待ち時間をDBトランザクションの中に入れると、DB側から見ると「何もしていないのに長く居座る処理」になりがちです。

ユースケースとしては正しくても、システム全体では非効率です。

### 競合やスループットの面でも不利

トランザクションが長いほど、他の処理と競合する可能性は高まります。

更新系ならロック競合やデッドロックの話がより重要になりますし、読み取り中心でも、長寿命なトランザクションが増えると全体のスループットには影響します。

だからこそ、**整合性を守るために本当に必要な範囲だけをトランザクションに閉じ込める**という考え方が有効です。

## Fetch / Process / Return で分けて考える

今回のようなユースケースは、次の3フェーズに分けて考えると整理しやすいです。

### Fetch (DB)

読み取り専用トランザクションなどを使って、後続処理に必要なメタデータだけを短時間で取得します。

- 対象ID
- ファイルパス
- バケット名
- ZIP内のファイル名
- 必要ならバージョンや更新時刻

ここでは「後で処理するのに必要な情報」を確定できれば十分です。

トランザクションは早めに閉じます。

### Process (App)

ここから先はアプリケーション層の仕事です。

- `errgroup`で並列ダウンロードする
- メモリ上や一時ファイル上でZIPにまとめる
- 必要な変換や整形を行う

このフェーズでは、DBトランザクションはもう不要です。

むしろ切り離されている方が、並行処理の設計に集中できます。

### Return (Entity)

最後に、ZIPバイト列やファイル名などを、ドメインオブジェクトやレスポンス用構造体に包んで返します。

ここを明示しておくと、「DBから取った生データをそのまま返す」のではなく、「アプリケーションとして返す結果を組み立てる」責務が見えやすくなります。

この流れは、言い換えると**Fetch & Process型**の設計です。

整合性が必要な情報取得は短く閉じ、重い処理はDBの外で行います。

## メタデータ取得を短いトランザクションに閉じ込める例

以下は、エクスポート対象のメタデータだけを短い読み取りトランザクションで取得する簡略化例です。

ポイントは、「メタデータ取得」だけをトランザクション境界に閉じ込めていることです。

このあとに続くダウンロードやZIP圧縮は、トランザクションの外で行います。

```go
package repository

import (
	"context"
)

type ExportMetadata struct {
	ReportID    string
	BucketName  string
	ObjectPath  string
	ArchiveName string
	Version     int64
}

type Rows interface {
	Next() bool
	Scan(dest ...any) error
	Close() error
	Err() error
}

type ReadTx interface {
	QueryContext(ctx context.Context, query string, args ...any) (Rows, error)
}

type TxManager interface {
	ReadOnly(ctx context.Context, fn func(ctx context.Context, tx ReadTx) error) error
}

type ReportQueryService struct {
	txManager TxManager
}

func (s *ReportQueryService) FindExportMetadata(
	ctx context.Context,
	reportIDs []string,
) ([]*ExportMetadata, error) {
	var result []*ExportMetadata

	err := s.txManager.ReadOnly(ctx, func(ctx context.Context, tx ReadTx) error {
		rows, err := tx.QueryContext(ctx, `
SELECT report_id, bucket_name, object_path, archive_name, version
FROM report_exports
WHERE report_id IN (?)
  AND deleted_at IS NULL
`, reportIDs)
		// `IN (?)` は簡略化のための表現です。
		// 実際には sqlx / query builder / プレースホルダ展開など、
		// 利用ライブラリに応じた書き方に置き換えてください。
		if err != nil {
			return err
		}
		defer rows.Close()

		for rows.Next() {
			var m ExportMetadata
			if err := rows.Scan(
				&m.ReportID,
				&m.BucketName,
				&m.ObjectPath,
				&m.ArchiveName,
				&m.Version,
			); err != nil {
				return err
			}
			result = append(result, &m)
		}

		return rows.Err()
	})
	if err != nil {
		return nil, err
	}

	return result, nil
}
```

この例で伝えたいポイントは3つです。

### 読み取りは短く閉じる

ここでは「メタデータを確定する」ことが目的なので、読み取り専用トランザクションで十分です。

後続のダウンロードやZIP圧縮をここに含めないことで、DB側の責務を明確に分離できます。

### トランザクション境界と処理境界を一致させない

ユースケース全体ではなく、「整合性を持って読みたい部分」にだけトランザクションを張る、という考え方です。

ここで境界を切っておくと、その後の重い処理をアプリケーション層へ素直に逃がせます。

### クエリの最適化は別軸で必要

本題はトランザクション境界ですが、Fetchフェーズ自体も速く終わるようにしておく必要があります。

インデックス設計、取得カラムの絞り込み、不要なJOINを避けるといった基本は、ここでも効いてきます。

## ダウンロードとZIP化はアプリケーション層へ逃がす

メタデータを取得したら、以降はDBから切り離して処理します。

次は、`errgroup`で並列ダウンロードし、その結果をZIPにまとめる簡略化例です。

```go
package usecase

import (
	"archive/zip"
	"bytes"
	"context"
	"fmt"
	"io"
	"sync"

	"golang.org/x/sync/errgroup"
)

type FileDownloader interface {
	Download(ctx context.Context, bucketName, objectPath string) ([]byte, error)
}

type ExportMetadata struct {
	ReportID    string
	BucketName  string
	ObjectPath  string
	ArchiveName string
}

type ExportArchive struct {
	FileName string
	Body     []byte
}

type downloadedFile struct {
	archiveName string
	body        []byte
}

type ExportUseCase struct {
	queryService QueryService
	downloader   FileDownloader
}

type QueryService interface {
	FindExportMetadata(ctx context.Context, reportIDs []string) ([]*ExportMetadata, error)
}

func (u *ExportUseCase) Execute(ctx context.Context, reportIDs []string) (*ExportArchive, error) {
	metas, err := u.queryService.FindExportMetadata(ctx, reportIDs)
	if err != nil {
		return nil, fmt.Errorf("fetch export metadata: %w", err)
	}

	files := make([]downloadedFile, 0, len(metas))
	var mu sync.Mutex

	eg, egCtx := errgroup.WithContext(ctx)
	for _, meta := range metas {
		meta := meta

		eg.Go(func() error {
			body, err := u.downloader.Download(egCtx, meta.BucketName, meta.ObjectPath)
			if err != nil {
				return fmt.Errorf("download %s: %w", meta.ReportID, err)
			}

			mu.Lock()
			files = append(files, downloadedFile{
				archiveName: meta.ArchiveName,
				body:        body,
			})
			mu.Unlock()
			return nil
		})
	}

	if err := eg.Wait(); err != nil {
		return nil, err
	}

	archiveBody, err := buildZIP(files)
	if err != nil {
		return nil, fmt.Errorf("build zip: %w", err)
	}

	return &ExportArchive{
		FileName: "reports.zip",
		Body:     archiveBody,
	}, nil
}

func buildZIP(files []downloadedFile) ([]byte, error) {
	var buf bytes.Buffer
	zw := zip.NewWriter(&buf)

	for _, f := range files {
		w, err := zw.Create(f.archiveName)
		if err != nil {
			return nil, err
		}
		if _, err := io.Copy(w, bytes.NewReader(f.body)); err != nil {
			return nil, err
		}
	}

	if err := zw.Close(); err != nil {
		return nil, err
	}

	return buf.Bytes(), nil
}
```

これは記事用にかなり簡略化した例ですが、意図は十分伝わるはずです。

## 並行処理と整合性のバランス

この構成のよいところは、DBトランザクションの責務と、アプリケーション側の重い処理の責務を素直に分けられることです。

### `errgroup`で並列ダウンロードする

ファイルダウンロードは待ち時間が支配的になりやすいので、1件ずつ直列で取るより、並列に流した方がレスポンス改善を見込みやすいです。

また、`errgroup.WithContext`を使えば、どれか1つが失敗したときに他の処理も止めやすくなります。

重いI/Oを扱うときは、このキャンセル伝搬がかなり大事です。

### 共有データは素直に保護する

並列ダウンロードした結果をスライスやMapに集めるなら、`sync.Mutex`などで保護します。

ここで雑に実装すると、せっかくトランザクションをきれいに分けても、今度はアプリケーション側でデータ競合を作ってしまいます。

### トランザクションは短く、重い処理は安全に並列化する

この役割分担がポイントです。

- DBトランザクションは短く切る
- 重いI/OやCPU処理はアプリケーション層で扱う
- 並行処理は`context`と`errgroup`で制御する

つまり、性能チューニングの話というより、**責務分離に基づいた設計判断**の話です。

## それでも注意したいトレードオフ

もちろん、この分割がいつでも正しいわけではありません。

### 取得時点と処理時点でズレる可能性はある

メタデータ取得後に、元データが更新されたり削除されたりする可能性はあります。

つまり、Fetchフェーズで見た世界と、Processフェーズで実際にアクセスする世界が完全には一致しないことがあります。

このため、この分割が向いているのは、**取得したメタデータをもとに後段処理してよい**ケースです。

例えば、次のようなケースでは比較的採用しやすいです。

- エクスポート時点の対象一覧さえ確定できればよい
- ダウンロード失敗時はリトライや部分失敗として扱える
- 厳密な最終一貫性より、全体スループットを優先したい

逆に、最後まで強い一貫性が必要なら、ユースケース全体を1トランザクションに寄せる選択もありえます。

### 強い一貫性が必要なら、全体を寄せる判断もある

例えば、途中で別トランザクションに見えてはいけない更新を伴う処理や、「取得した時点の状態が最後まで絶対に崩れてはいけない」処理なら、ユースケース全体をより強く1つの整合性境界として扱うべきです。

ここで重要なのは、「原則を守ること」そのものではありません。

**何を守るためのトランザクションなのか**を明確にしたうえで、必要なら広く取り、不要なら局所化することです。

## 設計判断としてどう捉えるか

この話は、「ユースケース単位のトランザクション」という原則を捨てよう、という話ではありません。

むしろ逆で、原則を知っているからこそ、「今回はどこまでを整合性境界にするべきか」を意識的に選べるようになります。

原則を丸暗記して当てはめる段階から、要件と処理特性を見て境界を調整できる段階に進むことが、設計の成熟だと思います。

特に、アプリケーション側でアクセスパターンや責務分担を意識して設計するシステムでは、トランザクション境界の引き方は実装詳細ではなく、かなり本質的な設計判断です。

## まとめ

トランザクション境界は、設計の本質的な判断ポイントです。

一律にユースケース単位にするのではなく、整合性要件と処理特性を見て決めるべきです。

特に、重い外部I/OやCPU処理を含むユースケースでは、DBの責務とアプリケーションの責務を分け、必要なメタデータだけを先に取得してトランザクションを早めに閉じる、という設計が有効な場面があります。

ユースケース全体を1トランザクションで囲うことも、局所化することも、どちらも選択肢です。

大事なのは、設計を決め打ちしないことです。

何を守りたいのか、どこにコストがかかるのか、外部I/Oをどこまで巻き込むのかを見て、状況に応じて境界を引く。

その積み重ねが、よりスケーラブルで説明可能な設計につながるはずです。

## 判断基準の目安

### こういうときはユースケース単位で囲う

- 最後まで強い一貫性を保ちたい
- 途中状態を他処理から見せたくない
- DB更新をまたぐ整合性保証が本質的に重要
- 外部I/Oや重いCPU処理がほとんどなく、短時間で閉じられる

### こういうときは局所化を検討する

- DB読み取りのあとに重い外部I/Oが続く
- ZIP圧縮などCPU負荷の高い処理がある
- `errgroup`などでアプリケーション層の並列化を活かしたい
- 必要なのは「取得時点のメタデータ確定」であり、後段はその結果を使って処理できる
- DB資源の占有時間を短くして、スループットやスケーラビリティを上げたい
