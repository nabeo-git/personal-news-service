# personal-news-service PLAN.md

## このドキュメントの目的

**何を・どの順で・何を作りながら進めるか** を定義する。
技術的な詳細は各ドキュメントに委譲し、このファイルでは「進め方」だけを管理する。

---

## プロジェクト概要

個人向けのパーソナライズ情報配信サービス。詳細: [`docs/requirements/00-project-overview.md`](docs/requirements/00-project-overview.md)

### 3つの機能

| 機能ID | 機能名 | 実装順 |
|--------|--------|--------|
| F-01 | ニュースフィード | **最初（MVP）** |
| F-02 | 金融インサイト | F-01完成後 |
| F-03 | タスク通知 | F-02完成後 |

---

## 進め方の原則

- **1機能ずつ完成させる** — F-01が動くまでF-02・F-03に手を付けない
- **各ステップで成果物ドキュメントを作る** — 実装より先にドキュメントを書き、合意してから実装に進む
- **スコープを広げない** — 思いついたアイデアは「未決定事項」に記録して後回しにする

---

## ドキュメントマップ

### 作成済み

| ドキュメント | パス | 内容 |
|------------|------|------|
| プロジェクト概要 | [`docs/requirements/00-project-overview.md`](docs/requirements/00-project-overview.md) | スコープ・MVP定義・全体方針 |
| F-01 要件定義 | [`docs/requirements/01-news-feed.md`](docs/requirements/01-news-feed.md) | ユーザーストーリー・機能要件・受け入れ基準 |
| F-02 要件定義 | [`docs/requirements/02-financial-insights.md`](docs/requirements/02-financial-insights.md) | 概要要件（詳細化は後で） |
| F-03 要件定義 | [`docs/requirements/03-task-notifications.md`](docs/requirements/03-task-notifications.md) | 概要要件（詳細化は後で） |
| 市場調査（参考） | [`docs/research/market-research.md`](docs/research/market-research.md) | 競合・技術動向の参考情報 |
| 初期仕様書（原本） | [`初期構築仕様書.txt`](初期構築仕様書.txt) | 最初期のアイデアメモ（参照のみ） |

### 未作成（今後のステップで作る）

| ドキュメント | パス | 作成タイミング |
|------------|------|--------------|
| F-01 技術選定 | `docs/tech-selection/01-news-feed.md` | F-01 Step 2 |
| F-01 アーキテクチャ設計 | `docs/architecture/01-news-feed.md` | F-01 Step 3 |
| F-02 技術選定 | `docs/tech-selection/02-financial-insights.md` | F-02 Step 2 |
| F-02 アーキテクチャ設計 | `docs/architecture/02-financial-insights.md` | F-02 Step 3 |
| F-03 技術選定 | `docs/tech-selection/03-task-notifications.md` | F-03 Step 2 |
| F-03 アーキテクチャ設計 | `docs/architecture/03-task-notifications.md` | F-03 Step 3 |

---

## タスク一覧

### 共通フェーズ（完了）

- [x] プロジェクト概要・スコープ定義
- [x] 3機能の要件定義（概要レベル）
- [x] MVPの決定（F-01を最初に実装）

---

### F-01 ニュースフィード（MVP）

#### Step 1: 要件定義
- [x] ユーザーストーリーの定義
- [x] 機能要件の定義
- [x] 受け入れ基準の定義
- [x] 未決定事項の洗い出し
- **成果物**: [`docs/requirements/01-news-feed.md`](docs/requirements/01-news-feed.md)

#### Step 2: 技術選定
- [ ] ニュース取得方法の比較（RSS / News API / スクレイピング）
- [ ] LLMの選定（コスト・機能の比較）
- [ ] 配信手段の選定（メール / Slack）
- [ ] スケジューラーの選定（cron / Lambda / n8n）
- [ ] 除外条件の保存方法の選定
- **成果物**: `docs/tech-selection/01-news-feed.md`

#### Step 3: アーキテクチャ設計
- [ ] コンポーネント図の作成
- [ ] データフローの設計
- [ ] データモデル設計（除外条件の構造）
- [ ] ディレクトリ構成の決定
- **成果物**: `docs/architecture/01-news-feed.md`

#### Step 4: リポジトリ初期化・環境構築
- [ ] 基本ディレクトリ構成の作成
- [ ] `.env.example` の作成
- [ ] `README.md` の作成
- [ ] 開発環境のセットアップ手順確認

#### Step 5: 実装
- [ ] ニュース収集モジュール
- [ ] AIによる記事フィルタリング・要約
- [ ] 除外条件の保存・適用ロジック
- [ ] 配信モジュール（メール or Slack）
- [ ] スケジューラー設定

#### Step 6: テスト・動作確認
- [ ] 受け入れ基準の全項目を確認
- [ ] 実際に数日間運用して品質確認

---

### F-02 金融インサイト（F-01完成後）

- [ ] Step 1: 要件の詳細化
- [ ] Step 2: 技術選定
- [ ] Step 3: アーキテクチャ設計
- [ ] Step 4: 実装
- [ ] Step 5: テスト

---

### F-03 タスク通知（F-02完成後）

- [ ] Step 1: 要件の詳細化
- [ ] Step 2: 技術選定
- [ ] Step 3: アーキテクチャ設計
- [ ] Step 4: 実装
- [ ] Step 5: テスト

---

## 現在のステータス

**F-01 Step 2（技術選定）が次のアクション**

```
共通フェーズ ████████████ 完了
F-01 Step 1  ████████████ 完了
F-01 Step 2  ░░░░░░░░░░░░ 未着手 ← ここ
F-01 Step 3  ░░░░░░░░░░░░ 未着手
F-01 Step 4  ░░░░░░░░░░░░ 未着手
F-01 Step 5  ░░░░░░░░░░░░ 未着手
F-01 Step 6  ░░░░░░░░░░░░ 未着手
F-02 ...     ░░░░░░░░░░░░ 未着手
F-03 ...     ░░░░░░░░░░░░ 未着手
```

---

## 未決定・後回し事項

（思いついたアイデアをここに記録して、スコープ外として管理する）

| アイデア | 検討タイミング |
|---------|--------------|
| マルチユーザー対応 | F-01完成後に判断 |
| Webダッシュボード | F-01完成後に判断 |
| F-01〜F-03の統合配信 | F-03着手時に設計 |
| モバイルアプリ | サービス全体完成後 |
