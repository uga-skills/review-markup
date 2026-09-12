---
トークン使用量推定値: 10819
計測方法: "Anthropic Messages API count_tokens (claude-sonnet-5)"
SKILL.md行数: 391
---

# review-markup

Claude Code 用のマークアップレビュースキルです。
WHATWG HTML Living Standard、WAI-ARIA、ARIA in HTML、APG Patterns などの仕様に照らし合わせ、セマンティクスとアクセシビリティの観点で HTML をレビューします。

## Install

### すべてのプロジェクトで利用する場合

```bash
git clone git@github.com:uga-skills/review-markup.git ~/.claude/skills/review-markup
```

### 特定のプロジェクトでサブモジュールとして使う場合

```bash
git submodule add git@github.com:uga-skills/review-markup.git .claude/skills/review-markup
```

## 使い方

Claude Code のチャットで `/review-markup` を実行するか、HTML のレビューを依頼します。

### 仕様ソースのローカル clone

このスキルは WAI-ARIA・ARIA in HTML・WCAG（Understanding Docs）・APG Patterns・MDN Web Docs（英語原文）を、スキル自身のディレクトリ直下の `sources/` に `git clone --depth 1` し、`grep`/`Read` で参照します（初回実行時に自動で clone、以降は `pull` で更新）。WCAG・APG Patterns は必要なディレクトリ（`understanding/`、`content/patterns/`）のみを `sparse-checkout` の部分cloneで取得し容量を抑えています（MDN は `mdn/content` リポジトリ自体が元々英語版のみのため sparse化の対象外）。WHATWG HTML Living Standard は本家がビルド前提として公開している単一ソースファイル（`source`）を `curl` で直接取得し、同様に `grep`/`Read` で参照します（ファイルが存在する限り再取得しません）。合計サイズは概ね 236MB（内訳: WAI-ARIA 約6MB、ARIA in HTML 約1MB、WCAG 約11MB、APG Patterns 約9.2MB、MDN 約201MB、WHATWG HTML 約8MB）。`sources/` は `.gitignore` 済みで、このリポジトリのコミット対象には含まれません。

WHATWG HTML Living Standard の `source` ファイルはビルド前の中間形式（Developer Edition用の条件分岐タグ等を含む）で、規範文はプレーンに読めますが `html.spec.whatwg.org` の最終的なアンカーID（例: `#the-p-element`）はビルド時に自動生成されるため含まれません。そのため、レビュー結果に引用URLを記載する際はアンカーID確認のため該当ページを都度フェッチします。

初回clone自体が失敗した場合（ネットワーク不通など）は、`sources/` を使わず各仕様の公式ページを `WebFetch` で直接読んでレビューを続行します。

**推奨：** 追加後、以下の権限を追加してください（`sources/` で不足する場合のフォールバック用）。

```json
{
  "permissions": {
    "allow": [
      "WebFetch(domain:www.w3.org)",
      "WebFetch(domain:html.spec.whatwg.org)",
      "WebFetch(domain:bugs.webkit.org)",
      "WebFetch(domain:developer.mozilla.org)"
    ]
  }
}
```
