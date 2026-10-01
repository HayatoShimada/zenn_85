---
title: "microCMS をやめて、Raspberry Pi で Payload CMS を動かす（Tailscale でログイン、R2 に書き出し・バックアップ）"
emoji: "🗂️"
type: "tech"
topics: ["payloadcms", "nextjs", "tailscale", "cloudflare", "raspberrypi"]
published: true
---

古着屋のサイト（Next.js 16 / Vercel）のブログとバナーを、microCMS から**自宅の Raspberry Pi で動かす Payload CMS** に移しました。

結論から言うと、ポイントは「**サイトは Pi を一切読まない**」ことです。CMS は公開のたびに中身を JSON にして Cloudflare R2 に書き出し、サイトはその JSON だけを読みます。Pi が止まっても困るのは「編集できない」ことだけで、サイトの表示もビルドも影響を受けません。ログインは Tailscale のアカウントで行い、パスワードは持ちません。DB は Litestream で R2 に随時バックアップしています。

## なぜやめたか

microCMS 自体に不満があったわけではなく、店の使い方に合わなくなってきました。

- 本文の書き方が不自由（写真を横に並べたい、埋め込みを増やしたい）
- 無料枠では API や項目を自由に増やせない
- 画像の管理（整理・サイズ違いの用意）が手作業
- スマホから書きにくく、スタッフにも書いてほしい

このうち「項目を増やしたい」「本文に独自の部品を足したい」は、スキーマをコードで持てる CMS なら解決します。そこで、TypeScript でコレクションを定義する Payload を選びました。Payload 3 は Next.js 16 に正式対応していて、管理画面ごと Next.js アプリとして動きます。

## 前提

- サイトは Next.js 16（App Router / Cache Components）で Vercel にデプロイ。データ取得は `"use cache"` + `cacheTag` で、CMS からの通知で `revalidateTag` する
- 記事 38 件、バナー 4 件、画像 108 枚。小さなサイトです
- サーバーは Raspberry Pi 5（8GB、SD カード）。すでに Tailscale と Docker が入っていて、ほかのサービスも動いている
- ドメインの DNS は Cloudflare

## 構成

```
[メンバーのスマホ・PC（tailnet 内）]
   │ https://cms.<tailnet>.ts.net
   ▼
[Raspberry Pi: docker compose]
   ├─ tailscale   tailnet に「cms」として参加。HTTPS で payload に転送し、ログインした人のメールを渡す
   ├─ payload     管理画面（ホストにポートを出さない）
   └─ litestream  SQLite を R2 の非公開バケットへ随時バックアップ
        │ 公開・更新・削除のたびに
        ▼
[R2（media.<ドメイン>）]
   ├─ media/     画像（元画像と avif / webp の各サイズ）
   └─ content/   公開中の記事・バナーの JSON
        │ 書き出したら /api/revalidate を呼ぶ（HMAC 署名）
        ▼
[サイト（Vercel）] ビルド時と再検証時に R2 の JSON を読む
```

「ローカルサーバーで CMS を動かす」と聞くと、サイトから Pi に API を取りに行く構成を想像しがちです。でもそうすると、Pi をインターネットに出す必要があり、Pi が落ちた瞬間にビルドも再検証も失敗します。

ブログはほぼ静的なので、**読むのはビルド時と更新時だけ**です。それなら、更新したときに CMS 側から「完成品」を置いておけば十分です。Pi はインターネットに一切公開していません（Cloudflare Tunnel も Funnel も使っていません）。

## ログイン：Tailscale のアカウントで入る

`tailscale serve` は、tailnet 内のユーザーからのリクエストに `Tailscale-User-Login`（ログイン名＝メール）を付けて転送します。これを Payload のカスタム認証で受け取ります。

```ts
export const tailscaleStrategy: AuthStrategy = {
  name: 'tailscale',
  authenticate: async ({ headers, payload }) => {
    const login = headers.get('tailscale-user-login')?.trim().toLowerCase()
    if (!login) return { user: null }
    const { docs } = await payload.find({
      collection: 'users',
      where: { email: { equals: login } },
      limit: 1,
      overrideAccess: true,
    })
    return { user: docs[0] ? { ...docs[0], collection: 'users' } : null }
  },
}
```

`users` コレクションは `auth: { disableLocalStrategy: true, strategies: [tailscaleStrategy] }` にして、パスワードを持たせません。「メンバー」に登録したメールの人だけが入れて、権限は管理者と編集者の 2 つです。最初の 1 人だけは、環境変数に書いたメールなら自動で管理者として登録されるようにしました。

ヘッダーで認証する以上、**ヘッダーを偽装できない経路**にしておく必要があります。docker compose で payload を tailscale のサイドカーと同じネットワーク名前空間に入れ、`127.0.0.1` でだけ待ち受けます。

```yaml
services:
  tailscale:
    image: tailscale/tailscale:stable
    hostname: cms
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
      - TS_EXTRA_ARGS=--advertise-tags=tag:cms
    # …
  payload:
    build: .
    network_mode: service:tailscale   # ポートはホストに出さない
    environment:
      - HOSTNAME=127.0.0.1
```

