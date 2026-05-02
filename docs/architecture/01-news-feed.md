# F-01 ニュースフィード アーキテクチャ設計

## このドキュメントの目的

技術選定（`docs/tech-selection/01-news-feed.md`）の決定を受け、
**何をどう組み合わせて動かすか** の構造を定義する。
設計上の分岐点では「なぜそちらを選んだか」を記録する。

---

## システム全体像

```
[GitHub Actions: cron 毎朝7:00 JST]
         │
         ▼
┌─────────────────────────────────────────────┐
│  main.py（エントリーポイント）               │
│                                             │
│  1. CategoryConfig を読み込む               │
│  2. ExclusionRuleManager で除外条件を読み込む│
│  3. NewsCollector でカテゴリごとに記事収集   │
│  4. ArticleFilter で除外条件に合う記事を除去 │
│  5. ArticleSummarizer で各記事を要約         │
│  6. NewsletterBuilder でレイアウト構築       │
│  7. SlackNotifier で送信                    │
└─────────────────────────────────────────────┘
         │
         ▼
   [Slack Webhook]
```

---

## コンポーネント一覧

| コンポーネント | ファイル | 責務 |
|--------------|---------|------|
| NewsCollector | `src/collector.py` | Google News RSS / GNews API から記事を取得する |
| ArticleFilter | `src/filter.py` | Claude API で除外条件に合致する記事を除去する |
| ArticleSummarizer | `src/summarizer.py` | Claude API で各記事を1〜2文に要約する |
| ExclusionRuleManager | `src/exclusion.py` | `data/exclusion-rules.json` を読み書きする |
| NewsletterBuilder | `src/newsletter.py` | フィルタ・要約済み記事からSlack送信テキストを組み立てる |
| SlackNotifier | `src/notifier.py` | Slack Webhook に POST する |
| データモデル | `src/models.py` | Article・ExclusionRule・CategoryConfig の Pydantic モデル |
| エントリーポイント | `src/main.py` | 上記を呼び出す実行フロー |
| フィードバックCLI | `scripts/add_exclusion.py` | 手動で除外条件を追加するスクリプト |

---

## データフロー（詳細）

### Step 1: 設定の読み込み

```
data/categories.json
    └─→ CategoryConfig (カテゴリID・名前・クエリ・上限件数)

data/exclusion-rules.json
    └─→ list[ExclusionRule] (除外条件リスト)
```

### Step 2: ニュース収集（NewsCollector）

カテゴリごとに以下を実行する：

```
Category.queries = ["ChatGPT 新機能", "Claude 新機能", ...]
    │
    ├─ Google News RSS: https://news.google.com/rss/search?q={query}&hl=ja&gl=JP
    │      └─ feedparser でパース → list[Article]
    │
    └─ GNews API（補助）: 英語クエリに限定して使用
           └─ httpx でGET → list[Article]

重複除去（URL一致）→ 公開日の新しい順にソート → Category.max_articles 件に絞る
```

**設計判断**: 1カテゴリあたり記事数を上限で絞るのは収集段階ではなくソート後にする。
理由: フィルタリングで除外されると最終件数が少なくなりすぎるため、
収集段階では多めに取り（上限の2〜3倍）、フィルタ後に再度上限を適用する。

### Step 3: フィルタリング（ArticleFilter）

**2ステップ方式を採用する**（コスト最適化のため）:

```
Step 3-1: タイトルのみで除外判断（Haiku、低コスト）
  入力: 記事タイトル一覧 + 除外条件リスト
  出力: 除外すべき記事のインデックスリスト
  → 対象記事を除去

Step 3-2: 残った記事のみ要約（Haiku）
  入力: フィルタ済み記事の概要テキスト
  出力: 1〜2文の日本語要約
```

**なぜ2ステップか**:
- 全記事の全文をLLMに渡すとトークン消費が多く、コストが増える
- タイトルだけなら1記事あたり20〜50トークン程度
- 除外判断はタイトルで十分な精度が出る

