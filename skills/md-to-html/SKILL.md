---
name: md-to-html
description: Converts any Markdown file into a single self-contained HTML page whose layout is chosen based on the document's actual content — tables become a dashboard/card grid, steps or dates become a timeline, comparisons become side-by-side columns, code-heavy docs become code cards, and everything else becomes a clean article layout. Use whenever the user wants to turn Markdown into HTML, visualize/render/prettify a .md file, or make a Markdown doc look good as a web page. 日本語のMarkdown(「このmdをHTMLにして」「Markdownをきれいなページにして」等)にも対応。
when_to_use: Markdownファイル(.md)をHTMLに変換・可視化・整形したいとき。「このmdをHTMLにして」「Markdownをきれいなページにして」等の依頼、または /md-to-html <path> の明示実行時。
argument-hint: "<path-to-markdown-file> [--theme light|dark]"
allowed-tools: Read, Write, Glob
---

# md-to-html

## 目的

このスキルは、決まったテンプレートにMarkdownを流し込むのではなく、対象ドキュメントの構造(表・時系列・比較・階層・コードのいずれが骨格になっているか)をそのつど判定し、その内容に最も適したレイアウトを持つ単一の自己完結HTMLファイルを設計・生成する。固定フォーマットのコンバータではなく、内容を読んでレイアウトを都度考えるLLM主導の変換であることが本スキルの前提であり、この判断を省略してはならない。

## 入力の解決

1. `$ARGUMENTS` を解析し、Markdownファイルのパスと `--theme light|dark` オプションを取り出す。`--theme` 省略時は `light` を既定値とする。
2. パスが曖昧・未指定の場合は、直前の会話文脈で言及されているMarkdownファイルを対象候補とする。
3. それでも特定できない場合はGlobで `*.md` を探索し、候補を絞り込む。
4. それでも一意に特定できない場合は、生成を進めずユーザーに確認する。
5. 対象ファイルが定まったら、Readで全文を読み込む。部分読みで判定しない。

## レイアウト判定ロジック

Markdown全文を読んだうえで、以下の手順に従ってレイアウトを判定する。この手順は `.agent/skills/md-to-html/INSTRUCTIONS.md` のSYNC-BLOCKと内容が完全一致している必要があり、どちらか一方だけを変更してはならない。

<!-- SYNC-BLOCK: layout-decision-logic v2 — keep identical in SKILL.md and INSTRUCTIONS.md -->
### Step A: 特徴量の抽出(生成前に必ずチェックリストとして列挙する)

Markdown本文を読んだら、HTMLを書き始める前に以下を数え上げ、判定材料として明示すること。

- テーブル数 / テーブル行が全体に占める割合
- コードブロック数 / コード行が全体に占める割合
- 見出し数と最大階層(h1〜h6)
- 時系列シグナル: 「Step N」「手順」、連続する番号付きリスト、日付パターン(YYYY-MM-DD、〇〇年〇月 等)
- 比較シグナル: 「vs」「比較」「メリット/デメリット」「Pros/Cons」、対象名が列見出しになった2〜3列テーブル

### Step B: しきい値による分類(数値は目安としてのガイド)

| レイアウト | 発火条件 |
|---|---|
| ダッシュボード風グリッド/テーブルカード | テーブル3個以上、またはテーブル面積40%以上 |
| タイムライン/ステッパー | 時系列シグナルが文書の骨格(Step・連番が5個以上連続し本文の主構造) |
| 2〜3カラムのカード比較 | 比較シグナルがあり、比較対象が2〜3個に明確化できる |
| サイドバーナビ+アコーディオン(またはタブ) | 最大階層h3以上 かつ 見出し8個以上 |
| コードカード(シンタックスハイライト風+行番号) | コード面積30%以上、またはコードブロック5個以上 |
| シンプル縦スクロール記事 | 上記いずれも非該当 |

### Step C: 複数該当時の優先順位カスケード

複数のレイアウトが同時に発火条件を満たす場合、以下の優先順位で1つを選ぶ(上位が下位を上書きする)。

1. 比較(読者の意図が最も明確。意味構造を最優先)
2. 時系列/手順(順序が本質。崩すと情報が壊れる)
3. コード中心(可読性要件が特殊)
4. 表主体
5. 深い階層(単独発火は最下位。他レイアウトと併用可 — 例: 比較レイアウト内のセクションをアコーディオン化)
6. シンプル記事

### Step D: 迷った場合のフォールバック

スコアが拮抗する・どの特徴も弱い場合は、シンプル記事レイアウトを基調に、該当する局所セクションのみ強調表現(表はテーブルカード化、手順部分のみミニステッパー化など)を適用するハイブリッドとする。誤判定で情報構造を歪めるより、素直な提示を優先する。

### Step E: 判定根拠の記録

選定したレイアウトと主要な判定根拠(特徴量)を、生成HTMLの生成マーカーコメントと完了報告の両方に必ず残す。

生成マーカーコメント内のレイアウト表記には、必ず以下の正規化レイアウトID(英語)を使用する。完了報告の説明文は日本語のままでよいが、マーカー内のIDは必ずこの英語IDにする(日本語のレイアウト名をマーカーに書いてはならない)。

| 正規化ID | レイアウト |
|---|---|
| `dashboard` | ダッシュボード風グリッド/テーブルカード |
| `timeline` | タイムライン/ステッパー |
| `comparison` | 2〜3カラムのカード比較 |
| `sidebar-nav` | サイドバーナビ+アコーディオン(またはタブ) |
| `code-cards` | コードカード(シンタックスハイライト風+行番号) |
| `article` | シンプル縦スクロール記事 |

