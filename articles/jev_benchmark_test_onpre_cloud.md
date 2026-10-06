---
title: "Clef-flashでの画像分類をベンチマークする（オンプレ vs クラウド）"
emoji: "📸"
type: "tech"
topics: ["jev", "clef"]
published: true
---

# Clef-flashでの画像分類をベンチマークする（オンプレ vs クラウド）

## はじめに

エンジニアが小売りをやると、最新技術を使いたいがために「どうでもいい仕事を探す」という本末転倒が発生しがちです。

最近はAIにオンラインストアの商品を検索させる機会が増えており、これまで以上に画像の `alt` 属性の役割が重要になってきています。そこで今回は、Cloudflare Workers AI の新モデル **`clef-flash`** を使って、EC（Shopify）の商品画像を自動分類し、`alt` の自動作成や画像整理を強化する仕組みを構築しました。

あわせて、Cloudflare Workers AI（クラウド）と、手元の AMD GPU（オンプレ）での推論速度・精度・コストをベンチマーク比較してみました。

![](/images/jev/jev_1.png)

---

## 全体構成と分類の流れ

商品写真の分類には、Cloudflare Workers AI の `@cf/cloudflare/clef-flash` を使用します。

### 構成

* **Shopify API**: 商品画像の取得
* **Cloudflare Workers AI**: `clef-flash` による画像判定
* **ホストPC / ローカルUI**: 中継処理および人間による判定レビュー画面

### 分類ロジック

Clef は一般的なVLMのように長文テキストを生成するのではなく、与えられた質問（はい・いいえ）に対して各選択肢の確率を返す判定専用モデルです。写真ごとに以下の2つの質問を投げます。

1. **人が写っているか**（着用判定）
2. **商品の全体が1枚に収まっているか**（全体 vs アップ判定）

この2つの質問に対する応答確率（しきい値 0.5）から、画像を以下の3パターンに振り分けます。

* **着用 (worn)**: 人（着用・手など）が写っている
* **全体 (whole)**: 人はおらず、商品の全体が写っている
* **アップ (closeup)**: 人はおらず、タグ・生地・部分拡大などが写っている

---

## 実装（Cloudflare Workers AI 呼び出し）

Workers AI の Clef は画像を Base64 形式で受け取ります。混雑時（HTTP 429 / 5xx）のエラーハンドリングとリトライ処理を組み込んだ分類ロジックの実装例です。

```python
"""商品写真に人が写っているか・商品の全体が写っているかを、
Cloudflare Workers AI の Clef で判定する。

Clef は、質問（はい・いいえ など）ごとに選択肢の確率を返す判定用のモデル（文章は生成しない）。
画像は base64 で渡す（URL は受け付けない。1枚 4MiB・最大4枚）。
"""

import base64
import time
from dataclasses import dataclass

import httpx

DEFAULT_MODEL = "@cf/cloudflare/clef-flash"
VERSION = 1
STATE = "オンラインストアの商品写真を分類する"
QUESTIONS = {
    "person": {
        "type": "noul",
        "instructions": "写真に人（体の一部を含む。着用している人・モデル・手）が写っているか",
    },
    "whole": {
        "type": "noul",
        "instructions": (
            "商品（服・小物）の全体が1枚に収まっているか。"
            "一部だけのアップ（タグ・生地・ボタン・プリントの寄り）なら no"
        ),
    },
}
THRESHOLD = 0.5

class ClefError(Exception):
    pass

@dataclass(frozen=True)
class PhotoLabels:
    """はいの確率（0〜1）。"""

    person: float
    whole: float

    @property
    def kind(self) -> str:
        return photo_kind(self.person >= THRESHOLD, self.whole >= THRESHOLD)

def photo_kind(person: bool, whole: bool) -> str:
    """写真の区分。人が写っていれば着用、いなければ全体かアップ。"""
    if person:
        return "worn"
    return "whole" if whole else "closeup"

KIND_LABELS = {"worn": "着用", "whole": "全体", "closeup": "アップ"}

class Clef:
    def __init__(self, account_id: str, api_token: str, model: str = DEFAULT_MODEL, http=None):
        if not account_id or not api_token:
            raise ClefError("CLOUDFLARE_ACCOUNT_ID と CLOUDFLARE_API_TOKEN を設定してください")
        self.url = f"https://api.cloudflare.com/client/v4/accounts/{account_id}/ai/run/{model}"
        self.model = model.rsplit("/", 1)[-1]
        self._http = http or httpx.Client(timeout=60)
        self._headers = {"Authorization": f"Bearer {api_token}"}

    def classify(self, image: bytes, content_type: str) -> PhotoLabels:
        payload = {
            "model": self.model,
            "state": STATE,
            "questions": QUESTIONS,
            "images": [{"content_type": content_type, "base64": base64.b64encode(image).decode()}],
        }
        wait = 1.0
        for _ in range(6):
            res = self._http.post(self.url, json=payload, headers=self._headers)
            if res.status_code == 429 or res.status_code >= 500:
                time.sleep(wait)
                wait = min(wait * 2, 16)
                continue
            body = res.json()
            if not body.get("success"):
                errors = " / ".join(e.get("message", "") for e in body.get("errors", []))
                raise ClefError(f"Workers AI: {res.status_code} {errors}")
            answers = body["result"]["answers"]
            return PhotoLabels(person=answers["person"]["noul"], whole=answers["whole"]["noul"])
        raise ClefError("Workers AI: やり直しても応答がありません")
```

---

## 運用ツールとプロンプト改善

全画像に対する判定結果の確からしさを人間の目で素早くチェックできるよう、確認用 Web UI（`photo_review.py` / `photo_review.html`）をローカル（`127.0.0.1:8010`）に用意しました。

