---
title: "「AIに選ばれる店」をつくる。地方の古着屋が1日でやった、AI時代の小売店舗戦略"
emoji: "🛍️"
type: "idea"
topics: ["ai", "seo", "shopify", "llm", "claudecode"]
published: true
---

## はじめに

富山県南砺市井波で、古着・セレクトショップ「[85-Store（ハコストア）](https://85-store.com)」をやっています。木彫りの町の、人口の少ない地域にある小さなお店です。

最近、「富山 古着屋」「GILDAN のスウェット 古着」のような探し物を、検索エンジンではなく ChatGPT や Gemini に聞く人が増えました。Google の検索結果の上にも、AI の要約が出るようになっています。

ここで困るのは、**AI はお店のことを「知っている範囲」でしか答えない**ことです。AI に知られていなければ、答えの候補にすら入りません。間違って知られていれば、間違ったまま紹介されます。

そこで、AI コーディングエージェントの Claude Code と一緒に、1日（2026年10月5日）かけて、お店のデータを「AI に正しく伝わる形」に作り直しました。この記事は、そのときの取り組みと、わかったことのまとめです。

:::message
**結論**: AI 時代の小売店で効くのは、新しい魔法ではありませんでした。次の4つを、地道に揃えることでした。
1. **正しいデータを1か所に持つ**（商品は Shopify、店舗情報はサイトの1ファイル）
2. **機械が読める形で出す**（構造化データ、llms.txt、Merchant Center）
3. **変更をすぐに伝える**（営業日カレンダーの webhook）
4. **外からの裏付けを増やす**（地図のプラットフォーム、レビュー、第三者のサイト）
:::

## お店の構成

- **サイト**（[85-store.com](https://85-store.com)）: Next.js、Vercel。ブログ・お店の案内・来店予約
- **オンラインストア**（[shop.85-store.com](https://shop.85-store.com)）: Shopify
- **CMS**: 自宅の Raspberry Pi 5 で動く Payload CMS。記事と、Shopify の商品の入力画面
- **営業日カレンダー**: Cloudflare Workers。休業日をスマホから入れると、サイトにすぐ出る

## 1. AI に「中古なのに新品」と思われていた

最初に Google の Merchant Center（ショッピング検索や AI の商品の答えの元になるデータ）を API で調べて、驚きました。

| 項目 | 結果 |
|---|---|
| 状態（condition） | **805件のうち804件が「新品」**。古着屋なのに |
| ブランド | ほぼ全部が店名の「85-store」 |
| 返品ポリシー | 「不良品のみ返品可」。実際は「15日以内なら返品可」 |
| 送料 | 一律500円。実際は「1万円以上は無料」 |

原因は単純でした。Shopify の Google アプリは、状態を**バリアントのメタフィールド** `mm-google-shopping.condition` から読みます。この値がないと「新品」になります。誰もそれを入れていなかったのです。

CMS の「区分」（古着・新品・委託）から自動で入れるようにし、既存の 428商品・541バリアントには一括で入れました。

```ts
// 区分（タグ）から、Google に送る状態を決める
export const googleCondition = (kind: Kind) => (kind === 'new' ? 'NEW' : 'USED')

await shopifyGraphQL(`mutation($m: [MetafieldsSetInput!]!) { metafieldsSet(metafields: $m) { userErrors { message } } }`, {
  m: variantIds.map((ownerId) => ({
    ownerId, namespace: 'mm-google-shopping', key: 'condition', type: 'single_line_text_field', value: googleCondition(kind),
  })),
})
```

数時間後には、Merchant Center の表示が「中古 230件・新品 85件」（商品の数）に変わりました。返品ポリシーと送料も、Merchant API でストアの実際の条件に揃えました。

**教訓**: AI に間違って覚えられる原因の多くは、AI ではなく、**自分たちが出しているデータの欠け**です。

## 2. 写真を「機械にも人にも」読みやすくする

### 1227枚を縮めて、alt を入れる

Shopify の商品画像は 1633枚あり、そのうち 1227枚が長辺 4000px を超えていました。手元の PC の GPU（Radeon RX 7900 XTX、ROCm）で OpenCV を動かし、長辺 2048px に縮めて差し替えました。

- 容量: 2539MB → 699MB（72%減）
- 画像の ID と並び順はそのまま（Shopify の `fileUpdate` で中身だけ差し替え）

あわせて、画像の説明文（alt）が空だった 1617枚に、「商品名（n枚目）」を入れました。AI は画像そのものより、まず文字を読みます。

### 写真の色を揃える

撮った日やカメラで、写真の色がばらばらでした。青っぽい壁の写真もあれば、黄色っぽい壁の写真もあります。

そこで、次の2段で揃えました。
1. **写真の種類を見分ける**: CLIP（画像と文章の類似度のモデル）で、「単品」「着用」「ディテール」に分けました。1555枚のうち、単品は 780枚でした。
2. **単品の写真だけ、壁の色で合わせる**: 基準の写真の右上の壁の色（RGB 205, 207, 207）に、それぞれの写真の同じ位置の壁が揃うよう、リニア RGB でゲインを掛けました。

![補正の前後（左が補正前、右が補正後。前面と背面）](/images/ai-era-retail-strategy/white-balance.jpg)

白い T シャツは、1色だけ白飛びしていることが多く、そのまま補正すると黄色くなります。そこで、補正後に白に近い画素は無彩色に寄せるようにしました。

これからは、撮影のときに右上にカラーチェッカーを置き、写真ごとに色を合わせる予定です。

## 3. 構造化データで「お店」と「商品」を結ぶ

AI と検索エンジンは、ページの文章に加えて、構造化データ（schema.org の JSON-LD）を読みます。

### サイト（85-store.com）

ClothingStore（実店舗）に、住所・緯度経度・営業時間・電話・支払い方法・取り扱いブランドを入れています。あわせて、FAQ のページ（FAQPage）と [llms.txt](https://85-store.com/llms.txt)（AI 向けのサイトの要約）も作りました。

### オンラインストア（shop.85-store.com）

Shopify 標準の構造化データには、古着の状態も、本来のブランドも、送料と返品の条件も入っていませんでした。そこで、テーマで自前の JSON-LD を出すようにしました。

```liquid
{%- liquid
  assign condition_url = 'https://schema.org/UsedCondition'
  if product.tags contains 'NOT USED'
    assign condition_url = 'https://schema.org/NewCondition'
  endif
-%}
"itemCondition": {{ condition_url | json }},
"brand": { "@type": "Brand", "name": {{ product.metafields.custom.brand.value | json }} },
"seller": { "@id": "https://85-store.com/#organization" },
```

ポイントは、**`@id` でサイトとストアを同じ「お店」として結んだ**ことです。サイトの Organization（`https://85-store.com/#organization`）を、ストアの商品の売り手として参照しています。AI から見て、「井波の古着屋」と「オンラインストア」が別々の何かではなく、1つのお店になります。

## 4. 臨時休業を、保存した瞬間に AI へ

地方の小さなお店は、臨時休業や時間変更が多いです。AI に古い営業時間で案内されると、せっかく来てくれたお客さまが閉まった店の前に立つことになります。

営業日カレンダーは、もともとブラウザから読む仕組みでした。そのため、人には即時に見えても、JavaScript を動かさない AI のクローラーには見えませんでした。そこで、次の流れにしました。

```
カレンダーで保存 → Worker が署名付き webhook を送る
  → サイトの /api/revalidate が revalidateTag("business-calendar")
  → 構造化データ（specialOpeningHoursSpecification）と llms.txt が取り直される
```

```ts
// Cloudflare Workers: 本文の HMAC-SHA256 を署名にして、保存のたびに知らせる
const body = JSON.stringify({ ...extra, event: "calendar.updated", updatedAt: now.toISOString() });
await fetch(env.WEBHOOK_URL, {
  method: "POST",
  headers: { "Content-Type": "application/json", [env.WEBHOOK_SIGNATURE_HEADER || "x-signature"]: await sign(body, env.WEBHOOK_SECRET) },
  body,
});
```

llms.txt には、「今後の臨時休業・営業時間の変更」の一覧が出ます。

## 5. レビューを集める（Shopify の Plus でなくても）

AI がお店を推すかどうかには、レビューが効きます。オンラインストアでは、Google カスタマーレビュー（購入後に Google から満足度のアンケートが届く仕組み）を入れました。

ただ、Shopify では、Plus 以外のプランだと注文完了ページにスクリプトを入れられません。そこで、次の形にしました。
- 注文完了ページに、チェックアウト UI 拡張で「アンケートに協力する」ボタンを置く
- ボタンから、テーマの専用ページで Google の参加画面を出す

注文の情報（メールアドレスなど）は、URL の `#` 以降で渡しています。`#` 以降はサーバーにもアクセス解析にも送られないので、ログに残りません。

## 6. 外からの裏付け（ローカル GEO）

AI は、自分のサイトの説明よりも、**別々の信頼できるサイトが同じことを書いているか**を重く見ます。

- **地図**: Google ビジネスプロフィールに加え、Bing Places（ChatGPT の検索は Bing の索引を使う）と、Apple Business Connect（Apple マップと Siri）に登録する
- **表記を揃える**: 店名・住所・電話・URL を、どこでもまったく同じ書き方にする。揺れると「別の店」と見なされる
- **第三者のサイト**: 地域の観光協会・商工会のお店の紹介、取り扱いブランドの「取扱店」ページ、地元のメディア、イベントの出店者一覧

ここはコードでは片付かない、地道な仕事です。

## 7. 仕組みとして続ける

1日で直しても、日々の入力でまた崩れては意味がありません。そこで、CMS に次のものを足しました。
- Merchant Center の掲載の状態・問題・クリック数を、6時間ごとに取り込んで商品の画面に出す
- カラー・生地・素材の入力欄（Shopify の絞り込みと、Google の「色」に使われる）
- イベントの記事の日時と会場の欄（サイトが Event の構造化データにする）

## わかったこと

- AI 時代の店舗戦略の出発点は、**「AI にどう見えているか」を測ること**でした。Merchant Center を開くまで、古着がほぼ全部「新品」で出ていることに誰も気づいていませんでした。
- やることは、昔からの「正確な情報を、どこでも同じに」です。違うのは、**読み手が人だけでなく機械にもなった**ことです。機械が読める形（構造化データ、llms.txt、API）で出し、変わったらすぐ知らせる。
- 小さなお店ほど有利な面もあります。商品も情報も少ないので、全部を正しく揃えきれます。
- 1日でここまでできたのは、AI エージェントと一緒に作業したからです。調査・実装・確認を任せ、こちらは「どうしたいか」を決めることに集中できました。

効果（AI の答えに出てくるか、来店やオンラインの注文が増えるか）は、これから測っていきます。

:::message
この記事の取り組みのうち、オンラインストアの構造化データと、営業日カレンダーの webhook は、公開の準備中です（実装は済んでいます）。Bing Places・Apple Business Connect への登録と、第三者のサイトへの掲載の依頼は、これから進めます。
:::

## 85-Store について

富山県南砺市井波の古着・セレクトショップです。アメリカ・ヨーロッパの古着と、River・VOIRY・SOWBOW などのブランドを扱っています。2階は共創スペース「85-UpStore」です。

- 住所: 富山県南砺市本町4丁目100
- 営業時間: 12:00〜18:00（木曜定休。営業日は[サイト](https://85-store.com)のカレンダーで）
- オンラインストア: [shop.85-store.com](https://shop.85-store.com)（1万円以上で送料無料）
- Instagram: [@85store_inami](https://www.instagram.com/85store_inami/)

井波に来たときは、ぜひ寄ってください。AI に「富山の古着屋」と聞いて、うちが出てきたら、なおうれしいです。