Step Dのハイブリッド構成を選んだ場合は `article+<強調要素>` 形式(例: `article+table-cards`)とする。

マーカー例: `<!-- generated-by: md-to-html | layout: timeline | theme: dark -->`
<!-- /SYNC-BLOCK -->

## テーマCSSの取り込み

1. `${CLAUDE_SKILL_DIR}/templates/theme-<theme>.css`(`<theme>` は `light` または `dark`)をReadで読み込む。`${CLAUDE_SKILL_DIR}` が解決できない環境では、このSKILL.mdファイル自身が置かれているディレクトリ配下の `templates/` を探すこと。
2. 読み込んだ内容(`:root { ... }` ブロック)を、生成するHTMLの `<style>` タグの先頭にそのままインラインで展開する。`<link>` タグによる外部参照は禁止する。
3. コンポーネント側のCSS(見出し・表・カード・コードブロックなどのスタイル)は、色・フォント・余白などの値を必ずこの `:root` ブロックのCSS変数経由で参照する形で書く。これにより、`:root` ブロックを差し替えるだけでテーマ切替が完結する状態を保つ。

## HTML生成の品質要件

**MUST**

- 単一HTMLファイルであること(CSS・JSはすべてインライン。別ファイルへの分割禁止)
- 外部ネットワーク依存がゼロであること(CDN・Webフォント・外部画像の取得を行わない。フォントはローカルにあるフォントスタックのみを使用する)
- `<meta name="viewport" content="width=device-width, initial-scale=1">` などによるレスポンシブ対応を入れること
- 元Markdownの見出し階層(h1〜h3)を、生成HTMLでも `<h1>`〜`<h3>` として保持すること
- 元Markdownの情報(見出し・段落・リスト・表・コード・リンク)を欠落させないこと。レイアウトの都合で内容を省略・要約してはならない
- 色・フォント・余白は必ずCSS変数経由で参照し、`:root` ブロックの外に生のカラー値(hex/rgb/hsl等)を書かないこと
- テーマ変数に存在しない色が必要になった場合(正負の強調色、影、ホバー背景など)は、生成HTMLの `:root` ブロック内に追加のCSS変数(例: `--positive`, `--negative`, `--shadow`)を定義してから `var()` 経由で参照すること。`:root` ブロックの外に生のカラー値(hex / rgb / rgba / hsl 等)を書くことは、box-shadow等の一見軽微な用途を含め一切禁止。これはテーマ差し替え(`:root` ブロックの置換)だけで全配色が切り替わる状態を保つための制約である。追加変数の値は選択中のテーマ(light/dark)の雰囲気に調和させること
- `lang` 属性をコンテンツの言語に合わせて設定すること(日本語コンテンツなら `lang="ja"`)
- `dashboard` レイアウト選定時は、広い画面で複数カラムになるレスポンシブグリッド(CSS grid等)でテーブルカードを配置し、狭い画面では1カラムに折り返すこと(テーブルカードを縦一列に積むだけの構成は不可)

**SHOULD**

- セマンティックHTML(`header` / `nav` / `main` / `section` 等)を用いること
- インラインJSは数十行以内のvanilla JSに留め、タブ切替・アコーディオン開閉・目次ハイライトなど限定的な用途にのみ使うこと。フレームワークやビルドツールは使わない
- JavaScriptが無効な環境でも、全コンテンツが読める状態(プログレッシブエンハンスメント)を保つこと

## 元Markdown内の特殊コンテンツの扱い

- Markdown中の生HTMLタグは、原則そのまま出力に保持する。ただし以下は除去する(生成HTMLは配布・共有され得る単一ファイルであるため): `<script>` タグ、`onclick` 等のイベントハンドラ属性(`on*` 属性全般)、`javascript:` スキームのURL(href/src等)。
- 相対パスの画像・リンクはそのまま出力する。単体のHTMLファイルとして開いた場合にリンク切れになる可能性がある旨を、完了報告で言及する。
- 外部URLを指す画像タグはそのままタグとして保持してよいが、fetchして内容を取得したりBase64等でインライン化したりはしない。

## 出力

- 入力Markdownと同じディレクトリに `<入力ファイル名>.html` として生成する(例: `report.md` → `report.html`)。
- 生成するHTMLの冒頭、`<!DOCTYPE html>` の直後に、以下の形式で生成マーカーコメントを必ず埋め込む。

  ```html
  <!-- generated-by: md-to-html | layout: <正規化レイアウトID> | theme: <テーマ> -->
  ```

  例: `<!-- generated-by: md-to-html | layout: timeline | theme: dark -->`

- `layout` の値には、レイアウト判定ロジックのStep Eで定義した正規化レイアウトID(`dashboard` / `timeline` / `comparison` / `sidebar-nav` / `code-cards` / `article`、ハイブリッドは `article+<強調要素>`)を必ず使用する。完了報告の説明文は日本語でよいが、マーカー内のIDは必ずこの英語IDにする。
- 出力先に同名ファイルが既に存在する場合:
  - そのファイルが上記の生成マーカーコメントを持つ場合は、黙って上書きしてよい。
  - マーカーを持たない場合(手書きファイルである可能性がある)は、上書きしてよいかユーザーに確認してから処理を進める。

## 完了報告

生成が完了したら、以下を含めて簡潔に報告する。

- 選定したレイアウトと、その判定理由(Step A〜Cで確認した主要な特徴量)を1〜2行で
- 出力ファイルの絶対パス
- 相対リンク・相対画像パスが含まれていた場合は、その旨と単体で開いた際に切れる可能性