**プロンプト設計方針（フィルタ）**:
```
以下の記事タイトルリストを確認し、除外条件に該当する記事の番号をJSON配列で返してください。

除外条件:
{exclusion_rules}

記事リスト:
{numbered_title_list}

回答形式: {"excluded_indices": [1, 3, ...]}
除外条件に該当しない場合は空配列を返してください。
```

### Step 4: 要約（ArticleSummarizer）

```
入力: フィルタ済み記事（タイトル + RSS概要テキスト）
出力: 1〜2文の日本語要約

バッチ処理: 1カテゴリの記事をまとめて1回のAPI呼び出しで要約する（API呼び出し回数削減）
```

### Step 5: ニュースレター構築（NewsletterBuilder）

Slackのメッセージ形式（Block Kit）で構築する：

```
📰 パーソナルニュース - 2026-05-02 (土)

━━━━━━━━━━━━━━━━━━
🤖 AI最新動向
━━━━━━━━━━━━━━━━━━
• *Claude 3.7 新機能発表* — Anthropicが新モデルを発表。〇〇の点が改善された。
  https://...

• *GPT-5 リリース情報* — OpenAIが〇〇を発表。〇〇の点で注目される。
  https://...

（以下、カテゴリごとに繰り返し）
```

### Step 6: Slack送信（SlackNotifier）

```python
POST https://hooks.slack.com/services/...
Content-Type: application/json

{
  "text": "...",
  "blocks": [...]  # Block Kit形式（オプション）
}
```

エラー時: HTTPステータスが2xx以外なら例外を発生させて、GitHub Actionsの失敗通知を受け取る。

---

## データモデル

### Article

```python
class Article(BaseModel):
    id: str                    # URLのSHA256（先頭8文字）
    title: str
    url: str
    raw_summary: str           # RSSの概要テキスト（未加工）
    ai_summary: str | None     # LLMによる要約（フィルタ後に追加）
    published_at: datetime
    source: str                # メディア名
    category_id: str
```

### ExclusionRule

```python
class ExclusionRule(BaseModel):
    id: str                          # "rule_" + タイムスタンプ
    description: str                 # 人間可読な説明（例: "中国の規制ニュース"）
    keywords: list[str] = []         # 補助情報（LLMが参考にする）
    added_at: datetime
    source_article_title: str | None # きっかけになった記事タイトル
```

### CategoryConfig

```python
class Category(BaseModel):
    id: str
    name: str                   # 表示名（例: "AI最新動向"）
    queries: list[str]          # 検索クエリ一覧
    max_articles: int = 5       # 最終的な配信件数上限
    collect_multiplier: int = 3 # 収集段階では max_articles × この値を取得

class CategoryConfig(BaseModel):
    categories: list[Category]
```

---

## 設定ファイル

### `data/categories.json`（初期値）

```json
{
  "categories": [
    {
      "id": "ai_news",
      "name": "AI最新動向",
      "queries": ["ChatGPT 新機能", "Gemini 新機能", "Claude 新機能", "生成AI 最新"],
      "max_articles": 5,
      "collect_multiplier": 3
    },
    {
      "id": "tech_news",
      "name": "テック業界",
      "queries": ["テクノロジー スタートアップ", "IT 最新ニュース"],
      "max_articles": 3,
      "collect_multiplier": 3
    }
  ]
}
```

### `data/exclusion-rules.json`（初期値）

```json
{
  "exclusion_rules": []
}
```

### `.env.example`

```
ANTHROPIC_API_KEY=your_key_here
GNEWS_API_KEY=your_key_here
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
```

---

## ディレクトリ構成

