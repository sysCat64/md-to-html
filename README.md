# md-to-html

Markdownファイルを読み、内容の性質(表・時系列・比較・階層・コード中心など)に応じて最適なレイアウトを都度設計し、単一の自己完結HTML(インラインCSS+軽量インラインJS)として出力するClaude Codeプラグインです。

## 概要

固定のテンプレートにMarkdownを流し込む変換ツールではなく、ドキュメントの構造をそのつど読み取り、最も内容に合ったレイアウトをLLM自身が設計するという思想に基づいています。この発想はAnthropicのブログ記事 [The unreasonable effectiveness of HTML](https://claude.com/blog/using-claude-code-the-unreasonable-effectiveness-of-html) に由来しており、静的なコンバータでは得られない、内容ごとに最適化された表現力を狙っています。

## インストール

GitHub公開後は、以下のコマンドでマーケットプレイス経由でインストールできます。

```
/plugin marketplace add <this-repo>
/plugin install md-to-html
```

ローカルで試用する場合は、このリポジトリをクローンした状態のディレクトリをそのままプラグインディレクトリとして読み込ませることで動作を確認できます。

なお、本プラグインは `skills/` ディレクトリをClaude Codeが自動発見する仕組みを利用しており、`plugin.json` へのスキル登録記述は不要です(仕様どおりの構成です)。

## 使い方

```
/md-to-html path/to/file.md
/md-to-html path/to/file.md --theme dark
```

スラッシュコマンドを使わず、会話の中で「このmdをHTMLにして」「Markdownをきれいなページにして」のように依頼しても自動的に発火します。

## レイアウト判定の概要

Markdownの特徴量(テーブル数・コード比率・見出し階層・時系列シグナル・比較シグナルなど)を抽出し、以下の優先順位カスケード(上が優先)で6種類のレイアウトから1つを選定します。

| レイアウト | 主な発火条件 |
|---|---|
| 2〜3カラムのカード比較 | 比較シグナルがあり、比較対象が2〜3個に明確化できる |
| タイムライン/ステッパー | Step・連番などの時系列シグナルが文書の骨格(5個以上連続) |
| コードカード | コード面積30%以上、またはコードブロック5個以上 |
| ダッシュボード風グリッド/テーブルカード | テーブル3個以上、またはテーブル面積40%以上 |
| サイドバーナビ+アコーディオン(またはタブ) | 最大階層h3以上 かつ 見出し8個以上 |
| シンプル縦スクロール記事 | 上記いずれも非該当 |

判定がどの条件にも強く当てはまらない場合は、シンプル記事を基調に該当箇所だけ局所的に強調するハイブリッド構成にフォールバックします。判定ロジックの詳細な手順(特徴量の数え方、優先順位の理由)は [`skills/md-to-html/SKILL.md`](skills/md-to-html/SKILL.md) のSYNC-BLOCKに記載しています。

## テーマのカスタマイズ

生成されるHTMLは、[`skills/md-to-html/templates/theme-light.css`](skills/md-to-html/templates/theme-light.css) と [`theme-dark.css`](skills/md-to-html/templates/theme-dark.css) の `:root` ブロックをそのままインライン展開して使用します。両テーマは以下のCSS変数を同名・同数で定義するというコントラクトを守っており、生成HTML側は色・フォント・余白の値をこの変数経由でのみ参照します。

- `--bg`, `--surface`, `--text`, `--text-muted`, `--border`, `--accent`, `--accent-contrast`, `--code-bg`
- `--font-sans`, `--font-mono`, `--line-height`, `--letter-spacing`
- `--space-unit`, `--radius`, `--content-max-width`

配色や余白を変更したい場合は、これらのCSSファイルの値を編集するだけで、生成されるすべてのHTMLに反映されます。

## ディレクトリ構成

```
.claude-plugin/plugin.json          プラグインマニフェスト
skills/md-to-html/SKILL.md          Claude Code用スキル定義(判定ロジック含む)
skills/md-to-html/templates/        テーマCSS(light / dark)
.agent/skills/md-to-html/INSTRUCTIONS.md  ツール非依存の判定ロジック正本
evals/evals.json                    評価ケース定義
evals/samples/                      評価用サンプルMarkdown
```

## マルチエージェント対応

判定ロジックの正本は [`.agent/skills/md-to-html/INSTRUCTIONS.md`](.agent/skills/md-to-html/INSTRUCTIONS.md) にあり、Claude Code用の `SKILL.md` はこのファイルとSYNC-BLOCK部分を完全一致させる運用としています。Codex CLIやAntigravity等、他ツール向けのラッパー追加は今後の対応予定です。

## ライセンス

MIT
