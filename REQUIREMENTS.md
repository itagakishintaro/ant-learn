# Anthropic最新情報学習スキル - 要件定義書

## 概要

Claude Codeのスラッシュコマンド（スキル）として実装する、Anthropic公式サイトの最新情報を抽出・学習できるツール。
SAとしてお客様にわかりやすく情報を伝えるための知識習得を目的とする。

## ゴール

Anthropicのソリューションアーキテクトが、最新の機能・発表をキャッチアップし、お客様に正確かつわかりやすく伝えられるようになること。

---

## 機能要件

### 1. スラッシュコマンド

| 項目 | 内容 |
|------|------|
| コマンド名 | `/ant-learn` |
| 実行方法 | Claude Codeで `/ant-learn` を入力 |
| 引数 | `--list`（履歴一覧）、`--review {id}`（復習）、`--reset`（履歴リセット） |

### 2. トピック抽出

- **情報源**: 以下の公式サイト配下のコンテンツのみ（外部サイト・SNS等は対象外）
  - https://www.anthropic.com/news — ニュース・発表
  - https://www.anthropic.com/research — リサーチ
  - https://www.anthropic.com/engineering — エンジニアリングブログ
  - https://code.claude.com/docs/ja — Claude Code公式ドキュメント
- **抽出対象**: 新機能リリース、モデルアップデート、API変更、安全性に関する発表、パートナーシップ等
- **抽出方法**: WebFetchツールを使用して https://www.anthropic.com/ 配下のページのみから情報を取得
- **優先度**: 新しいトピックを優先。新しいトピックがない場合は、重要だが未学習のトピックを提示

### 3. 学習コンテンツの構成

1トピックにつき、以下の構成で解説を提供する：

```
# [トピック名]

## 概要
- トピックの簡潔な要約（2-3文）

## 背景・経緯
- なぜこの機能/発表が生まれたのか
- 業界のトレンドや課題との関連
- Anthropicの思想・ミッションとの関連

## 詳細解説
- 技術的な詳細
- 具体的な機能・仕様
- 従来との違い・改善点

## お客様への説明ポイント
- SAとしてお客様に伝えるべきキーメッセージ
- 想定されるお客様の質問とその回答（Q&A形式）
- ユースケース・活用例

## 関連トピック
- このトピックを理解するために知っておくべき前提知識
- 関連する過去のリリース・機能
```

### 4. 履歴管理・コンテンツ保存

| 項目 | 内容 |
|------|------|
| 履歴保存先 | `~/.claude/ant-learn/history.json` |
| コンテンツ保存先 | `~/.claude/ant-learn/contents/{id}.md` |
| 保存内容（履歴） | トピック名、URL、学習日時、概要 |
| 保存内容（コンテンツ） | 生成した学習コンテンツ全文（Markdown） |
| 重複排除 | 履歴にあるトピックは提示しない |
| 復習 | `--review {id}` で保存済みコンテンツを再表示 |
| リセット | `--reset` で履歴・コンテンツを全削除 |

### 5. フォールバック

- 最新トピックがない場合（全て学習済み等）:
  - Anthropicの重要な基盤技術・思想から未学習のものを提示
  - 例: Constitutional AI、RSP、MCP、Tool Use、Claude Code、Extended Thinking等
  - 「トピックがない」という状態は発生させない

---

## 非機能要件

| 項目 | 内容 |
|------|------|
| 実行環境 | Claude Code（ローカル） |
| 外部依存 | なし（Claude Code内蔵ツールのみ使用） |
| データ永続化 | ローカルファイル（JSON + Markdown） |
| 言語 | 日本語で出力 |
| スキルスコープ | プロジェクトスコープ（`.claude/skills/ant-learn/`） |

---

## 技術設計

### スキル構成

```
<project>/.claude/skills/ant-learn/
└── SKILL.md          # スキル定義（スラッシュコマンドの実装）
```

### データファイル構成

```
~/.claude/ant-learn/
├── history.json              # 学習履歴（インデックス）
└── contents/                 # 学習コンテンツ保存先
    ├── claude-opus-4-6.md
    ├── building-effective-agents.md
    └── ...
```

### history.json スキーマ

```json
{
  "topics": [
    {
      "id": "claude-opus-4-6",
      "title": "Claude Opus 4.6",
      "url": "https://www.anthropic.com/news/claude-opus-4-6",
      "category": "news",
      "learned_at": "2026-04-08T10:00:00Z",
      "summary": "概要テキスト..."
    }
  ]
}
```

---

## 実行フロー

```
ユーザーが /ant-learn を実行
    │
    ├─ --list → 学習済みトピック一覧を表示
    ├─ --review {id} → 保存済みコンテンツを表示（復習）
    ├─ --reset → 履歴・コンテンツを削除してリセット
    │
    └─ 引数なし（新規学習）
        │
        ▼
    履歴ファイル（history.json）を読み込む
        │
        ▼
    Anthropic公式サイトから記事一覧を取得
    （WebFetch: /news, /research, /engineering, code.claude.com/docs/ja）
        │
        ▼
    取得したトピック一覧から履歴にないものを抽出
        │
        ├─ 未学習トピックあり → 最新の1つを選択
        └─ 全て学習済み → フォールバックリストから選択
            │
            ▼
      個別記事ページをWebFetchで取得
            │
            ▼
      学習コンテンツを生成・表示
            │
            ▼
      コンテンツをMarkdownファイルに保存
            │
            ▼
      履歴ファイルに記録
```

---

## 今後の拡張案（スコープ外）

- 複数トピックの一括学習（`--count 3` 等）
- トピックのカテゴリ別フィルタ（`--category api` 等）
- クイズ形式での理解度チェック
- チーム内共有機能