```json:tailscale/serve.json
{
  "TCP": { "443": { "HTTPS": true } },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": { "/": { "Proxy": "http://127.0.0.1:3000" } }
    }
  }
}
```

実際に、tailnet 内の端末から `Tailscale-User-Login: someone@example.com` を付けてリクエストしても、serve が本物のログイン名で上書きしていました。HTTPS の証明書も serve が自動で取ってきます。

スタッフは tailnet に招待して使ってもらいます。ただ、同じ Pi でパスワードマネージャーや写真のサーバーも動いているので、ACL で**スタッフは `tag:cms` の 443 番だけ**に絞っています。

```jsonc
"tagOwners": { "tag:cms": ["autogroup:admin"] },
"grants": [
  { "src": ["autogroup:admin"],  "dst": ["*"],       "ip": ["*"] },
  { "src": ["autogroup:member"], "dst": ["tag:cms"], "ip": ["443"] },
],
```

## 画像：R2 に置き、アップロード時に avif / webp を作る

画像は `@payloadcms/storage-s3` で R2 に保存し、カスタムドメインから直接配信します。R2 は無料枠が 10GB で、配信（egress）に費用がかからないので、画像置き場に向いています。

```ts
s3Storage({
  bucket: process.env.R2_BUCKET,
  config: {
    endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
    region: 'auto',
    credentials: { /* … */ },
  },
  collections: {
    media: {
      prefix: 'media',
      disablePayloadAccessControl: true,
      generateFileURL: ({ filename, prefix }) => `${process.env.MEDIA_PUBLIC_URL}/${prefix}/${filename}`,
    },
  },
})
```

サイズ違いは Payload の `imageSizes` で、幅 480 / 800 / 1200 / 1600 を avif と webp で作ります。ここで 1 つハマりどころがあります。

`withoutEnlargement` の既定値（`undefined`）では、**元画像が指定の幅より小さいと、そのサイズは作られません**（`null` になります）。たとえば幅 1024 の写真だと 1200 と 1600 が無くなり、srcset の最大が 800 になってしまいます。`withoutEnlargement: true` にすると、小さい画像は拡大せず元のサイズのまま、**指定のフォーマットに変換して**作ってくれます。重複した幅は書き出すときに除いています。

```ts
const imageSizes = ['avif', 'webp'].flatMap((format) =>
  [480, 800, 1200, 1600].map((width) => ({
    name: `${format}${width}`,
    width,
    withoutEnlargement: true,
    formatOptions: { format, options: format === 'avif' ? { quality: 55, effort: 3 } : { quality: 80 } },
  })),
)
```

## 書き出し：microCMS と同じ HTML を出す

記事・バナー・カテゴリの `afterChange` / `afterDelete` フックで、公開中のものを全部書き出します。

- `content/posts/index.json`：一覧（本文なし）
- `content/posts/<slug>.json`：本文の HTML 込み
- `content/banners.json`：並び順どおり

続けて保存したときは 2 秒待って 1 回にまとめ、同時には走らせません。38 件なら、Pi 5 で数秒〜20 秒ほどで終わります。毎回全件を書き出すので、非公開にした記事や消した記事のファイルの削除も、同じ処理で済みます。

本文は Lexical の JSON なので、`convertLexicalToHTML` で HTML にします。ここで**microCMS のリッチエディタと同じ構造の HTML を出す**ことにしました。サイト側の本文の加工（画像の `<picture>` 化、目次、縦長写真の 2 枚並び）と CSS を、ほぼそのまま使えるからです。

- 写真は `<figure><img src alt width height data-avif="…" data-webp="…"></figure>`。サイトは `data-*` の srcset から `<picture>` を組み立てる
- 写真の横並びブロックは `<div class="article-gallery">`
- 埋め込みブロック（YouTube・Spotify・Instagram・Google マップ）は、microCMS と同じ `padding-bottom` の div と iframe

既定の変換器には、2 つ手を入れました。

- **空の段落**：既定では `<p><br /></p>` になります。サイトの CSS は `p:empty` を詰めていたので、そのままだと空行が 34 か所も現れました。`<p></p>` を出すように変換器を差し替えています
- **リスト**：`<li class="" style="" value="1">` のように空の属性が付き、改行も入ります。書き出しの最後に属性を掃除しています

## 移行：画像ごと自動で移す

移行スクリプトは Payload の Local API を使い、`payload run` で動かします。

1. microCMS から記事とバナーを全件取得する
2. 本文の `<img>`・アイキャッチ・バナーの画像を、クエリを外した元の画像でダウンロードし、`media` に登録する（R2 へのアップロードとサイズ違いの生成は Payload がやる）
3. 本文の HTML を Lexical に変換する
4. 書き出して、元の HTML と比べる

3 では `convertHTMLToLexical` を使いますが、`<img>` の変換には既知の問題があります。そこで、本文のトップレベルの要素を順に見て、`<figure><img>` と `<div><iframe>` は**先に取り出して**、upload ノード・埋め込みブロックを直接作るようにしました。それ以外の HTML だけを、まとめて変換しています。

