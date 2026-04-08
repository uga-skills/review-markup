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

**推奨：** 追加後、以下の権限を追加してください。

```json
{
  "permissions": {
    "allow": [
      "WebFetch(domain:www.w3.org)",
      "WebFetch(domain:html.spec.whatwg.org)",
      "WebFetch(domain:wicg.io)",
      "WebFetch(domain:bugs.webkit.org)",
      "WebFetch(domain:w3c.github.io)"
    ]
  }
}
```
