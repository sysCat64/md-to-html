# md-to-html 開発記録(HANDOFF)

## セッション記録: 2026-07-03 (Antigravity専用プラグイン実装)

### 実施体制
- 司令塔: Antigravity
- 実装: Antigravityサブエージェント (self)

### 変更内容
- `.antigravity-plugin/plugin.json` を追加し、Antigravity用プラグインマニフェストを定義。
- `.antigravity-plugin/skills/` ディレクトリを作成し、`md-to-html` から `../../skills/md-to-html` へのシンボリックリンクを作成。
- `README.md` を更新し、Antigravityについての概要、インストール・マニフェストの使用方法、ディレクトリ構成、マルチエージェント対応の記述を追記。

### 非破壊境界
- Claude CodeおよびCodex用の既存マニフェストやスキル定義（`skills/`配下）、判定ロジックの正本（`.agent/`配下）は変更せず、既存のプラグイン構成を維持。

## セッション記録: 2026-07-03 (Codex側プラグインマニフェスト追加)

### 実施体制
- 司令塔: Codex
- 調査: Codexサブエージェント(read-only)
- 実装: Codexメインセッション

### 変更内容
- `.codex-plugin/plugin.json` を追加し、Codex用プラグインマニフェストを定義。
- 既存の `.claude-plugin/plugin.json`、`skills/md-to-html/SKILL.md`、テーマCSSは変更しない方針で実施。
- Codexマニフェストは既存 `skills/` を参照し、判定ロジックの正本は引き続き `.agent/skills/md-to-html/INSTRUCTIONS.md` とする。

### 非破壊境界
- Claude Code用マニフェストとスキル本文は維持。
- `SKILL.md` と `INSTRUCTIONS.md` のSYNC-BLOCK運用も維持。
- READMEのみ、Claude Code / Codexの両マニフェストが存在する現状に合わせて更新。

## セッション記録: 2026-07-03 (Claude Code側実装)

### 実施体制
- 司令塔: Fable 5(メインセッション)
- 設計: Opus 4.8 サブエージェント(architect役)
- 実装: Sonnet 5 サブエージェント(builder役)
- セカンドオピニオン: Codex CLI 0.142.5 / GPT-5.5 / reasoning xhigh(`codex exec -s read-only`)
- 注: `.claude/agents/architect.md` / `builder.md` は作成済みだが、作成セッション内では
  Agentツールに反映されないため、今回は general-purpose + model指定で同等構成を実現した。
  次回セッション以降は `architect` / `builder` として直接呼び出せる。

### ユーザー確定事項(2026-07-03)
1. テーマCSSは light / dark の2種
2. 生成HTMLに軽量インラインJS(タブ・アコーディオン等、外部依存なし)を許容
3. 日本語既定値(line-height 1.8、Noto Sans JP系スタック、和文letter-spacing)をCSS変数に含める

### 設計判断の要点(architect: Opus 4.8)
- レイアウト判定: 特徴量抽出 → しきい値分類 → 優先順位カスケード
  (比較 > 時系列 > コード > 表主体 > 深い階層 > シンプル記事)
- SKILL.md description は英語主体+日本語トリガー語併記(undertrigger対策)
- INSTRUCTIONS.md は参照方式ではなく内容複製+SYNC-BLOCKマーカー
  (プラグイン配布先に .agent/ が存在しないため、SKILL.md は自己完結が必須)
- CSS変数コントラクト15変数を両テーマ同名で定義、生成HTMLは変数経由のみ

### Codexセカンドオピニオンの採否(司令塔判断)
| 提案 | 採否 | 理由 |
|---|---|---|
| 判定根拠をHTMLコメントに埋め込む(`layout=...`) | 採用 | eval可能性・透明性が向上。上書き保護マーカーと兼用できる |
| 迷った場合「記事ベース+局所強調」ハイブリッド | 採用 | 単純フォールバックより情報価値を保てる |
| 既存HTML上書き時の保護 | 採用 | 生成マーカー付きなら上書き可、無印なら確認、の2段構え |
| Markdown内HTML/相対リンク/外部画像の扱いを仕様化 | 採用 | rawHTMLは`<script>`除き保持、相対リンクはそのまま、外部fetchなし |
| フォントスタックはローカルのみ・Webフォント非読込を明記 | 採用 | 「外部依存ゼロ」assertとの整合のため |
| `:root`外の生カラー値禁止をevalでassert | 採用 | テーマ差し替え可能性の機械的検証になる |
| INSTRUCTIONS.md正本+ビルドスクリプトで埋込 | 一部採用 | スクリプトは過剰。SYNC-BLOCK複製を維持し、一致チェックのCI化は将来課題として記録 |
| allowed-toolsからGlobを削除 | 不採用 | 曖昧なパス指定時のファイル特定にGlobが必要 |

### 構成変更(HANDOFF原案からの逸脱)
- `templates/`(ルート直下・任意扱い)→ `skills/md-to-html/templates/` に変更。
  理由: `${CLAUDE_SKILL_DIR}` がスキルディレクトリを指すため、配下に置くと
  配布・キャッシュコピー後もパス解決が確実(architect・Codex双方が推奨)。

### evals実行結果(サブエージェント孤立実行・構造ベースassert)
- iteration-1: 49/52パス。失敗3件 → スキル修正2点(生成マーカーのlayoutラベルを正規化英語IDに統一
  = SYNC-BLOCK v2、dashboardレイアウトのレスポンシブグリッド要件をMUST化)
- iteration-2: マーカー正規ID・グリッド配置は解消。残1件 = 生成側が `:root` 外に生カラー値を直書き
  → 「追加色が必要な場合は `:root` に変数を追加してから `var()` 参照」をMUSTに追記
- iteration-3(表主体のみ再実行): 全パス。**最終結果: 3ケースすべて全assertグリーン**
  (dashboard 17/17、timeline 18/18(darkテーマ適用含む)、article 17/17)

### Codex実装レビューの採否(司令塔判断・2回目)
| 指摘 | 採否 | 理由 |
|---|---|---|
| 生HTMLサニタイズ不足(イベント属性・javascript: URL) | 採用 | 配布され得る単一HTMLとして実害のあるXSS懸念 |
| READMEにskills/自動発見の前提を明記 | 採用 | 軽微だが利用者の混乱を防ぐ |
| ${CLAUDE_SKILL_DIR}が配布時に壊れる | 一部採用 | 公式仕様でプラグインスキルにも定義されるため誤検知。ただし未解決環境向けフォールバック1文を保険として追記 |
| Markdown変換手順が未定義 | 一部採用 | 手順固定はLLM主導設計に反するため不採用。「情報を欠落させない」MUSTのみ追加 |
| evalカバレッジ不足(比較/code-cards/sidebar-nav/上書き確認) | 一部採用 | HANDOFF要件「最低3パターン」は充足。将来課題として記録 |

### 将来課題
- SYNC-BLOCK(SKILL.md ↔ .agent/skills/md-to-html/INSTRUCTIONS.md)の一致をCIで自動チェック
- Antigravity CLI 用ラッパーの作成(別セッション予定。INSTRUCTIONS.md が正本)
- evalケースの追加: 比較(comparison)、コード中心(code-cards)、深い階層(sidebar-nav)、
  既存HTML上書き確認、生HTML/相対リンク警告の動作検証
- README.md のリポジトリURLプレースホルダをGitHub公開時に差し替え
- plugin.json の `repository` フィールドをGitHub公開時に追加
