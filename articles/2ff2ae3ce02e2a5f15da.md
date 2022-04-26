---
title: "GoのS3 ダウンロード処理で知っておくと良いこと - バックエンドパフォーマンス改善"
emoji: "🍀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "s3", "io", "bufio", "sync"]
published: true
---

こんにちは、[@igsr5](https://twitter.com/igsr5_) です。普段はある高専の情報科に通いながら、[Wantedly, Inc.](https://www.wantedly.com/companies/wantedly) で長期インターンをしています。興味領域はフロント・バックエンド、インフラで、最近は業務でもっぱらGoを書いています。今回はGoのパフォーマンスチューニングの話です。

## 対象読者
- aws-sdk-go(aws-sdk-go-v2)[^1] で s3 ダウンロード処理のパフォーマンス改善を行いたい人
- Go[^2] の io パッケージの話に興味がある人
- バックエンドのパフォーマンス改善に興味がある人


## TL;DR

内部で s3 ダウンロードが行われるバックエンドAPI などを考えたとき、

```go
// 1. Downloader の作成
downloader := s3manager.NewDownloader(sess, func(d *s3manager.Downloader) {
        // + ここを追加
        d.BufferProvider = s3manager.NewPooledBufferedWriterReadFromProvider(5 * 1024 * 1024) // 5MB
})
```
- aws-sdk-go/s3manager で `Downloader.BufferProvider` に `PooledBufferdWriterReadFromProvider` を指定すると、
  - 平均的なダウンロード処理速度が2, 3倍早くなる
  - ダウンロード処理の全体的なレイテンシが安定する
- なぜ改善したか
  - ダウンロード処理時に `bufio` パッケージと `sync` パッケージの `Pool` によってメモリ空間を効率的に使い回すことができたため
- パフォーマンス改善を行う際は「何が、どういう状況でボトルネックになっているのか」「そのボトルネックに対してどういったアプローチを取れるのか」が大切


## 改善結果
こういったパフォーマンス改善の記事は具体的な改善結果を先に載せた方がテンションが上がると思うので初めに紹介します。赤い線の右側がパフォーマンス改善後の測定値です。

![](https://storage.googleapis.com/zenn-user-upload/dff7f164f1d3-20220425.png)
*内部でS3ダウンロード処理を行っているAPIのレイテンシ*

| | |
| :---: | :---: |
| ![](https://storage.googleapis.com/zenn-user-upload/39793afcaa74-20220425.png) *APIサーバ内のGCの呼び出し回数* | ![](https://storage.googleapis.com/zenn-user-upload/ed26739564e9-20220425.png) *APIサーバ内のGCにかかった時間* |

これらの画像は内部でS3から数100 KB ~ 数10 MB のファイルをダウンロードしているAPIサーバを [New Relic](https://newrelic.com/) で計測したものです。パフォーマンス改善地点からかなり処理が高速・安定になっていることが読み取れます。特にGC[^3]に関してはほとんど発生しなくなっています。

## 具体的にやったこと
具体的にコード上でやったことはとても単純です。

```go
import (
        "github.com/aws/aws-sdk-go/service/s3"
        "github.com/aws/aws-sdk-go/service/s3/s3manager"
)

sess, _ := session.NewSession()

// 1. Downloader の作成
downloader := s3manager.NewDownloader(sess)

// 2. 実際のダウンロード処理
buf := aws.NewWriteAtBuffer([]byte{})
downloader.Download(
        buf,
        &s3.GetObjectInput{
                Bucket: aws.String("my-bucket"),
                Key:    aws.String("image.png"),
        })
```
上記のような処理を考えたとき、`NewDownloader` の呼び出し時にこのような初期設定を与えてあげるだけです。

```go
// 1. Downloader の作成
downloader := s3manager.NewDownloader(sess, func(d *s3manager.Downloader) {
        // + ここを追加
        d.BufferProvider = s3manager.NewPooledBufferedWriterReadFromProvider(5 * 1024 * 1024) // 5MB
})
```
これだけで先ほど紹介したような改善結果が得られます。
詳しくは後述しますが、この `Downloader` の設定追加により s3 ダウンロードに使用するメモリ空間をsyncパッケージ[^4] (`sync.Pool`)で効率的に扱えるようになりました。

## なぜパフォーマンスが改善したのか
先ほどの改善結果で「レイテンシの高速化」「全体的なレイテンシの安定化(テイルレイテンシの改善)」が行われていることを示しました。ではなぜこのような結果が得られたのでしょうか。

### s3 ダウンロードに使用するメモリ空間を効率的に使い回せた
どちらの改善結果も一言で言うと `sync.Pool`を利用することで s3 ダウンロード時に使用するメモリ空間を効率的に使い回せるようになったことが挙げられます。

**レイテンシの高速化**
→ サーバ内でメモリ空間を効率的に使い回せると、処理毎のメモリのアロケーションを減らせる。

**全体的なレイテンシの安定化(テイルレイテンシの改善)**
→ サーバ内でメモリ空間を効率的に使い回せると、不要なメモリが生まれないためGCがほぼ発生しなくなる。

これらの要因がそれぞれのパフォーマンス改善につながっています。

:::message
先ほどから「 `sync.Pool` を利用することによってメモリを効率的に...」と書いていますが、ここで少しだけ `sync.Pool` の説明をしておきます。

syncパッケージは Go 標準パッケージの一つで非同期処理に関連する機能を提供します。その中でも `sync.Pool` は複数のgoroutine間で使えるバッファプールを実現できます。バッファプールを用いると、「事前にある程度のリソースを確保」「それ以降はそのリソースを取得・解放を繰り返して使い回す」ので基本的にリソースの調達に時間がかかる処理に対してはパフォーマンスの向上が期待できます。
例えば今回の記事だとs3ダウンロード毎に大きなサイズのメモリ割り当てが発生していたので、事前にメモリ割り当てを済ませておくことでパフォーマンスを改善させることができました。
:::

## 今回のボトルネック調査と内部実装
記事内容的にはここで終わっても良いのですがせっかくなのでもう少し詳しく今回のパフォーマンス改善について解説しようと思います。具体的には今回のボトルネック調査と対応の流れを io パッケージ[^5]内部の話を交えながら話します。

### ボトルネック調査
先ほどパフォーマンス改善の要因についてこう述べました。
> s3 ダウンロードに使用するメモリ空間を効率的に使い回せた           

この章では実際にプロファイラを仕込んで、ボトルネックをどう見つけたかを説明します。

:::details 利用したコード
エラー処理は省略しています。S3 ダウンロード処理をループさせながら pprof でプロファイルを取得できるようにしています。
  ```go
  package main

  import (
    "net/http"
    _ "net/http/pprof"

    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3"
    "github.com/aws/aws-sdk-go/service/s3/s3manager"
  )

  func main() {
    // pprof
    go func() {
      http.ListenAndServe("localhost:6060", nil)
    }()

    sess, _ := session.NewSession()
    downloader := s3manager.NewDownloader(sess)

    for {
      // NOTE 1.2 MB の画像.
      download(downloader, "my-bucket", "image.png")
    }
  }

  func download(downloader *s3manager.Downloader, bucket string, key string) {
    buf := aws.NewWriteAtBuffer([]byte{})

    downloader.Download(
      buf,
      &s3.GetObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(key),
      })
  }
  ```
:::

![](https://storage.googleapis.com/zenn-user-upload/83f760ff4241-20220425.png)
*1.2MBの画像をs3からダウンロードしたときのpprof出力*

プロファイル結果を見ると `s3manager.(*downloader).Download` の中でも`io.Copy`の実行に時間がかかっています。つまり`s3manager.Download` のパフォーマンスチューニングを行うためには`io.Copy` のパフォーマンスチューニングを行えば良いわけです。

### `io.Copy` の挙動
ここまでの話で S3 ダウンロードの処理性能を改善するためには `io.Copy` の処理性能を改善すれば良いことが分かりました。

ここで一旦 `io.Copy` の挙動をこの先の話に必要になるため説明しておきます。`io.Copy` のコードを実際に見てみましょう。

```go
// https://pkg.go.dev/io#Copy

func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}

func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	if buf != nil && len(buf) == 0 {
		panic("empty buffer in CopyBuffer")
	}
	return copyBuffer(dst, src, buf)
}

func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	if buf == nil {
		size := 32 * 1024
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
  // 省略...
  // io.Read & io.Write を行う
  }
}
```
コメントは一部省略しています。 `src io.Reader` → `dst io.Writer`  へのコピー処理の大まかな流れをまとめると下記のようになります。

1. `src io.Reader` が `io.WriteTo` 関数を持っていたらそれを呼び出して return する
2. `dst io.Writer` が `io.ReadFrom` 関数を持っていたらそれを呼び出して return する
3. `io.Read` , `io.Write` を用いて `src io.Reader` → `dst io.Writer`  へのコピー処理を行う
    - 実際には `io.Read` で一旦 `[]byte` にデータを読み込んで、その `[]byte` を `io.Write` で dst に書き込んでいる
    - これが `io.Copy` のデフォルトの処理

今回のパフォーマンス改善で大事なのは 1, 2 のステップです。何をしているかを一言で説明すると「デフォルトの `io.Copy` の挙動を他実装に置き換える」です(デフォルトの `io.Copy` の挙動とは 3 で説明したもの)。詳しくは後述しますが、これらの仕組みを使って`io.Copy`を自分の都合の良い挙動にすることができます。

:::message
https://pkg.go.dev/io#ReaderFrom
```go
type ReaderFrom interface {
	ReadFrom(r Reader) (n int64, err error)
}
```
`io.ReadFrom`とは`io.Reader`型を引数にとり、そのReaderからデータをEOFになるまで読み込む関数です。`io.Writer`インターフェースでこの関数を実装すると `io.Copy` で代わりにその関数を実行することができます。(`io.WriteTo`も `io.Reader`, `io.Writer`の関係が逆になるだけなので説明は割愛します。)
:::


### `io.Copy` のパフォーマンスチューニング
さて`io.Copy` のボトルネックについてもう少し具体的に考えてみましょう。先ほどのpprof出力で実行時間の大きいものから順に改善できそうな関数を見ていくと `runtime.makesilce()`が見つかります。
![](https://storage.googleapis.com/zenn-user-upload/64dfd567fe77-20220426.png)
*`runtime.makeslice()` - `s3manager.(*dlchunk).Write`の下にある*

ここでの `makeslice` は「`io.Copy` 内で使用するバッファのメモリ確保」にあたります。処理毎にメモリ確保を行うのは無駄なので出来れば`sync.Pool` 的な機能で効率的にメモリを使いまわしたいです。先ほど説明しましたが`io.Copy` ではそういった時に `ReadFrom` (or `WriteTo`) で挙動を上書きすることができるのでこの対応を考えていきましょう。

### aws-sdk-go と io パッケージ
`io.Copy` で利用する `io.ReadFrom`実装は自分で用意することもできますが、実は aws-sdk-go の s3manager パッケージには Download 呼び出し時の `io.Copy` に `ReadFrom` を指定するための仕組みと `sync.Pool` でバッファのメモリ空間を効率的に扱う`ReadFrom`実装が既に用意されています。

#### Download 呼び出し時の `io.Copy` に `ReadFrom` を指定できる仕組み
下のコードは s3manager.Downloader の struct 定義です。**`BufferProvider` が Download 時に呼び出される `io.Copy` に `ReadFrom` を指定できる仕組みに相当します。**

```go
// https://github.com/aws/aws-sdk-go/blob/v1.43.45/service/s3/s3manager/download.go#L43-L73

type Downloader struct {
    // 省略...

    // Defines the buffer strategy used when downloading a part.
    //
    // If a WriterReadFromProvider is given the Download manager
    // will pass the io.WriterAt of the Download request to the provider
    // and will use the returned WriterReadFrom from the provider as the
    // destination writer when copying from http response body.
    BufferProvider WriterReadFromProvider
}
```

`BufferProvider` の型情報である `WriterReadFromProvider` の interface 定義です。
```go
// https://github.com/aws/aws-sdk-go/blob/v1.43.45/service/s3/s3manager/writer_read_from.go#L17-L20

// WriterReadFrom defines an interface implementing io.Writer and io.ReaderFrom
type WriterReadFrom interface {
	io.Writer
	io.ReaderFrom
}

// WriterReadFromProvider provides an implementation of io.ReadFrom for the given io.Writer
type WriterReadFromProvider interface {
	GetReadFrom(writer io.Writer) (w WriterReadFrom, cleanup func())
}
```

`WriterReadFromProvider.GetReadFrom()`が返す`WriterReadFrom`の実装が`io.Copy`時の `io.Writer`として使われるようになります。


#### `sync.Pool` でバッファのメモリ空間を効率的に扱う`ReadFrom`実装
正確には前節で説明した `WriterReadFromProvider` 実装です。下が実際のコードです。

```go
// https://github.com/aws/aws-sdk-go/blob/v1.43.45/service/s3/s3manager/writer_read_from.go

type PooledBufferedReadFromProvider struct {
	pool sync.Pool
}

func NewPooledBufferedWriterReadFromProvider(size int) *PooledBufferedReadFromProvider {
	if size < int(32*sdkio.KibiByte) {
		size = int(64 * sdkio.KibiByte)
	}

	return &PooledBufferedReadFromProvider {
		pool: sync.Pool{
			New: func() interface{} {
				return &bufferedReadFrom{bufferedWriter: bufio.NewWriterSize(nil, size)}
			},
		},
	}
}

func (p *PooledBufferedReadFromProvider) GetReadFrom(writer io.Writer) (r WriterReadFrom, cleanup func()) {
	buffer := p.pool.Get().(*bufferedReadFrom)
	buffer.Reset(writer)
	r = buffer
	cleanup = func() {
		buffer.Reset(nil) // Reset to nil writer to release reference
		p.pool.Put(buffer)
	}
	return r, cleanup
}
```

:::details bufferedReadFrom の実装

```go
// https://github.com/aws/aws-sdk-go/blob/v1.43.45/service/s3/s3manager/writer_read_from.go#L11-L39

// WriterReadFrom defines an interface implementing io.Writer and io.ReaderFrom
type WriterReadFrom interface {
	io.Writer
	io.ReaderFrom
}

// WriterReadFromProvider provides an implementation of io.ReadFrom for the given io.Writer
type WriterReadFromProvider interface {
	GetReadFrom(writer io.Writer) (w WriterReadFrom, cleanup func())
}

type bufferedWriter interface {
	WriterReadFrom
	Flush() error
	Reset(io.Writer)
}

type bufferedReadFrom struct {
	bufferedWriter
}

func (b *bufferedReadFrom) ReadFrom(r io.Reader) (int64, error) {
	n, err := b.bufferedWriter.ReadFrom(r)
	if flushErr := b.Flush(); flushErr != nil && err == nil {
		err = flushErr
	}
	return n, err
}
```

:::

`NewPooledBufferedWriterReadFromProvider` の中で `sync.Pool{New: ...}` によって`bufferedReadFrom`をプールに格納しているのがポイントです。`GetReadFrom` では `WriterReadFrom` の値を毎回生成するのではなく`bufferedReadFrom`をプールから取り出して返すことでメモリのアロケーションを抑えています。

### `PooledBufferedWriterReadFromProvider` 利用後の pprof

前章で aws-sdk-go/s3manager の `Downloader.BufferProvider` と `NewPooledBufferedWriterReadFromProvider` を利用すれば、s3ダウンロード処理時に`io.Copy`で利用するメモリを`sync.Pool`で使いまわせることを説明しました。では実際にそれらの関数を利用して再度pprofで測定してみましょう。

:::details 利用したコード
エラー処理は省略しています。S3 ダウンロード処理をループさせながら pprof でプロファイルを取得できるようにしています。
  ```go
  package main

  import (
    "net/http"
    _ "net/http/pprof"

    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3"
    "github.com/aws/aws-sdk-go/service/s3/s3manager"
  )

  func main() {
    // pprof
    go func() {
      http.ListenAndServe("localhost:6060", nil)
    }()

    sess, _ := session.NewSession()
    downloader := s3manager.NewDownloader(sess, func(d *s3manager.Downloader) {
      d.BufferProvider = s3manager.NewPooledBufferedWriterReadFromProvider(5 * 1024 * 1024)
    })

    for {
      // NOTE 1.2 MB の画像.
      download(downloader, "my-bucket", "image.png")
    }
  }

  func download(downloader *s3manager.Downloader, bucket string, key string) {
    buf := aws.NewWriteAtBuffer([]byte{})

    downloader.Download(
      buf,
      &s3.GetObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(key),
      })
  }
  ```
:::

![](https://storage.googleapis.com/zenn-user-upload/cbe940889f1b-20220425.png)
*パフォーマンス改善後 - 1.2MBの画像をs3からダウンロードしたときのpprof出力*

`io.Copy`の内部で`s3manager.(*bufferedReadFrom).ReadFrom`が実行されていることが確認できます。`makeslice`の表示はなくなりDownload全体の実行時間は **1.37 s** から **510 ms** にまで改善しています。(実際にはメモリのアロケーションが最適化されただけでなく bufio パッケージの `ReadFrom`実装が利用されるようになったのもパフォーマンス改善につながっていそうですが、、)

また、パフォーマンス改善後に本番環境で取得したメトリクスからも GC の発生が極端に減少しているのでメモリの利用を最適化できたと言えそうです。(記事冒頭の改善結果より)

## まとめ
この記事ではGoでs3ダウンロード処理のパフォーマンス改善を行う方法とボトルネック調査の過程を紹介しました。かなりお手軽にダウンロード処理の性能を上げることができたのでs3ダウンロード関連でパフォーマンス改善を行いたい人はやってみるといいかもしれません。

しかし実際にパフォーマンス改善を行う際は **「何が、どういう状況でボトルネックになっているのか」「そのボトルネックに対してどういったアプローチを取れるのか」が大切**です。今回の記事の内容で言うと、私たちが施策をおこなったAPIサーバが重たいファイルをS3から頻繁にダウンロードしていたため、ダウンロード処理時のメモリ最適化で処理性能を改善することができました。

置かれている状況が違えば、ボトルネックやその対応も変わってくるはずなのでシステム監視ツールやpprof、ベンチマークなどで適切に計測しながら施策を打っていくのが良いでしょう。


## 参照
- https://levyeran.medium.com/high-memory-allocations-and-gc-cycles-while-downloading-large-s3-objects-using-the-aws-sdk-for-go-e776a136c5d0
- https://pkg.go.dev/io
- https://pkg.go.dev/bufio
- https://pkg.go.dev/sync

[^1]: この記事で説明する内容は aws-sdk-go, aws-sdk-go-v2 の両方で動作します。
[^2]: この記事で紹介する検証コードは golang:1.17 で書かれています。
[^3]: GC = Garbage collection
[^4]: sync = Go標準パッケージ。非同期処理に関連する機能を提供する。https://pkg.go.dev/sync#Pool
[^5]: io = Go標準パッケージ。I/Oに関連する機能を提供する。ただしファイルなどの特定のI/Oに限らない抽象的な機能。https://pkg.go.dev/io