### 確認UIの特徴

* キーボード（`1` `2` `3`）で直感的に正しい区分を選べる。
* 確率が `0.5` に近く判定に迷っている画像から優先表示。
* ページ単位の一括確認（`Enter`）や正解率のリアルタイム集計機能を実装。

### プロンプトチューニング

![](/images/jev/jev_3.png)

初期プロンプト（`ver.1`）では、「人が写っているか？」という設問に対して**商品の端を持った指**が人間として検知されてしまい、商品単体写真が「着用」に誤分類されるケースがありました。

プロンプトを以下のように見直すことで精度が向上しました。

* **変更前**: `写真に人（体の一部を含む）が写っているか`
* **変更後**: `写真に人が着用しているか`

プロンプト改善後の誤判定は3件のみとなり、それらもチラシや店内写真といった商品以外の例外画像（イレギュラーデータ）でした。

---

## オンプレ（ローカルGPU）での検証環境

比較のため、Hugging Faceで公開されている [Cloudflare/clef-flash](https://huggingface.co/Cloudflare/clef-flash) を手元のローカル環境に載せて推論を動かしてみました。

* **GPU**: AMD Radeon RX 7900 XTX (VRAM 24GB / 当時8万円で購入)
* **環境**: ROCm 10版 PyTorch 2.13 + transformers 5.10.2
* **CPU**: Intel Core Ultra (Z890) ※Ryzenにしたい……
* **設定**: 量子化なしでロード（VRAM約19GB使用）

### オンプレ推論のボトルネックとロード時間

* **ライブラリ読み込み**: 約0.7秒
* **モデルのGPU配置**: 約6.4〜6.6秒
* **初回推論（カーネル準備込み）**: 約2.0秒
* **起動オーバーヘッド合計**: **約9秒**

バッチサイズは **4〜8** でスループットが頭打ち（毎秒 2.8 枚程度）となり、バッチサイズを 16 まで上げると VRAM 使用量が 23.4 GB に達して速度低下（スワップ・メモリの詰め込みによる失速）が発生しました。

---

## クラウド vs オンプレ ベンチマーク比較

実画像 1,555 枚・4問判定（同等条件）で処理速度および精度を比較しました。

### 処理速度・実行時間の比較


| 環境                        | 処理速度     | 1枚あたりの時間    | 全1555枚の処理時間 |
| ------------------------- | -------- | ----------- | ----------- |
| **Workers AI（1並列）**   | 2.2 枚/秒  | 往復中央値 250ms | 約 12 分      |
| **Workers AI（6並列）**   | 12.1 枚/秒 | 往復中央値 257ms | 約 2 分 10 秒  |
| **Workers AI（12並列）**  | 20.3 枚/秒 | 往復中央値 295ms | 約 1 分 20 秒  |
| **オンプレ 7900XTX（1並列）** | 2.5 枚/秒  | 中央値 424ms   | 約 10 分      |
| **オンプレ 7900XTX（8並列）** | 2.8 枚/秒  | 中央値 362ms   | 約 9 分       |


* **単体パフォーマンス**: 単一リクエストあたりの処理能力（1並列）はオンプレもクラウドも大きな差はありません。
* **並列性能**: Workers AI 側でマルチリクエスト（12並列）にスケールさせることで、全1,555枚の処理が**わずか1分20秒**で完了します（※ Workers AI 側の詰まりを考慮すると 6〜12 並列が実用上の目安）。

### 精度検証

クラウド（Workers AI）とオンプレ（7900XTX）で判定結果に差はなく、確率の差も最大で **0.012** 程度に収まりました。

---

## コスト試算

Cloudflare Workers AI (`@cf/cloudflare/clef-flash`) の利用コストは非常に格安です。

* **料金体系**: 入力 $0.09 / 100万トークン（出力は無料）
* **消費量**: 1枚あたり約 450 トークン

### 実画像1,494枚でのコスト実績

* **2問分類**: 約 **12 円**
* **alt属性生成用（4問分類）**: 約 **20 円** （全画像を通しても約 $0.2 程度）

---

## まとめ

![](/images/jev/jev_2.png)

* **Clef-flashの使い勝手**: 従来のCLIPのような埋め込みベクトル同士の比較と異なり、質問に対する「はい/いいえ」の確率値が直接返るため、画像分類のしきい値調整やロジック構築が非常に簡単でした。
* **オンプレ vs クラウド**: 精度面では差がないものの、クラウド（Workers AI）なら並列リクエストを投げるだけでバッチ処理を圧倒的に高速化（1分台）できます。コストも1,500枚で10〜20円程度と極めて安価です。
* **運用面**: プロンプトの工夫（例: 「人が写っているか」→「着用しているか」）と、確認用UIをセットで運用することで、誤検知を限りなくゼロに抑えた画像ラベル付けパイプラインを構築できました。

## 85-Store について

富山県南砺市井波の古着・セレクトショップです。アメリカ・ヨーロッパの古着と、River・VOIRY・SOWBOW などのブランドを扱っています。2階は共創スペース「85-UpStore」です。

- 住所: 富山県南砺市本町4丁目100
- 営業時間: 12:00〜18:00（木曜定休。営業日は[サイト](https://85-store.com)のカレンダーで）
- オンラインストア: [shop.85-store.com](https://shop.85-store.com)（1万円以上で送料無料）
- Instagram: [@85store_inami](https://www.instagram.com/85store_inami/)

この記事で分類した商品写真は、オンラインストアで見られます。井波に来たときは、ぜひお店にも寄ってください。