```
personal-news-service/
├── .github/
│   └── workflows/
│       └── daily-news.yml          # GitHub Actions: cron定時実行
├── src/
│   ├── __init__.py
│   ├── main.py                     # エントリーポイント
│   ├── models.py                   # Pydanticモデル
│   ├── collector.py                # NewsCollector
│   ├── filter.py                   # ArticleFilter
│   ├── summarizer.py               # ArticleSummarizer
│   ├── exclusion.py                # ExclusionRuleManager
│   ├── newsletter.py               # NewsletterBuilder
│   └── notifier.py                 # SlackNotifier
├── scripts/
│   └── add_exclusion.py            # 除外条件を手動追加するCLI
├── data/
│   ├── categories.json             # カテゴリ設定（Gitで管理）
│   └── exclusion-rules.json        # 除外条件（Gitで管理）
├── tests/
│   ├── test_collector.py
│   ├── test_filter.py
│   ├── test_summarizer.py
│   └── test_newsletter.py
├── docs/                           # 設計ドキュメント（本ドキュメント等）
├── .env.example
├── .gitignore
├── README.md
└── pyproject.toml                  # 依存管理（uv）
```

---

## GitHub Actions ワークフロー

### `.github/workflows/daily-news.yml`（概要）

```yaml
name: Daily News

on:
  schedule:
    - cron: '0 22 * * *'   # UTC 22:00 = JST 07:00
  workflow_dispatch:         # 手動実行も可能にする

jobs:
  send-newsletter:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync
      - run: uv run python -m src.main
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GNEWS_API_KEY: ${{ secrets.GNEWS_API_KEY }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**設計判断: `workflow_dispatch` を必ず入れる**
理由: 開発・デバッグ時に手動実行できないと、実際の動作確認のたびにcronを待つ必要がある。

---

## エラーハンドリング方針

| 障害ケース | 対応方針 |
|-----------|---------|
| RSSフィード取得失敗 | そのクエリをスキップして処理継続。失敗クエリをログに出力 |
| GNews API 取得失敗（レート制限等） | GNews をスキップしてRSSのみで処理継続 |
| Claude API 呼び出し失敗 | 最大1回リトライ。それでも失敗なら要約なし（タイトルのみ）で配信 |
| フィルタリング失敗 | フィルタをスキップして全記事を通す（除外されないほうが安全） |
| Slack 送信失敗 | 例外を上げてActionsを失敗させる（失敗メール通知を受け取る） |

**設計判断: 障害時は「止めるより届ける」を優先する**
理由: 毎朝のニュースは多少品質が落ちても届くほうが価値がある。
完全に失敗するのはSlack送信失敗など致命的なケースのみに限定する。

---

## フィードバック（除外条件追加）の運用

### MVP: 手動CLIによる追加

```bash
# 記事URLと理由を指定して除外条件を追加
python scripts/add_exclusion.py \
  --title "中国AI規制強化の記事" \
  --description "中国の規制・政策ニュース" \
  --keywords "中国,規制,政策"

# 除外条件の一覧を表示
python scripts/add_exclusion.py --list

# 除外条件を削除
python scripts/add_exclusion.py --delete rule_001
```

実行後、`data/exclusion-rules.json` が更新される。Gitにコミットすることで次回以降の実行に反映される。

### 将来拡張（MVP後に検討）

Slackメッセージに 👎 リアクションをつけると自動的に除外条件が追加される仕組みを検討する。
これはSlack Events APIとGitHub Actions（または別のWebhookサーバー）との連携が必要になる。

---

## 未解決・要確認事項

| 項目 | 内容 | 判断タイミング |
|------|------|--------------|
| Google News RSS の利用規約 | 商用利用の制限があるか確認が必要（個人利用なので問題ない可能性が高い） | 実装前 |
| GNews API の日本語精度 | 日本語クエリでの記事品質を実際に試して確認する | 実装中 |
| Slack Block Kit の使用 | シンプルなtextで十分か、Block Kitが必要かは実装しながら決める | 実装中 |
| `data/` ファイルのGit管理 | 除外条件をコミットするか、別ストレージ（GitHub Gist等）にするか | 実装中 |

---

## 次のステップ

**F-01 Step 4: リポジトリ初期化・環境構築**

- `pyproject.toml` の作成（uv、依存ライブラリの定義）
- ディレクトリ構成の骨格を作成
- `.env.example`・`.gitignore`・`README.md` の作成
- `data/categories.json`・`data/exclusion-rules.json` の初期ファイル作成
- GitHub Actions ワークフローファイルの作成