```ts
for (const element of Array.from(document.body.children)) {
  const img = element.tagName === 'FIGURE' ? element.querySelector('img') : null
  const iframe = element.tagName === 'DIV' ? element.querySelector('iframe') : null
  if (img) {
    flush()
    children.push({ type: 'upload', version: 3, relationTo: 'media', value: await ensureMedia(img), fields: {} /* … */ })
  } else if (iframe) {
    flush()
    children.push({ type: 'block', version: 2, fields: { blockType: 'embed', url: iframe.src, shape: shapeOf(element) } /* … */ })
  } else {
    buffer += element.outerHTML // まとめて convertHTMLToLexical に渡す
  }
}
```

ほかに気をつけたこと：

- **何度やり直しても重複しない**：記事は microCMS の ID（`legacyId`）、画像は元の URL（`sourceUrl`）で探してから上書きする
- **URL を変えない**：これまでの記事はスラッグ未設定で、URL に microCMS のコンテンツ ID が使われていました。そこで、ID をそのままスラッグにしました。サイトマップの URL 53 件が、移行前と完全に一致しています
- **日付を引き継ぐ**：`createdAt` / `updatedAt` は Payload が上書きするので、作成後に `payload.db.updateOne` で直接書き戻す（サイトマップの `lastmod` に使うため）
- **検証**：全記事について、タグを除いたテキスト・画像の数・埋め込みの数を新旧で比べ、レポートに出す。38 件すべて一致しました

## バックアップ：Litestream で R2 へ

DB は SQLite（`@payloadcms/db-sqlite`）です。SD カードの Pi で動かすので、Litestream のコンテナを横に置き、R2 の非公開バケットへ随時複製しています。画像と書き出した JSON はもともと R2 にあるので、DB さえ戻せば元どおりです。

```yaml:litestream.yml
dbs:
  - path: /data/payload.db
    replicas:
      - type: s3
        bucket: ${LITESTREAM_BUCKET}
        path: payload.db
        endpoint: https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com
        region: auto
        access-key-id: ${R2_ACCESS_KEY_ID}
        secret-access-key: ${R2_SECRET_ACCESS_KEY}
        retention: 720h
```

戻すときは、`litestream restore -o /data/payload.restored.db /data/payload.db` で別ファイルに戻してから差し替えます。R2 の API トークンは、画像用とバックアップ用の 2 つのバケットだけに権限を絞っています。

## サイト側

`lib/microcms.ts` を `lib/cms.ts` に置き換えました。関数名とシグネチャは同じにしたので、ページ側はほぼ import の書き換えだけです。一覧は `index.json` を読み、絞り込み・並べ替え・ページ分けはサイト側で行います（38 件なら全部メモリに載せて問題ありません）。

エラーの扱いは以前と同じで、**取得の失敗は握りつぶしません**。空の一覧や 404 をキャッシュすると、壊れたページが数日残ります。それより、ビルドを失敗させて直前のデプロイを残す方が安全です。

## ハマったところ

- **ローカルで fetch が `ECONNRESET`**：Node 24（OpenSSL 3.5）の耐量子 TLS 鍵交換で ClientHello が大きくなり、一部の CDN 宛の接続がリセットされました（curl では起きない）。ローカルの移行スクリプトだけ、`require('tls').DEFAULT_ECDH_CURVE = 'X25519:P-256:P-384'` をプリロードして回避しています
- **`payload run` のスクリプトが何も出力せず終わる**：`main()` を呼ぶだけだと、読み込みが終わった時点でプロセスが終了します。トップレベルで `await main()` する必要がありました
- **Next.js 16.3 が `AGENTS.md` と `CLAUDE.md` を自動生成する**：`next dev` を起動すると、プロジェクト直下に作られます。リポジトリ直下の `CLAUDE.md` と競合するので、`agentRules: false` で止めました
- **ダークモードで埋め込みの角が白い**：ページが `color-scheme: dark`、iframe の中身がライトだと、ブラウザが iframe の下地を白で塗ります。Spotify のプレーヤーの角丸の外側や、プレーヤーの下の余白が白く見えていました。iframe に `color-scheme: light` を付けると、下地が透明になります（microCMS の頃からの問題でした）

## 費用

- R2：画像は数百 MB で、無料枠（10GB、書き込み 100 万回/月）に収まる
- Tailscale：無料プラン
- Pi：もともと動いているもの

microCMS の料金はかからなくなりました。ただ、念のため 1 か月は契約を残し、すぐ戻せるようにしています。

## まとめ

- CMS を自宅で動かすなら、**サイトから CMS を読みに行かない**構成にすると、可用性の心配がほぼ消える。公開時に完成品（JSON）を R2 に置き、再検証の通知を飛ばすだけ
- ログインは `tailscale serve` のユーザーヘッダーに任せ、Payload のカスタム認証で受ける。サイドカーとネットワークを共有して、ヘッダーを偽装できない経路にしておく
- 移行は「元と同じ HTML を出す」と決めると、サイト側の変更が小さく済み、新旧の比較もしやすい
