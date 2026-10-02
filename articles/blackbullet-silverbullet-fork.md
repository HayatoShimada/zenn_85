---
title: "SilverBullet をフォークして「AI ネイティブな自分専用メモアプリ」BlackBullet を作った話"
emoji: "🖤"
type: "tech"
topics: ["silverbullet", "mcp", "claudecode", "rag", "typescript"]
published: true
---

## はじめに

Markdown のメモを自宅の Raspberry Pi 5 で動かしている [SilverBullet](https://silverbullet.md) に溜めてきました。ジャーナル、プロジェクトのメモ、タスクがすべてプレーンな Markdown ファイルです。

このメモを中心に、次のものが欲しくなりました。

- **Notion のような操作感**: ページツリーのドラッグ、ブロックの並べ替え、データベースビュー
- **意味で引ける検索**: キーワードが一致しなくても「あのときのメモ」が出てくる
- **Obsidian のようなグラフ**: 明示リンクに加えて「内容が似ているノート」も線で結ぶ
- **AI からの読み書き**: Claude などのアシスタントが同じメモを検索・追記できる

そこで SilverBullet をフォークして **BlackBullet** を作りました。GitHub で GPL-2.0 として公開しています。

https://github.com/HayatoShimada/blackbullet

この記事では、BlackBullet の中身と、どういう順番で何を決めながら作ったかを書きます。開発はほぼすべて [Claude Code](https://claude.com/claude-code) と一緒に進めたので、その進め方にも触れます。

:::message
BlackBullet は SilverBullet プロジェクトとは無関係の非公式フォークです。SilverBullet 由来のコードは MIT、フォークで追加・変更した部分は GPL-2.0-only です。
:::

## BlackBullet とは

一言でいうと「**ローカルファースト**で **AI ネイティブ**な個人用メモアプリ」です。

- メモは自分のフォルダにある**プレーンな Markdown ファイル**。これが唯一の正で、検索インデックスもグラフもすべて派生データです
- Notion 風のエディタ（ページツリー、ブロック、データベースビュー）
- 単語と意味の両方で引けるハイブリッド検索と、関連ノート
- 意味的な類似も辺にした、ノートのグラフ
- MCP サーバーを同梱しているので、AI アシスタントが同じメモを読み書きできる
- すべて自分のマシンで動く。自分で決めない限り、メモの本文は外に出ない

![ページツリー、カバー画像とアイコンのヘッダー、下に関連ノート（デモ用のスペース）](/images/blackbullet-silverbullet-fork/01-overview.png)
*ページツリー、カバー画像とアイコンのヘッダー、下に関連ノート（デモ用のスペース）*

起動は Docker だけで完結します。

```bash
git clone https://github.com/HayatoShimada/blackbullet.git
cd blackbullet
./setup.sh
# → http://127.0.0.1:3000
```

![./setup.sh の出力。最後に MCP クライアントの登録コマンドが表示される（トークンは伏せ字）](/images/blackbullet-silverbullet-fork/02-setup.png)
*./setup.sh の出力。最後に MCP クライアントの登録コマンドが表示される（トークンは伏せ字）*

### 全体構成

![全体構成。app と memo-mcp は 1 つのネットワーク名前空間で動き、どちらも ./space の Markdown を読む](/images/blackbullet-silverbullet-fork/fig-architecture.png)
*全体構成。app と memo-mcp は 1 つのネットワーク名前空間で動き、どちらも ./space の Markdown を読む*

リポジトリには 3 つの成果物が入っています。

| 部品 | 中身 |
| --- | --- |
| ノートアプリ本体 | SilverBullet v2.11 系のフォーク（Rust サーバー + TypeScript / CodeMirror 6 クライアント） |
| `packages/memo-mcp/` | MCP サーバー兼 REST サイドカー。ハイブリッド検索・関連ノート・グラフ・RAG 用の API |
| `deploy/tailnet-proxy/` | Tailscale + Caddy + whois 認証による「tailnet 限定」の HTTPS 入口 |

## なぜ SilverBullet をフォークしたのか

SilverBullet を選び続けた理由はシンプルで、**データが Markdown ファイルのまま**だからです。アプリが消えてもメモは残り、git で履歴も取れます。さらに Space Lua というスクリプト環境とプラグ（プラグイン）の仕組みがあり、拡張しやすい作りになっています。

一方、Notion 的な操作や意味検索は「プラグを足す」だけでは届かない部分がありました。ツリーのドラッグ挙動やエディタのブロック操作はクライアントのコアに手を入れる必要があります。

フォークするにあたって、最初に次の方針を決めました。

- **Markdown ファイルが唯一の正**。インデックスは派生データで、消しても作り直せる
- **既存スペースと互換を保つ**。今のメモフォルダをそのまま開ける
- **upstream への変更は最小・追加的に**。クレート名、`SB_*` 環境変数、プラグ API、`@silverbulletmd/silverbullet/...` の import はあえて変えない。upstream のプラグがそのまま動き、upstream の変更も取り込みやすくなる
- **Pi 5 上で LLM は動かさない**。Pi は検索・インデックス・同期だけを担当する

## 開発の経緯

### 前史: memo-mcp（2026 年 9 月）

フォークより先に生まれたのが、検索サイドカーの **memo-mcp** です。最初は「Claude から自分のメモを検索・追記したい」というだけの MCP サーバーでした。

SilverBullet + Raspberry Pi + Tailscale + 自作 MCP という構成にたどり着いた経緯は、以前の記事に書きました。

https://zenn.dev/85store/articles/3f0f6a1bd22bb8

最初のコミットの翌日には、検索を**節（`##` 見出し）単位のハイブリッド検索**に作り替えています。

- **語彙検索**: SQLite FTS5 の trigram トークナイザ。日本語を形態素解析なしで部分一致できる
- **意味検索**: `Xenova/multilingual-e5-small`（ONNX q8、384 次元）をローカルの CPU で実行
- **統合**: 2 つの順位を Reciprocal Rank Fusion（RRF、k=60）でまとめ、ページ名一致・`status: active`・ジャーナルの新しさでブーストする

![節単位のハイブリッド検索](/images/blackbullet-silverbullet-fork/fig-hybrid.png)
*節単位のハイブリッド検索*

SQLite は公式の WASM ビルドをインメモリで使っています。最初の呼び出しで遅延構築し、以降は呼び出しのたびに mtime を見て差分だけ更新します。ネイティブ拡張がないので、Pi の ARM でもビルドで悩みません。

評価用のゴールデンセットも用意し、`npm run eval` で recall@k と MRR を測れるようにしました。当時の自分のメモでは **hybrid の recall@5 が 0.92、MRR が 0.856** でした。これは以降の改修の回帰基準にしています。

### Phase 0: Pi 5 の上でフォークをビルドする

2026 年 10 月 1 日にフォークを決め、まず Pi 5 で SilverBullet をビルドするところから始めました。ホストには Rust も Node 24 もないので、ビルドはすべて Docker 内で行います。

ここで最初につまずきました。

- upstream の `.cargo/config.toml` がリンカーを `aarch64-linux-gnu-gcc` に固定している。musl の Alpine イメージでは、Dockerfile 側で gcc / ar へのシンボリックリンクを張って回避した（upstream の設定は無改変）
- Debian でビルドして Alpine で実行する案（gcompat）は `__res_init: symbol not found` で起動しなかったので不採用

release イメージのビルドには Pi で 10〜20 分かかります。これでは 1 行直すたびに待つことになるので、開発用の高速ループ `scripts/fork-dev.sh` を作りました。

ポイントは、debug ビルドの `rust-embed` が `client_bundle/` を**実行時にディスクから読む**ことです。そこで次のように分けました。

- サーバーは debug バイナリを一度だけビルドする
- クライアントは Node 24 コンテナでビルドして `client_bundle/` に出力し、実行中のコンテナに read-only でマウントする

実測はこうなりました。

| 操作 | 時間 |
| --- | --- |
| クライアント 1 行変更 → 再ビルド → ブラウザをリロード | **約 8 秒** |
| Rust 1 行変更 → 再ビルド | 約 2 分 20 秒 |
| コールドの debug サーバービルド | 約 10 分 |

### Phase 1: 検索パネルとセマンティックなグラフ

最初の機能は、memo-mcp の検索をアプリから使えるようにすることでした。

まず memo-mcp に REST API（`/api/search`、`/api/related`、`/api/graph`）を足しました。ブラウザからは直接呼べないので（SilverBullet には CORS 層がありません）、サーバーの `/.proxy/{host}/{path}` を経由します。この経路は Rust に手を入れずに使えるのが利点です。

```lua
-- スペースの CONFIG ページ（setup.sh が自動で書きます）
config.set("memoSidecar", {url = "/.proxy/127.0.0.1:3010", token = "...", space = "notes"})
```

- **`Memo: Search`**（`Ctrl-Shift-f`）: Space Lua の `view.define` で作った検索パネルです。節単位で結果を出し、Enter で該当する見出し行へ飛びます。Space Lua のライブラリはビルド時に埋め込まれるので、Rust の変更は不要でした
- **`Memo: Related Notes`**: ページ下部に関連ノートを出すドックです
- **グラフ**: upstream に既にあった `plugs/object-graph/` を拡張しました。埋め込みの類似度による辺を破線で描き、類似度のしきい値・k・Hops・Status / Area のフィルタを付けています

e5 の類似度は全体的に高めの値に集まります。そのため、しきい値 0.80 ではほとんど何も絞れず、既定値は 0.92 に落ち着きました。

![Memo: Search の Details 表示。L は語彙検索の順位、S は意味検索の順位、右端は RRF のスコア](/images/blackbullet-silverbullet-fork/03-search.png)
*Memo: Search の Details 表示。L は語彙検索の順位、S は意味検索の順位、右端は RRF のスコア*

![グラフ。実線が明示リンク、ティールの破線が内容の似ているノート（意味辺）](/images/blackbullet-silverbullet-fork/04-graph.png)
*グラフ。実線が明示リンク、ティールの破線が内容の似ているノート（意味辺）*

実ブラウザ（Playwright + Chromium）で確かめている途中で、いくつかバグが見つかりました。

- `Page@L<n>` 形式のリンクが、後ろの行ほど手前にずれていく。原因は `plug-api/lib/ref.ts` の `getOffsetFromLineColumn` が改行を数えていなかったことで、upstream のバグでした
- `net.proxyFetch` は上流が 401 を返しても**外側は 200**を返し、本当のステータスは `res.status` に入る。`res.ok` だけで判定していたため、トークンが間違っていても、グラフはエラーを出さずに意味辺 0 本で描かれていた
- ローカルグラフの Hops 2 が Hops 1 と同じ表示になっていた

### UI の設計ルール: Passive View + Chain of Responsibility + Mediator

Phase 2 に入る前に、UI の設計ルールを決めました。今もリポジトリの `CLAUDE.md` の先頭に書いてあります。

> すべてのコンポーネントを Root からなる階層構造下に置き、各コンポーネントは MVP パターンの Passive View として描画に関わるパラメータだけを操作し、動作は Chain of Responsibility でイベントをバブリングさせて、ステートマシンとして振る舞う Mediator に裁定させること。
> 設計しやすさではなく、ユーザビリティを高めること。

具体的には、どの機能も次の 3 層に分けています。

```ts
// Mediator: 純関数の状態機械（副作用なし、テストしやすい）
export function transition(state: TreeState, event: TreeEvent): Transition {
  switch (state.kind) {
    case "idle":
      return fromIdle(state, event);
    // dragging / picking / moving ...
  }
}
// → { state, effects } を返す

// Runner: effects を実行して、結果をまたイベントとして Mediator に戻す
//         （move.done / move.noop / move.failed など。依存はすべて注入）

// View: props を描画して emit するだけ。editor.* や syscall を直接呼ばない
```

![View・Mediator・Runner の関係](/images/blackbullet-silverbullet-fork/fig-mediator.png)
*View・Mediator・Runner の関係*

ドラッグ、ピッカー、書き込み中といった状態が絡み合う UI では、この形の効き目が大きいと感じました。たとえばツリーの移動中は他のイベントをすべて無視するので、rename が二重に走りません。DB ビューでは書き込みを 1 件ずつ直列化しています。遅れて届いた古い応答は捨てられます。こうした「ユーザーが困る競合」が、状態機械のテストで潰せます。

Phase 1 のグラフ UI はこのルールを決める前に作っていたので、あとから合わせました。約 10 個の状態を抱えた 468 行の `App` コンポーネントを、Mediator・Runner・Passive View に分解しています。

### Phase 2: ページツリー

調べてみると、ページ名のパスによる階層、ドラッグ移動、リンクの書き換えは upstream に既に揃っていました。そこで `parent` のような frontmatter は**導入しない**ことにしました。パスを階層の正とし、二重の真実を作らないためです。

足したものは次のとおりです。

- ツリーの Mediator（idle / dragging / picking / moving）と、移動の Undo（`Tree: Undo Move`、8 秒の猶予）
- 行アクションの `Move to…` と Pin / Unpin
- **同じフォルダ内の手動並べ替え**。順序は `pageDecoration.tree.priority` で表します。そのうえで「上げるだけ」と「下げるだけ」の 2 つの戦略を比べ、**書き換えるファイル数が少ない方**を選ぶ `planReorder` を純関数で書きました。総当たりテストで、計画を適用すると要求どおりの並びになることを確認しています。複数ファイルへの書き込みは、すべて成功するか、すべて巻き戻すかのどちらかです
- ページヘッダー（カバー画像と大きな絵文字アイコン）
- `@` 補完に人・日付・ページを混ぜる（`@今日` で今日のジャーナルにリンク）

![ツリーで「技術ブログ執筆」をドラッグ中。挿入位置に線が出る](/images/blackbullet-silverbullet-fork/05-tree-drag.png)
*ツリーで「技術ブログ執筆」をドラッグ中。挿入位置に線が出る*

![@ の補完に、人・日付・ページが混ざって出る](/images/blackbullet-silverbullet-fork/06-mention.png)
*@ の補完に、人・日付・ページが混ざって出る*

ここでも upstream のバグを 1 つ直しています。`plug-api/lib/yaml.ts` の `applyPatches` は、既存のキーが複数行の値（ネストした mapping や list）を持つとき、キーの行だけを置き換えていました。古い行が残るので、**キーが重複した壊れた frontmatter** ができてしまいます。

### Phase 3: ブロックエディタ

CodeMirror 6 の上に、Notion のようなブロック操作を載せました。

- 段落、見出しの節、リスト項目（子ごと）、引用、コード、表を「ブロック」として構文木から導出する
- 左余白の「⠿」ハンドルでドラッグして並べ替える。リスト項目は**横方向の移動量でネストの深さ**が変わる
- 見出しの折りたたみ（「▾ / ▸」）

設計で気をつけた点は次のとおりです。

- ハンドルと折りたたみは `.cm-line::before / ::after` で描き、**DOM 要素を足さない**。キャレットやクリック位置に影響を出さないため
- 移動は `minimalChange` で文字単位の最小の 1 変更にして、1 回の `dispatch` で反映する。カーソル、折りたたみ、Undo 履歴が保たれ、`Ctrl-z` 一発で戻せる
- 日本語 IME の変換中（`view.composing`）はドラッグとホバーを無視する
- CodeMirror の更新中にレイアウトを読むと `Reading the editor layout isn't allowed during an update` で落ちる。挿入ラインの位置計算は `queueMicrotask` で更新の外に出した
- 600 節（約 6,600 行）のページでも動くよう、ドラッグ中はポインタの前後 60 行だけを計測する

![ブロックエディタ。左余白のハンドルでドラッグ、三角で折りたたみ](/images/blackbullet-silverbullet-fork/07-block.png)
*ブロックエディタ。左余白のハンドルでドラッグ、三角で折りたたみ*

### Phase 4: データベースビュー

ページ内に ```` ```db ```` ブロックを書くと、frontmatter を持つページやタスクが表・ボード・カレンダーとして表示され、その場で編集できます。

````markdown
```db
source: projects
view: board
```
````

ボードのカードをドラッグすると、そのページの `status` が書き換わります。カレンダーでドラッグすると期限が変わり、表のセルを編集すると frontmatter が更新されます。

![db ブロックのボード表示。カードをドラッグすると status が書き換わる](/images/blackbullet-silverbullet-fork/08-db-board.png)
*db ブロックのボード表示。カードをドラッグすると status が書き換わる*

ここでの一番の問題は、**Markdown ファイルが正**であることとの整合でした。

- 書き込む直前にページの更新日時を、行を読んだ時点の `modified` と比べる。違っていれば書かずに「ページが変わっています」と通知する
- 画面は**書き込みが成功してから**更新する（楽観的更新はしない）
- 書き込み直後に読み直すと、インデックスがまだ古いことがある。そのまま表示すると値が元に戻ったように見えるので、書いたページのインデックス上の更新日時が追いつくまで、400ms × 最大 12 回待つ

その後、Notion の「データベース」と「ビュー」を分けたいという要望（自分の）が出てきました。それに応えて `database.define` を足しています。

```lua
database.define {
  name = "projects",
  tag = "project",
  folder = "Projects/",
  template = "Templates/Project",
  properties = {
    { key = "status", type = "select", options = {"active", "someday", "done"}, default = "active" },
    { key = "due",    type = "date" },
    { key = "area",   type = "page" },
  },
  order = {"active", "someday", "done"},
}
```

```` ```db ```` ブロックで `database: projects` と書くと、型・選択肢・列順がビューに反映され、「+ New」でテンプレートから行（＝ページ）を作れます。続いて、行メニュー（リネーム、複製、アーカイブ、ゴミ箱）、保存ビュー、`lt` / `before` / `contains` などのフィルタ、タッチ操作でのドラッグも足しました。

![database: projects のテーブル表示。右上の「+ New」で行（ページ）を作り、「…」で行メニュー](/images/blackbullet-silverbullet-fork/09-db-table.png)
*database: projects のテーブル表示。右上の「+ New」で行（ページ）を作り、「…」で行メニュー*

### 公開: 1 コマンドで動くようにする

10 月 2 日、自分用だったものを汎用化して公開しました。

- `compose.yaml` で `app` と `memo-mcp` を**同じネットワーク名前空間**に置く。`/.proxy/` はサーバー側から接続するので、ブリッジネットワークのコンテナ内では `127.0.0.1` がコンテナ自身を指してしまうためです（ここは実際にハマりました）
- `setup.sh` が `.env` を作り、トークンを生成し、スペースの `CONFIG` にサイドカーの接続設定を書き込み、MCP クライアント用のコマンドを表示する
- 個人的なホスト名・パス・例をすべて取り除く
- upstream のファイルは MIT のまま、追加分は GPL-2.0-only。`NOTICE.md` に追加部分を列挙する

公開の直後に、memo-mcp のバグを 1 つ直しています。埋め込みモデルのダウンロードに失敗すると、**reject された Promise がキャッシュされたまま**になり、以降の検索がすべて 500 を返していました。今は次の呼び出しで再試行し、それまでは語彙検索だけで答えて `warning` を返します。モデルが取れなくても検索が止まらないようにするのが方針です。

### その後: Memo: Ask と文書検索

公開後には、次の機能を足しました。

- **`Memo: Ask`**: メモに質問すると、サイドカーが関連する節を集め、アプリの `/.proxy/` 経由で Anthropic の Messages API に送り、`[[Page@L12]]` 形式の引用付きで答えます
  - `:confidential` と印を付けたスペースは、明示的に許可しない限り送りません（印の有無が分からない場合も機密扱い）
  - メモ本文は「データ」として区切って渡し、回答に含まれる Space Lua は無効化します
  - フォローアップ質問、スコープ指定（`in:Projects/`）、回答のノート保存、プロンプトキャッシュにも対応しています
- **PDF / Office 文書の検索**: `.pdf`、`.docx`、`.xlsx`、`.pptx` などを `pdftotext` / `unzip` で抽出し、「`<file> p.3`」「`slide.N`」の単位で同じインデックスに入れます。OCR（tesseract）はオプションです
- **運用コマンド**: `./setup.sh --backup / --restore / --status / --upgrade`

![Memo: Ask の流れ](/images/blackbullet-silverbullet-fork/fig-ask.png)
*Memo: Ask の流れ*

## AI から使う

memo-mcp は MCP を `http://127.0.0.1:3010/mcp` で話します。Claude Code なら次のように登録できます。

```bash
claude mcp add --transport http memo http://127.0.0.1:3010/mcp \
  --header "Authorization: Bearer <MEMO_MCP_TOKEN>"
```

ツールは `search_notes`、`read_note`、`related_notes`、`list_tasks`、`list_journal`、`append_journal`、`add_inbox` などです。書き込み系は、読んだときの `modified` を `expected_modified` として渡す楽観的な競合検出付きです。書き込み自体も、同じディレクトリの一時ファイルに書いてから fsync と rename をするアトミックな方式にしています。

![Claude Code から memo-mcp のツールを呼び、メモを根拠に答えたところ（claude -p の実行ログから描き起こし）](/images/blackbullet-silverbullet-fork/11-claude-code.png)
*Claude Code から memo-mcp のツールを呼び、メモを根拠に答えたところ（claude -p の実行ログから描き起こし）*

`CONFIG.md` にはサイドカーのトークンや API キーが入るため、インデックスに入れないだけでなく、すべての読み書き経路で拒否しています。

## 外から使う: tailnet 限定の入口

スマホや外出先から使いたい。でも、メモアプリにパスワード画面を作ってインターネットに晒すのは避けたい。そこで `./setup.sh --tailnet` で、Tailscale のネットワーク内からだけ届く HTTPS の入口を立てられるようにしました。

![tailnet 限定の入口。インターネットからは届かない](/images/blackbullet-silverbullet-fork/fig-tailnet.png)
*tailnet 限定の入口。インターネットからは届かない*

- 証明書は Let's Encrypt の DNS-01（Cloudflare）で取るので、受信ポートを開ける必要はありません。名前は普通のドメインですが、解決先は tailnet のプライベートアドレスです
- 認証サービスが `tailscaled` に「接続元は誰か」（whois）を問い合わせ、許可リストにあるログインだけを通します。Tailscale の ACL とは独立した二重のチェックです
- Caddy はクライアントが送ってきた `Tailscale-User-*` ヘッダーを必ず捨ててから、認証サービスの答えを付け直します。こうすることで、アプリはこのヘッダーを信頼できます

## Claude Code との開発の進め方

この開発はほぼすべて Claude Code と進めました。memo-mcp の最初のコミットから約 2 週間、フォーク本体は着手から公開までおよそ 2 日です。うまくいったと感じたやり方を挙げます。

- **調査してから計画を直す**。「ファイルツリーを作る」「グラフを作る」と計画しても、調べると upstream に既にあることがよくありました（ツリーのドラッグ移動、object-graph、スラッシュメニュー）。各フェーズの最初に読み取り専用の調査を挟み、ゼロから作らず拡張する方針にしたので、変更が小さく保てました
- **設計書を先に書き、決めることはオーナーが決める**。`dev-docs/phase*-design.md` に調査結果・判断・未決事項を書きました。「見出しは節ごと動かす」「折りたたみ状態は保存しない」「`@` にページと日付を混ぜる」といった UX の判断は、選択肢を並べてもらって自分で決めています
- **引き継ぎ資料（HANDOFF.md）を育てる**。セッションをまたいでも文脈が切れないよう、状態・決定・実測値・見つけたバグ・未確認事項を 1 枚にまとめ続けました。最終的に 50KB を超えましたが、「未確認」を正直に書いておく欄が一番役に立ちました
- **実データに触らない**。テストは必ずスペースのコピーで行い、サイドカーの実データへの書き込みテストは環境変数で明示しない限り走らないようにしました。`CLAUDE.md` にも「実ノートに向けない」「同じスペースを 2 つのサーバーで同時に編集しない」と書いています
- **純関数 + 実ブラウザ**。Mediator や並べ替え計画のような判断は純関数にして、vitest で総当たりに近いテストをしました。最後は Playwright + Chromium で、実際にドラッグして確かめています。upstream のバグのいくつかは、この実ブラウザ確認で見つかりました

## おわりに

BlackBullet は「Markdown ファイルが唯一の正」という SilverBullet の美点を保ったまま、Notion の操作感、意味検索、グラフ、AI アクセスを足したメモアプリです。

まだ足りないところもあります。モバイルでの操作感、数千ページ規模での検証、共同編集（upstream はファイル単位の 3-way マージで、CRDT はありません）などです。それでも毎日のメモはすでにこれで書いていて、「あのメモどこだっけ」を Claude に聞けるようになりました。メモアプリとの付き合い方が変わったと感じています。

興味があれば、ぜひ触ってみてください。

https://github.com/HayatoShimada/blackbullet
