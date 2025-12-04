---
title: "docker-buildのチューニングTips全部書く【Go×GitHubActions】"
date: 2025-12-04T12:16:21+09:00
image: "thumbnail.jpg"
---

docker の最新記法や、GitHubActions でのビルドチューニングについて網羅的に書かれている記事って意外と少ない！

主に Go\*GitHubActions での image ビルドについて、実行時間を短く・レイヤーを小さくする Tips を共有します 🚀

<!--more-->

## はじめに

[QualiArts Advent Calendar 2025](https://qiita.com/advent-calendar/2025/qualiarts) の 4 日目の記事です！

最近 Go のビルド周りを改善するのが趣味で、情報が結構散らばっていて困った経験があるのでまとめます。

皆様の docker-build 改善のきっかけになれるような記事を目指します！どれか刺され！🔥

**目次**

{{< toc >}}

## 1. .dockerignore を適切に設定する

まずどのように docker がファイルを扱うのかを確認しましょう。

`docker build`時、docker デーモンに指定ディレクトリ配下のファイルを tar アーカイブとして送信し、これがビルド時に`COPY`や`ADD`経由でアクセスできるファイル群（[BuildContext](https://docs.docker.com/build/concepts/context/)）になります。

{{< figure src="https://docs.docker.com/get-started/images/docker-architecture.webp" class="left" width="300" >}}

> [引用](https://docs.docker.com/get-started/docker-overview/#docker-architecture)

そして、**[.dockerignore](https://docs.docker.com/build/concepts/context/#dockerignore-files)はビルドコンテキストの対象から外すディレクトリ/ファイル一覧を設定できる**仕組みです。これを適切に設定することにより、docker デーモンに転送するファイルサイズが減るのはもちろん、レイヤーのサイズも削減できます。

特に重い`.git`ファイルや、本番で使うことのない**テスト・ドキュメントファイル**などは特別必要ないのであれば**除外しておきたい**ところですね 🙆

```t {style=emacs, class="small-font75"}
.*
# 再起的にファイルを除外するには*.mdだけでは足りず、**/指定が必要なので注意
**/*.md
**/*_test.go
```

これを活用して [infracost](https://www.infracost.io/) という terraform からコスト差分を検出できる CLI へ OSS コミットできたりしました 👏

{{< figure src="infracost2.jpg" class="left" width="300" >}}

cf. https://github.com/infracost/infracost/pull/3480

### 1-1. レイヤーごとのファイル状況を目視する「dive」

[dive](https://github.com/wagoodman/dive)はレイヤーごとに image のファイル状況を可視化できる神ツールです。

上記 PR も`dive ${image_tag}`の結果を参考にしたもので、`COPY . .`で本来必要のない`testdata`ディレクトリが余計に容量を圧迫していることが分かりますね。

{{< figure src="infracost3.jpg" class="left" width="800" >}}

定期的に image の中身を dive で確認し、必要ないものが入っていないか精査することをお勧めします 🙆

### 1-2. BuildKit による自動 ignore

「**私の`.dockerignore`薄すぎ...？**」と思った人も大丈夫。救済があります。

[**BuildKit**](https://docs.docker.com/build/buildkit/)は[DockerBuild](https://docs.docker.com/build/)の拡張ツールで、現在は本体のデフォルト機能として取り込まれています。BuildKit には、**明示的にアクセスがないファイルは自動的にビルドコンテキストの対象から除外する**機能が導入されています 👏

> Detect and skip transferring unused files in your build context

例えば`COPY dir_A .`のみの Dockerfile なら、`dir_A`以外のディレクトリは BuildContext に送信しないようにしてくれる感じですね！私たちは恵まれた時代に生きています。

一方、`dir_A`の**中にある不要ファイルの除外は行ってくれません**し、`COPY . .`と記述していたら問答無用で全て転送されてしまいます。

BuildContext から取得するファイルの範囲を**必要最小限に記述**した上で、**`.dockerignore`も適切に設定**するのが好ましいでしょう。

## 2. マルチステージビルドを行う

正直耳タコな話ですが一応記述します。

参照時は最終ステージのみが配信されるため、ビルド環境の配信環境はステージを切り分け、**成果物だけ最終ステージに残すことでイメージサイズを落とす**テクニックでしたね。

[公式の例示](https://docs.docker.com/build/building/multi-stage/)では以下のような結果となりました。

```diff {style=emacs,linenos=inline, class="small-font50"}
FROM golang:1.24
WORKDIR /src
COPY <<EOF ./main.go
package main

import "fmt"

func main() {
  fmt.Println("hello, world")
}
EOF
RUN go build -o /bin/hello ./main.go

+ FROM scratch
+ COPY --from=0 /bin/hello /bin/hello
CMD ["/bin/hello"]
```

```diff {style=emacs, class="small-font75"}
$ docker images hoge --format "{{.Size}}"
- // 1.37GB
+ // 3.47MB
```

ちなみに、[**BuildKit**](https://docs.docker.com/build/buildkit/)には各ステージを並列ビルドしてくれる機能も備わっています。

> Parallelize building independent build stages

弊プロジェクトはかつて [kaniko](https://github.com/GoogleContainerTools/kaniko)(2025/06 にアーカイブ)を使っており、これは[ステージごとの並列ビルドに対応していなかった](https://github.com/GoogleContainerTools/kaniko/issues/3444)ため、乗り換えるだけでビルド時間短縮の恩恵が得られました。

## 3. レイヤーキャッシュがヒットしやすい構成にする

耳タコな話が続きますが、まずは**レイヤーキャッシュ**についてのおさらいからです。

Dockerfile 内の各命令（`RUN`、`COPY`など）はそれぞれ独立した「レイヤー」を作成し、ファイルシステムの変更差分（diff）として積み重ねられていく構造になっています。

ビルド実行時、Docker は各レイヤーに対して、命令とファイル内容のチェックサムが同じか判定し、同じならそのレイヤーは再実行されずキャッシュから取得される。
**キャッシュが無効なレイヤーが挟まるとそれ以降のレイヤーは全て再実行される**点が重要です。

{{< figure src="https://docs.docker.com/build/images/cache-stack-invalidated.png" class="left" width="500" >}}

> [引用](https://docs.docker.com/build/cache/#how-the-build-cache-works)

よって、変更の少ない && 実行時間がかかる処理を前半に記述し、変更の多い処理を後半に記述するのが鉄則になります。

Go 言語であれば、モジュールのダウンロードを先んじて切り出すことで、アプリケーションロジックのみの変更があった際、モジュールの DL レイヤーをスキップできます。

```diff {style=emacs, class="small-font75"}
+ COPY go.mod go.sum .
+ RUN go mod download
  COPY . .
  RUN go build main.go
```

cf. https://docs.docker.com/build/cache/optimize/#order-your-layers

### 3-1. pnpm におけるレイヤーキャッシュ活用

蛇足ですが、pnpm の場合も同じようなことが言えます。

`package.json`ファイルは変更されたが`pnpm-lock`ファイルが変更されない(依存関係が変わらない)場合に、依存のダウンロードをスキップして`store`ディレクトリの`cache`を用いることができます。

```diff {style=emacs, class="small-font75"}
+ COPY pnpm-lock.yaml target=pnpm-lock.yaml .
+ RUN pnpm fetch --frozen-lockfile
+ COPY package.json .
+ RUN pnpm install --offline --frozen-lockfile
  COPY . .
  RUN pnpm build:hoge
```

`pnpm fetch`、最近知りました。便利です。

cf. https://pnpm.io/ja/cli/fetch

### 3-2. 中間レイヤーの確認と tar 圧縮について

ちなみに、中間レイヤーのサイズや MediaType は[crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane)というツールを使うことで確認できます。

```sh {style=emacs, class="small-font50"}
$ crane manifest --platform linux/amd64 nginx:1.29.3-alpine
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:d4918ca78576a537caa7b0c043051c8efc1796de33fee8724ee0fff4a1cabed9",
    "size": 10963
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:2d35ebdb57d9971fea0cac1582aa78935adf8058b2cc32db163c98822e5dfa1b",
      "size": 3802452
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:8f6a6833e95d43ac524f1f9c5e7c1316c1f3b8e7ae5ba3db4e54b0c5b910e80a",
      "size": 1835502
    },
	...
```

ここから、この image の中間レイヤーが Gzip で圧縮されていることが分かったりします（後で使うから覚えておいて！）。

cf. https://github.com/opencontainers/image-spec/blob/v1.1.0/layer.md#gzip-media-types

## 4. Go バイナリのシンボルテーブル・デバッグ情報を削除する

少し寄り道して、Go バイナリを小さくする話もします。

`go build`時に **`ldflags`で`-s`を指定するとシンボルテーブルなどのデバッグ情報が削除**されバイナリサイズを削減できます。
ビルド環境のローカルパス情報を隠蔽する`-trimpath`と並び、Gopher なら 2 億回目にするやつですね

Hello World で試すと以下のような差分になりました。

```diff {style=emacs, class="small-font75"}
- go build main.go && du -h main // 2.3M
+ go build -trimpath -ldflags="-s" main.go && du -h main // 1.5M
```

ちなみに、昔はよく`-s -w`という記法を見ましたが、[**Go1.22 からは-s のみで完結**](https://github.com/golang/go/commit/ba1deb1ceef956bdb3ca5a9570f132cf19ccc9f6)するようになっています。

このような昔からある話でも意外と OSS commit チャンスがあったりするのでおすすめです 🎉

{{< figure src="infracost1.jpg" class="left" width="300" >}}
cf. https://github.com/infracost/infracost/pull/3479

### 4-1. UPX によるバイナリ圧縮とレイヤー圧縮

さらに脱線して、Go バイナリの圧縮について検討してみましょう。

[UPX](https://github.com/upx/upx)は実行ファイルを圧縮するツールです。
圧縮されたファイルは自身の展開プログラムもバイナリに含むため、ユーザーが外から展開処理を記述する必要がなく非常にスマートです。

Hello World のバイナリサイズは UPX を噛ませることでこんなに小さくなります。

```diff {style=emacs, class="small-font75"}
- go build -trimpath -ldflags="-s" main.go && du -h main
- // 1.5M
+ go build -trimpath -ldflags="-s" main.go && upx -q --force-macos main && du -h main
+ // 676K
```

さて、**UPX を Dockerfile 内で行う必要があるのか**という議論が面白のいです。

先述した通り、**レイヤーは通常 tar.gz （Gzip）で圧縮**されるため、部分的に **2 重圧縮となり効率が悪くなる**ケースが存在します。
また、そもそも自身の展開プログラムが余分にバイナリに入っていたりする影響で、**思うように image サイズが下がらない**場合がままあるようです。

かなりハッキーなので、一旦は UPX は使わないでよさそうという結論に着地しましょう。

#### スーパー面白記事紹介コーナー

以下記事がとても勉強になりました、ありがとうございました。 [@knqyf263](https://x.com/knqyf263)

cf. [コンテナイメージ内の実行ファイルを upx で圧縮するべきか | フューチャー技術ブログ](https://future-architect.github.io/articles/20210520b/)

## 5. Bind mounts でレイヤーをスリムにたもつ

さて、Docker に話を戻しましょう。

今までは`COPY`命令で必要なファイルを BuildContext に移植した上でビルドしていましたが、本質的にはレイヤーの成果物として欲しいのはビルドの成果物だけで、それに必要なファイル群はビルドが終わってしまえばレイヤーに保持しておく必要ありません。

[**Bind mounts**](https://docs.docker.com/engine/storage/bind-mounts/)は、BuildContext のファイルを命令の間だけ image にマウントする機能です。

```diff {style=emacs, class="small-font75"}
- COPY main.go .
- RUN go build -o /bin/hello ./main.go
+ RUN --mount=source=main.go,target=main.go \
+  	go build -o /bin/hello ./main.go
```

右が Bind mounts を活用した image です。

ビルドのみに必要な`main.go`というファイル が中間レイヤーに入り込まないことを確認できます 👏
{{< figure src="bind-mount.jpg" class="left" width="700" >}}

たとえ最終ステージでなくても、レイヤーのサイズは小さいに越したことはないです。

ランタイムでファイルの参照があるなど特別な理由がない限り、Go 言語では **`COPY`や`ADD`を使わなくていい時代**になったのは少しだけ大きな転換点でしたね 🚀

## 6. Cache mounts でレイヤーを跨いでキャッシュする

次に、**レイヤーキャッシュが無効な状態でもビルドキャッシュなどを個別に活用**する方法についてです。

[**Cache mounts**](https://docs.docker.com/build/cache/optimize/#use-cache-mounts)は、ビルドを超えたキャッシュ置き場フォルダを指定する機能です。たとえレイヤーキャッシュが効かなくても、2 回目以降のビルドであれば前回の module/build キャッシュを活用することができます。

```diff {style=emacs, class="small-font75"}
+ RUN --mount=type=cache,target=/go/pkg/mod \
      --mount=source=go.mod,target=go.mod \
      --mount=source=go.sum,target=go.sum \
      go mod download

+ RUN --mount=type=cache,target=/root/.cache/go-build \
+     --mount=type=cache,target=/go/pkg/mod \
      --mount=source=go.mod,target=go.mod \
      --mount=source=go.sum,target=go.sum \
      --mount=source=cmd/api,target=cmd/api \
      CGO_ENABLED=0 go build -o api -trimpath -ldflags '-s -w' cmd/api/main.go
```

私が所属するプロジェクトのある image では、**Bind/Cache mounts を導入したことでビルド時間が大幅に改善**しました。

{{< figure src="cache-mount.jpg" class="left" width="500" >}}

ちなみに、[BuildKit はデフォルトで Cache mounts のキャッシュを GitHubActions で使うことができない](https://docs.docker.com/build/ci/github-actions/cache/#cache-mounts)ため、CI 環境では専用の action「[buildkit-cache-dance](https://github.com/reproducible-containers/buildkit-cache-dance)」を使う必要があることに注意が必要です。

#### スーパー面白記事紹介コーナー

とても勉強になりました、ありがとうございました。[@shibu_jp](https://x.com/shibu_jp)

cf. [2024 年版の Dockerfile の考え方＆書き方 | フューチャー技術ブログ](https://future-architect.github.io/articles/20240726a/)

## 7. distroless などの軽量イメージ を base-image に選択する

次に、そもそものベースイメージのサイズを小さくします。

[**distroless**](https://github.com/GoogleContainerTools/distroless) は Google が提供している必要最小限の依存のみが含まれる image で、alpine と比較しても軽量且つセキュアであることが特徴です。内容物は以下の通りです。

- [gcr.io/distroless/static](https://github.com/GoogleContainerTools/distroless/blob/8795dab120a20242db8a408c886a41f453ae5dde/base/README.md?plain=1#L7-L12)
  - ca-certificates
  - A /etc/passwd entry for a root user
  - A /tmp directory
  - tzdata
- [gcr.io/distroless/base](https://github.com/GoogleContainerTools/distroless/blob/8795dab120a20242db8a408c886a41f453ae5dde/base/README.md?plain=1#L19-L24)
  - distroless/static の内訳
  - glibc
  - libssl

とりあえず`CGO_ENABLED=0`で`distroless/static`に載せてビルドしてみて、ダメだったら base や他の選択肢考える心持ちで良きだと思います 🙆

```diff {style=emacs, class="small-font50"}
FROM golang:1.25.5-alpine3.21 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o api ./main.go

- FROM alpine:3.22.2
+ FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /usr/src/
COPY --from=builder /src/api .
CMD ["/usr/src/api"]
```

単純な http サーバーで検証したところ、以下のようなサイズ削減ができました 👏

```diff {style=emacs, class="small-font75"}
$ docker images hoge --format "{{.Size}}"
- 25MB
+ 17.9MB
```

## 8. BuildKit の圧縮を Gzip → zstd に変更する

次に、レイヤーの圧縮手法についてです。
先述した通り、**通常レイヤーは Gzip 圧縮されますが、実は zstd で圧縮することもできます**。

圧縮レベルにもよりますが、zstd は Gzip に比べ比較的圧縮率が高く、必要時間も短い、展開速度も同程度ということで、乗り換えることによるデメリットはあまりなさそうでした。

[docker/build-push-action](https://github.com/docker/build-push-action)は GitHubActions で docker-build する際のデファクトスタンダードで、こちらで検証してみましょう。

```diff {style=emacs, class="small-font50"}
  - name: Build and push Docker image
    uses: docker/build-push-action@263435318d21b8e681c14492fe198d362a7d2c83 # v6.18.0
    with:
      context: .
      file: ${{ needs.init.outputs.docker_file }}
-     push: true
+     outputs: type=registry,oci-mediatypes=true,compression=zstd,compression-level=3,force-compression=true
      tags: ${{ inputs.location }}-docker.pkg.dev/${{ env.GOOGLE_CLOUD_PROJECT }}/${{ inputs.repository_name }}/${{ inputs.target }}:${{ needs.init.outputs.image_tag_name }}
      platforms: linux/amd64
      cache-from: |
        type=registry,ref=${{ inputs.location }}-docker.pkg.dev/${{ env.GOOGLE_CLOUD_PROJECT }}/${{ inputs.repository_name }}/${{ inputs.target }}/cache:latest
      cache-to: |
-       type=registry,ref=${{ inputs.location }}-docker.pkg.dev/${{ env.GOOGLE_CLOUD_PROJECT }}/${{ inputs.repository_name }}/${{ inputs.target }}/cache:latest,mode=max
+       type=registry,ref=${{ inputs.location }}-docker.pkg.dev/${{ env.GOOGLE_CLOUD_PROJECT }}/${{ inputs.repository_name }}/${{ inputs.target }}/cache:latest,mode=max,oci-mediatypes=true,compression=zstd,compression-level=3,force-compression=true
      provenance: false
      sbom: false
```

> oci-mediatypes=true

[OCI 準拠](https://github.com/opencontainers/image-spec/blob/v1.1.0/layer.md#zstd-media-types)でビルドする。

> force-compression=true

中間レイヤーも強制的に圧縮する

> compression-level=3

[AWS-fagate のベストプラクティス](https://aws.amazon.com/jp/blogs/news/reducing-aws-fargate-startup-times-with-zstd-compressed-container-images/)に準拠した圧縮レベルを採用。

```txt {style=emacs, class="small-font50"}
compression-level=3 – zstd には 22 段階の圧縮レベルがあります。圧縮レベルが高いほど、コンテナイメージのサイズは小さくなりますが、イメージレイヤーを解凍するための CPU リソースも増加します。AWS Fargate の起動時間を短縮するには、ワークロードの開始前にイメージレイヤーをダウンロードして解凍する必要がありますが、最高レベルの圧縮が最速の AWS Fargate の起動時間をもたらすとは限りません。最適な圧縮レベルを見つけるために、自身のコンテナイメージで検証すると良いでしょう。私たちのテストでは、圧縮レベル 3 が最適でした。
```

私の所属するプロジェクトのとある image では、この**圧縮方法の変更だけで 20%サイズが削減**されました 🚀

{{< figure src="zstd1.jpg" class="left" width="300" >}}

また、ArtifactRegisty の画面から中間レイヤーが OCI 準拠で zstd されていることを確認できました。

{{< figure src="zstd2.jpg" class="left" width="300" >}}

## 9. GitHubActions のランナーを強化する（namespace.so）

最後に、**実行環境を強くする**というシンプルだけど最も効果的な方法です 🚀

[GitHub ホステッドランナー](https://docs.github.com/ja/actions/concepts/runners/github-hosted-runners)の性能をあげてもいいし、自前で CloudRunWorkerPool などで[セルフホステッドランナー](https://docs.github.com/ja/actions/concepts/runners/self-hosted-runners)を立ててもいいでしょう。

今回は、お得でハイスペックなセルフホステッドランナーを SaaS で建てられる[**namespace.so**](https://namespace.so/)を紹介します。

namespace の最も素晴らしい機能は[**Cache Volumes**](https://namespace.so/docs/architecture/storage/cache-volumes)です。
**キャッシュデータをネットワーク越しにアップロード・ダウンロードするのではなく、ボリュームを物理的にアタッチ**することで、キャッシュの I/O による遅延が大幅に改善されます。

namspace 専用 action に置き換えると以下になります。**cache の registry 設定が必要なくなっている**ことに注目です。

```diff {style=emacs, class="small-font50"}
  - name: Set up Docker Buildx
-   uses: docker/setup-buildx-action@e468171a9de216ec08956ac3ada2f0791b6bd435 # v3.11.1
+   uses: namespacelabs/nscloud-setup-buildx-action@7020d7d8e659afecbfec162ab4693c7e56278311 # v0.0.19

  # (中略)

  - name: Build and push Docker image
    uses: docker/build-push-action@263435318d21b8e681c14492fe198d362a7d2c83 # v6.18.0
    with:
      context: .
      file: ${{ needs.init.outputs.docker_file }}
      outputs: type=registry,oci-mediatypes=true,compression=zstd,compression-level=3,force-compression=true
      tags: ${{ inputs.location }}-docker.pkg.dev/${{ env.GOOGLE_CLOUD_PROJECT }}/${{ inputs.repository_name }}/${{ inputs.target }}:${{ needs.init.outputs.image_tag_name }}
      platforms: linux/amd64
-     cache-from: |
-       type=registry,ref=${{ inputs.location }}-docker.pkg.dev/${{ env.GOOGLE_CLOUD_PROJECT }}/${{ inputs.repository_name }}/${{ inputs.target }}/cache:latest
-     cache-to: |
-       type=registry,ref=${{ inputs.location }}-docker.pkg.dev/${{ env.GOOGLE_CLOUD_PROJECT }}/${{ inputs.repository_name }}/${{ inputs.target }}/cache:latest,mode=max,oci-mediatypes=true,compression=zstd,compression-level=3,force-compression=true
      provenance: false
      sbom: false
```

弊プロジェクトでは、この移行で CI の実行時間が訳半分に減少しました 👏

cf. https://namespace.so/docs/solutions/github-actions/docker-builds#skip-github-actions-caching

セルフホステッドランナーの SaaS は [blacksmith](https://www.blacksmith.sh/)なども候補ですかね。
namespace 程の最適化があるのかは存じ上げませんが、ぜひ色々検討してみてください 🙆

年間契約などで性能が上がったのにお得になるみたいな減少が起こるかもしれません ☺️

## 10. その他細かいやつと hadolint

他にもレイヤーサイズの削減として、[apt のキャッシュを削除したり](https://github.com/infracost/infracost/pull/3481)

```diff {style=emacs, class="small-font50"}
- RUN apt-get update -q && apt-get -y install unzip
+ RUN apt-get update -q && apt-get -y install unzip && rm -rf /var/lib/apt/lists/*
```

apk の cache を削除したり

```diff {style=emacs, class="small-font50"}
- RUN apk add gcc
+ RUN apk --no-cache add gcc
```

色々なテクニックがありそうですね。

Dockerfile の品質を保つためにも、定期的に[hadolint](https://github.com/hadolint/hadolint)で静的解析を行うことをお勧めします 🙆

## まとめ

いかがだったでしょうか？

数字がちゃんと出るパフォーマンスチューニングって楽しいですよね！🚀

このブログをきっかけに、皆様のプロジェクトの改善のきっかけになれば幸いです。

ではまた！
