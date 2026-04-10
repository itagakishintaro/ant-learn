---
name: ant-learn
description: Anthropic公式サイト(https://www.anthropic.com/)およびClaude Code公式ドキュメント(https://code.claude.com/docs/ja)の最新情報を学習するスキル。トピックの背景・詳細・お客様説明ポイントを含む学習コンテンツを生成する。実行するたびに新しいトピックを1つ提示し、学習済みのものは重複しない。`/ant-learn` で新規学習、`/ant-learn --list` で履歴一覧、`/ant-learn --review {id}` で復習、`/ant-learn --reset` で履歴リセット。
argument-hint: [--list] [--review <id>] [--reset]
---

## あなたのタスク

Anthropic公式サイトおよびClaude Code公式ドキュメントから最新トピックを1つ抽出し、SAがお客様に説明できるレベルの学習コンテンツを生成してください。

## 引数による分岐

引数（`$ARGUMENTS`）に応じて処理を分岐してください：

### `--reset` の場合
1. `~/.claude/ant-learn/history.json` と `~/.claude/ant-learn/contents/` ディレクトリを削除する
2. 「学習履歴をリセットしました」と表示して終了

### `--list` の場合
1. `~/.claude/ant-learn/history.json` を読み込む
2. 学習済みトピックの一覧を以下の形式で表示して終了:
```
| # | トピック | カテゴリ | 学習日 | ID |
|---|---------|---------|--------|-----|
| 1 | Claude Opus 4.6 | news | 2026-04-08 | claude-opus-4-6 |
```
3. 復習するには `/ant-learn --review {id}` を使うよう案内する

### `--review {id}` の場合
1. `~/.claude/ant-learn/contents/{id}.md` を読み込んで表示する
2. ファイルが存在しない場合は「指定されたトピックが見つかりません。`/ant-learn --list` で利用可能なIDを確認してください」と表示

### 引数なしの場合（新規学習）
以下の「新規学習フロー」に従う

## 新規学習フロー

### Step 1: 履歴読み込み

`~/.claude/ant-learn/history.json` をReadツールで読み込む。ファイルが存在しない場合は `{"topics": []}` として扱う。

### Step 2: トピック一覧の取得

WebFetchツールで以下の4ページを取得し、記事・ドキュメントのリンクとタイトルを抽出する:

1. `https://www.anthropic.com/news` — prompt: 「このページに掲載されている全ての記事のタイトル、URL（hrefの値）、日付を一覧で抽出してください。URLは /news/xxx 形式のものだけを対象とし、/research/team/ のようなチームページは除外してください。日付が新しい順にリストしてください。」

2. `https://www.anthropic.com/research` — prompt: 「このページに掲載されている全ての記事のタイトル、URL（hrefの値）、日付を一覧で抽出してください。URLは /research/xxx 形式の個別記事のみ対象とし、/research/team/ のようなチームページは除外してください。日付が新しい順にリストしてください。」

3. `https://www.anthropic.com/engineering` — prompt: 「このページに掲載されている全ての記事のタイトル、URL（hrefの値）、日付を一覧で抽出してください。URLは /engineering/xxx 形式のものだけを対象にしてください。日付が新しい順にリストしてください。」

4. `https://code.claude.com/docs/ja` — prompt: 「このページのサイドバーまたはナビゲーションに掲載されている全てのドキュメントページのタイトルとURL（hrefの値）を一覧で抽出してください。URLは /docs/ja/xxx 形式のものだけを対象にしてください。アンカーリンク（#付き）は除外してください。」

### Step 3: 未学習トピックの選択

1. 取得した全記事を日付の新しい順に並べる
2. 履歴（history.json の topics[].url）に既に存在するものを除外する
3. 残ったものから最も新しい1件を選択する
4. **全て学習済みの場合**: 後述の「フォールバックトピックリスト」から、履歴にないものを1つ選択する

### Step 4: トピック詳細の取得

選択したトピックの個別ページを WebFetch で取得する:
- anthropic.comの記事の場合:
  - URL: `https://www.anthropic.com{記事パス}`
  - prompt: 「この記事の全文を詳細に抽出してください。タイトル、公開日、本文の全内容を含めてください。」
- code.claude.comのドキュメントの場合:
  - URL: `https://code.claude.com{ドキュメントパス}`
  - prompt: 「このドキュメントの全文を詳細に抽出してください。タイトル、本文の全内容を含めてください。」

### Step 5: 学習コンテンツの生成

取得した記事内容を元に、以下のフォーマットで学習コンテンツを生成し表示する。
SAがお客様に説明する際に役立つ実践的な内容にすること。

```markdown
# {トピック名}

> **カテゴリ**: {news/research/engineering/claude-code-docs}
> **公開日**: {日付}
> **URL**: {記事URL}

## 概要
- トピックの簡潔な要約（2-3文）

## 背景・経緯
- なぜこの機能/発表が生まれたのか
- 業界のトレンドや課題との関連
- Anthropicの思想・ミッションとの関連（安全性、有用性のバランス等）

## 詳細解説
- 技術的な詳細
- 具体的な機能・仕様
- 従来との違い・改善点
- 数値データやベンチマーク結果（あれば）

## お客様への説明ポイント
- SAとしてお客様に伝えるべきキーメッセージ（3-5個の箇条書き）
- 想定されるお客様の質問と回答（Q&A形式で3つ以上）
- 具体的なユースケース・活用例

## 関連トピック
- このトピックを理解するために知っておくべき前提知識
- 関連する過去のリリース・機能へのリンク
```

### Step 6: コンテンツの保存

1. トピックIDを記事URLのパスから生成する（例: `/news/claude-opus-4-6` → `claude-opus-4-6`、`/docs/ja/hooks` → `hooks`）
2. 生成した学習コンテンツを `~/.claude/ant-learn/contents/{id}.md` にWriteツールで保存する
3. `~/.claude/ant-learn/history.json` に以下のエントリを追加してWriteツールで保存する:
```json
{
  "id": "{id}",
  "title": "{トピック名}",
  "url": "{記事の完全URL}",
  "category": "{news|research|engineering|claude-code-docs}",
  "learned_at": "{現在日時 ISO8601}",
  "summary": "{概要セクションの内容}"
}
```

### Step 7: 完了メッセージ

以下を表示する:
```
---
学習コンテンツを保存しました: ~/.claude/ant-learn/contents/{id}.md
復習するには: /ant-learn --review {id}
学習履歴の確認: /ant-learn --list
```

## フォールバックトピックリスト

公式サイトの全記事・ドキュメントが学習済みの場合に使用する。これらのトピックについては、WebFetchで `https://www.anthropic.com` または `https://code.claude.com/docs/ja` 配下の関連ページを取得して情報を収集し、学習コンテンツを生成すること。

1. **Constitutional AI** — Anthropicの根幹思想。AIの安全性を憲法的なルールで担保するアプローチ
   - 参考URL: `https://www.anthropic.com/research/constitutional-ai-harmlessness-from-ai-feedback`
2. **Responsible Scaling Policy (RSP)** — AIの能力に応じた安全対策を段階的に強化するフレームワーク
   - 参考URL: `https://www.anthropic.com/news/announcing-our-updated-responsible-scaling-policy`
3. **Model Context Protocol (MCP)** — AIモデルと外部ツール・データソースの接続を標準化するオープンプロトコル
   - 参考URL: `https://www.anthropic.com/news/model-context-protocol`
4. **Tool Use / Function Calling** — Claude APIでの外部ツール連携機能
   - 参考URL: `https://www.anthropic.com/engineering/tool-use`
5. **Claude Code** — CLIベースのAIコーディングアシスタント
   - 参考URL: `https://www.anthropic.com/engineering/claude-code-best-practices`
6. **Extended Thinking** — Claudeの拡張思考機能。複雑な問題を段階的に推論する
   - 参考URL: `https://www.anthropic.com/engineering/claude-think-tool`
7. **Prompt Engineering Best Practices** — Claudeを最大限活用するためのプロンプト設計手法
   - 参考URL: `https://www.anthropic.com/engineering/prompting-guide-for-claude`
8. **Contextual Retrieval** — RAGの精度を大幅に改善するコンテキスト付き検索手法
   - 参考URL: `https://www.anthropic.com/engineering/contextual-retrieval`
9. **Building Effective Agents** — AIエージェント構築のベストプラクティス
   - 参考URL: `https://www.anthropic.com/engineering/building-effective-agents`
10. **Interpretability Research** — AIモデルの内部動作を理解する解釈可能性研究
    - 参考URL: `https://www.anthropic.com/research/team/interpretability`

## 注意事項

- 情報源は `https://www.anthropic.com/` および `https://code.claude.com/docs/ja` 配下のコンテンツのみに限定すること（外部サイトは参照しない）
- 学習コンテンツは日本語で生成すること
- お客様への説明ポイントは、技術的に正確でありながら、非エンジニアのお客様にもわかるよう平易な表現を心がけること
- フォールバックトピックの場合も、必ずWebFetchで公式サイトの関連ページを取得してから学習コンテンツを生成すること
