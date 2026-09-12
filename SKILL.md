---
name: review-markup
description: HTML のセマンティクスとアクセシビリティを、WHATWG HTML Living Standard・WAI-ARIA・ARIA in HTML・APG Patterns に照らしてレビューする。ユーザーが「このHTMLレビューして」「マークアップ直して」「アクセシビリティ的に問題ないか見て」など、HTML構造やa11yの妥当性を確認したい場合は必ずこのスキルを使う。
---

# Skill: review-markup

HTML のセマンティクスとアクセシビリティのスペシャリストとして、マークアップのコードレビューを行う。

## 実行手順

### ステップ 0: 仕様ソースの準備（ローカル clone を優先）

このスキルは仕様の参照を、原則としてローカルにcloneした一次情報源への `grep`/`Read` で行う。フェッチは、cloneに含まれない情報（策定中のissue、最新のerrata等）が必要な場合のみ補助的に使う。

このスキル自身のディレクトリ（`SKILL.md` と同じ場所）直下に `sources/` を作り、以下をcloneする（初回のみ。合計で概ね236MB程度になる。以降は `pull` で更新する）。

`wcag`・`aria-practices` は必要なディレクトリ（`understanding/`、`content/patterns/`）のみを `git sparse-checkout`（`--filter=blob:none` の部分clone）で取得する。これにより `wcag` は約50MB→約11MB、`aria-practices` は約14MB→約9.2MBに削減できる（実測比較済み）。`mdn-content` は `mdn/content` リポジトリ自体が元々英語版（`files/en-us/**`）のみで多言語版は別リポジトリ（`mdn/translated-content`）のため、sparse化しても削減効果がなく対象外とする。`aria`・`html-aria` は単一HTMLファイルで十分小さいため通常clone。

```bash
cd "$(dirname "<このSKILL.mdの絶対パス>")"
mkdir -p sources sources/whatwg-html
for repo in \
  "aria|https://github.com/w3c/aria.git|" \
  "html-aria|https://github.com/w3c/html-aria.git|" \
  "wcag|https://github.com/w3c/wcag.git|understanding" \
  "aria-practices|https://github.com/w3c/aria-practices.git|content/patterns" \
  "mdn-content|https://github.com/mdn/content.git|"; do
  name="${repo%%|*}"; rest="${repo#*|}"; url="${rest%%|*}"; sparse="${rest#*|}"
  if [ -d "sources/$name/.git" ]; then
    git -C "sources/$name" pull --ff-only
  elif [ -n "$sparse" ]; then
    git clone --depth 1 --filter=blob:none --sparse "$url" "sources/$name"
    git -C "sources/$name" sparse-checkout set "$sparse"
  else
    git clone --depth 1 "$url" "sources/$name"
  fi
done
# whatwg-html は git 管理下にないため pull ではなく、未取得の場合のみ curl で取得する
[ -f sources/whatwg-html/source ] || curl -sL https://raw.githubusercontent.com/whatwg/html/refs/heads/main/source -o sources/whatwg-html/source
```

`sources/` は `.gitignore` 済み（このリポジトリ固有の作業コピーであり、コミット対象ではない）。

- 各ソースの中身:
    - `sources/aria/index.html` — WAI-ARIA
    - `sources/html-aria/index.html` — ARIA in HTML
    - `sources/wcag/understanding/**` — WCAG 達成基準の解説（Understanding Docs）
    - `sources/aria-practices/content/patterns/**` — APG Patterns
    - `sources/mdn-content/files/en-us/**` — MDN Web Docs（英語原文、翻訳版ではない）
    - `sources/whatwg-html/source` — WHATWG HTML Living Standard（ビルド前の単一ソースファイル。約8MB。`curl` で直接取得するため `.git` はない）
- W3C系4リポジトリとMDNはReSpec/Eleventy等のビルドを経る前のソースだが、規範文・解説文はファイル中にそのまま記述されているため、ビルドせず `grep`/`Read` で読める。
- `whatwg-html/source` も同様に規範文はプレーンに読めるが、これは本家が「HTMLではなく独自の中間言語」と明言している前処理前ファイルであり、`w-dev`/`w-nodev`（Developer Edition用の分岐）等の条件付き属性が混在する点、および `html.spec.whatwg.org` で使われる最終的なアンカーID（例: `#the-p-element`）はビルド時に自動生成されるため本ファイル中には存在しない点に注意する。内容の検索・引用にはgrepを使い、レポートに書くURL（アンカー付き）は該当箇所を都度フェッチして確認する（ステップ3参照）。

> **Claude へ**: `sources/` が存在しない、または各サブディレクトリに `.git` がない場合はcloneから開始すること。既に存在する場合も、レビュー開始前に一度 `pull --ff-only` して最新化すること（`whatwg-html/source` はファイルが存在する限り再取得不要。存在しない場合のみ `curl` で取得する）。
>
> - **既存の `sources/` の `pull` が失敗した場合**（ネットワーク不通など）: 失敗した旨をユーザーに伝えた上で、ローカルの内容のまま続行してよい。
> - **初回clone自体が失敗した場合**（`sources/` がまだ存在せず参照元が手元にない場合）: ローカルには何も参照できるものがないため、失敗した旨をユーザーに伝えた上で、以降のステップで `grep`/`Read` の代わりに各仕様の公式ページを `WebFetch` で直接読んで続行すること（フォールバックURLはステップ3の各仕様の項に記載）。

---

### ステップ 1: プロジェクト固有ルールと WCAG 達成基準レベルの確認

まず、メモリ（`MEMORY.md` および関連ファイル）を読み込み、以下の 2 点が記録されているか確認してください。

#### 1-1. プロジェクト固有の実装ルール

- 記録されている場合 → そのルールをレビューの評価軸に加える
- 記録されていない場合 → **レビュー開始前に必ずユーザーに以下を尋ねる**:

    > このプロジェクト固有の実装ルール（コーディング規約、使用する UI フレームワークの制約、禁止要素・属性など）はありますか？
    > あれば教えてください。ない場合はそのまま進めます。

    回答があればメモリに保存する。「ない」と回答された場合もその旨をメモリに記録する。

#### 1-2. 目標とする WCAG 達成基準レベル

- 記録されている場合 → そのレベルをアクセシビリティ評価の基準とする
- 記録されていない場合 → **レビュー開始前に必ずユーザーに以下を尋ねる**:

    > WCAG の達成基準として目標とするレベル（A / AA / AAA）はありますか？
    > 分からない場合はレベル AA を基準としてレビューします（JIS X 8341-3:2016 に基づく総務省「みんなの公共サイト運用ガイドライン」が公的機関に適合レベル AA を求めており、民間サイトでも AA を目標とするのが実務上一般的であるため）。
    - 回答があればメモリに保存する
    - 「分からない」または無回答の場合は **レベル AA を基準** とし、その旨をユーザーに伝える。メモリには「未指定のためレベル AA を適用」と記録する

> **Claude へ**: WCAG の達成基準レベルはレビューの評価軸に直接影響する。例えば WCAG 2.4.9 Link Purpose (Link Only) はレベル AAA のため、目標がAA以下であれば指摘しない。各指摘に WCAG 達成基準を明記し、目標レベルを超える基準への言及は「参考情報」として区別すること。

### ステップ 2: レビュー対象の把握

> **Claude へ**: 同じ会話内で同じファイルを再レビューする場合でも、必ず Read ツールでファイルを再取得すること。会話コンテキストにキャッシュされた古い内容を参照すると、修正済みの問題を指摘し続けるミスにつながる。

ユーザーが提示した HTML を確認し、以下の観点を洗い出す:

- 使用されている要素・属性の一覧
- ARIA 属性・ロールの使用箇所
- インタラクティブ要素（フォーム、ボタン、リンクなど）
- 見出し構造・ランドマーク構造
- **コンポーネントライブラリ（React/Vue/Svelte 等）を使用している場合**: コンポーネントの props やカスタムデータが DOM 要素に直接渡されていないかを確認する（後述 4-4 で評価）

### ステップ 3: 仕様の参照（grep を優先し、必要な場合のみフェッチ）

レビュー対象に応じて、関係する仕様のみ確認する。**毎回すべてを確認する必要はない。**

**優先順位:** ステップ0の方針（`sources/` の grep/Read を優先し、フェッチは補助）に従う。

#### WHATWG HTML Living Standard

- 要素定義の確認（コンテンツモデル、許可される属性、親要素の制約など）
    - `grep` 対象: `sources/whatwg-html/source`（例: `<dfn element><code>p</code></dfn>` で `p` 要素の定義セクションを検索）
    - レポートに引用URLを書く場合のみ、該当する `https://html.spec.whatwg.org/multipage/` 配下のページをフェッチしてアンカーIDを確認する（`source` 内には最終的なアンカーIDが存在しないため）。以下はよく参照する例であり網羅ではない。リストにない要素・トピックは `https://html.spec.whatwg.org/multipage/indices.html`（要素索引）等から該当ページを特定してフェッチすること:
        - （例）`p` 要素: https://html.spec.whatwg.org/multipage/grouping-content.html#the-p-element
        - （例）`div` 要素: https://html.spec.whatwg.org/multipage/grouping-content.html#the-div-element
        - （例）`section`/`article`/`aside`/`nav`: https://html.spec.whatwg.org/multipage/sections.html
        - （例）`button`/`input`/`select`/`textarea`: https://html.spec.whatwg.org/multipage/form-elements.html
        - （例）`a` 要素: https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-a-element
        - （例）コンテンツカテゴリ全般: https://html.spec.whatwg.org/multipage/dom.html#content-categories
        - （例）インタラクティブコンテンツ: https://html.spec.whatwg.org/multipage/dom.html#interactive-content-2

#### W3C WAI-ARIA 1.2

- ARIA ロール・属性の使用が適切かどうか
    - `grep` 対象: `sources/aria/index.html`（ロール定義は `id="role_definitions"` 周辺、ステート・プロパティ定義は `id="state_prop_def"` 周辺）
    - フォールバック: https://www.w3.org/TR/wai-aria-1.2/#role_definitions 、 https://www.w3.org/TR/wai-aria-1.2/#state_prop_def

#### ARIA in HTML (W3C)

- HTML 要素に対する暗黙ロール・許可される ARIA ロールの確認
    - `grep` 対象: `sources/html-aria/index.html`
    - フォールバック: https://www.w3.org/TR/html-aria/

#### APG Patterns (ARIA Authoring Practices Guide)

- UI パターン（モーダル、タブ、メニューなど）のキーボード操作・ARIA 使用の確認
    - `grep` 対象: `sources/aria-practices/content/patterns/<pattern-name>/<pattern-name>-pattern.html`
    - フォールバック: https://www.w3.org/WAI/ARIA/apg/patterns/

#### WCAG 達成基準（Understanding Docs）

- 個別の達成基準の意図・適合方法・失敗例の確認
    - `grep` 対象: `sources/wcag/understanding/<version>/<criterion-slug>.html`（例: `sources/wcag/understanding/22/target-size-minimum.html`）
    - フォールバック: https://www.w3.org/WAI/WCAG22/Understanding/

#### MDN Web Docs（英語原文。参考情報として扱う）

- ブラウザの実装状況・具体的な使用例の補助的な確認（一次仕様ではないため根拠としては弱い）
    - `grep` 対象: `sources/mdn-content/files/en-us/**/index.md`
    - フォールバック: https://developer.mozilla.org/en-US/docs/Web （英語版のみ。翻訳版は参照しない）

### ステップ 4: 評価の実施

以下の評価軸でレビューする。各指摘は **仕様の該当箇所を明示**すること。

---

#### 4-1. セマンティクスの妥当性

**コンテンツモデルの遵守**

- 各要素のコンテンツモデル（子要素として何を含められるか）に違反していないか
- 親要素の許可コンテンツに対して当該要素が妥当か

**要素選択の妥当性**

- 意味的に適切な要素が選ばれているか
- `div` や `span` が使われている箇所で、より意味のある要素が使えないか検討する

**`p` 要素 vs `div` 要素の判断基準**

`p` 要素はフレージングコンテンツのコンテナとして定義されている（WHATWG HTML Living Standard § 4.4.1）。
コンテンツモデルは "Phrasing content" であり、使用可能な文脈は「フローコンテンツが期待される場所」である。
子要素にフレージングコンテンツのみを含む場合、**レイアウト目的であっても `p` 要素を使う妥当性がある**。

`div` を `p` への置き換え候補として指摘する条件:

1. 対象が `div` 要素で、子要素がすべてフレージングコンテンツである
2. その `div` から祖先方向に `li`, `dd`, `td`, `th`, `dt`, `summary`, `figcaption`, `caption`, `blockquote` など、**すでに意味を持つ要素**が挟まっていない（`header`, `main`, `nav`, `aside`, `section`, `article`, `body` などランドマーク的要素の直下で `div` が素通しの役割しか果たしていない）

上記1・2を両方満たす場合のみ `p` への置き換えを提案する。**すでに意味を持つ要素が祖先にある場合、あるいはそもそも `div` を挟まず意味を持つ要素に直接フレージングコンテンツが置かれている場合は、それ自体で十分であり指摘不要**（`li`, `td` 等はそれ自身が「これはリスト項目/セルである」という意味をすでに持つため、内側の `div` を `p` に変えても意味的な向上がない）。

例:

- `body > header > div > img` → `p` へ置き換え推奨（`header` 直下で `div` が素通し）
- `body > header > ul > li > text` → 指摘不要（`li` がすでに意味を持つ）
- `li > div > small` → 指摘不要（`li` がすでに意味を持つため、内側の `div` を `p` にする必要はない）

`div` を維持すべき条件:

- 子要素にフレージングコンテンツではない要素（`div`, `p`, `ul`, `table`, `figure` 等）が含まれる（`p` の content model 違反）
- 祖先方向にすでに意味を持つ要素があり、`div` はその内側でスタイリングフック等の目的で使われている

> **Claude へ**: `p` への一律な置き換えを提案しないこと。断定してよいのは上記1・2を両方満たすときに限る。

---

#### 4-2. アクセシビリティの妥当性

**暗黙ロールと明示ロールの整合性**

- HTML 要素の暗黙ロール（ARIA in HTML § 5）を確認する
- `role` 属性で明示的にロールを付与している場合、その要素に許可されたロールか確認する（ARIA in HTML § 5 の "Allowed ARIA roles" を参照）

> **Claude へ（修正案でロールを提案する場合）**: 提案するロールが **著者指定可能（non-abstract）** か WAI-ARIA 1.2 § 5.3.1 で確認すること（抽象ロールは著者が指定してはならない）。ロールのカテゴリ（document structure / widget / landmark）がユースケースに合っているかも確認する。例: `role="grid"` は widget role でキーボードナビゲーション実装が前提のため、静的な表データには document structure role の `role="table"` が適切。

**必須 ARIA 属性の有無**

- ロールが要求する必須プロパティ・ステートが揃っているか（WAI-ARIA 1.2 § 5）

**ラベルの適切性**

- インタラクティブ要素・フォームコントロールに適切なアクセシブルネームが付いているか
- `aria-label`, `aria-labelledby`, `<label>` の使い分けが適切か

> **Claude へ（aria-label 提案時の Label in Name 確認・必須）**: 修正案として `aria-label` の追加・変更を提案する場合、提案前に必ず WCAG 2.5.3 Label in Name（レベル A、目標レベル内であれば常に適用）への抵触を自己チェックすること。対象要素に可視テキスト（テキストコンテンツ）がある場合、提案する `aria-label` の値はその可視テキストを部分文字列として含んでいなければならない。可視テキストと無関係な文言（別の説明文・用途の言い換え等）で `aria-label` を上書きする修正案は、可視ラベルと accessible name を乖離させる新たな 2.5.3 違反を生むため提案してはならない。可視テキストだけでは伝わらない補足情報を加えたい場合は、`aria-label` での上書きではなく次を優先的に検討する: (a) 可視テキスト自体に情報を追加する、(b) `aria-describedby` で補足の説明用要素を関連付ける、(c) 可視テキストの前後に visually-hidden なテキストを追加して合成する（可視テキストは維持される）。
>
> **Claude へ（title属性はaccessible descriptionとして機能しうる）**: `title` 属性は accessible name の算出（Accessible Name and Description Computation 1.1 § 4.3.1）では最終フォールバックだが、要素にテキストコンテンツ等の他の名前算出源があり accessible name がそちらから決まる場合、多くの UA は `title` の値を accessible description として提供する（HTML-AAM）。したがって「`title` の文言が accessible name に採用されない」こと自体は仕様違反ではなく、description として意図通り機能している可能性がある。この状態のみを根拠に `aria-label` への昇格を提案しないこと。提案する場合は、上記の Label in Name チェックを経たうえで、可視テキストと accessible name の一致を崩さない代替手段（`aria-describedby` 等）を優先する。

> **Claude へ（`<a>` without `href` と `aria-current`）**: パンくずリストや現在ページのリンクを `<a>` without `href` で表すのは WHATWG HTML Living Standard § 4.5.1 が明示する正当な実装。`aria-current` はグローバル属性のため任意要素に付与可能で、`<a aria-current="page">` に問題はない。`<span>` への変更を指摘する必要はない。

> **Claude へ（同名ラベルの判断）**: 複数のフォームコントロールが同じ accessible name でも、AT がロール名（例: `input[type="color"]` は「カラーウェル」）で読み上げ区別できる場合は冗長な追加ラベルを指摘しない。ただし `input[type="color"]` の暗黙ロールは "No corresponding role"（ブラウザ依存）である点に留意する。

**フォーカス管理**

- フォーカス可能な要素の順序が論理的か
- `tabindex` の使用が適切か（`tabindex="0"` と `tabindex="-1"` の使い分け）
- `tabindex` に正の値が使われていないか

**ライブリージョン**

- 動的に更新されるコンテンツに `aria-live` が適切に設定されているか

**APG パターンへの準拠**

- 既知の UI パターン（タブ、モーダル、アコーディオン、コンボボックス等）については APG のパターンに照らし合わせる
    - キーボード操作の要件（矢印キー、Escape, Enter, Space の処理）
    - 必要な ARIA ロール・属性・ステートの構成

---

#### 4-3. 追加評価軸（対象コードに応じて）

- 見出し（`h1`〜`h6`）の階層構造が適切か（飛び番がないか）
- ランドマーク（`main`, `nav`, `aside` 等）の使用が適切か
- 画像の代替テキスト（`alt` 属性）が適切か
- テーブルのマークアップ（`scope`, `headers`, `caption` 等）が適切か
- リストのマークアップが適切か（`ul`, `ol`, `dl` の使い分け）
- `aria-hidden="true"` の使用が適切か（フォーカス可能な要素を隠していないか）
- 装飾目的の空要素（`<span class="color-swatch" aria-hidden="true"></span>` など）が使われている場合、CSS 擬似要素（`::before` / `::after`）で代替できないか検討する。擬似要素はアクセシビリティツリーに現れず、DOM も増えない。

---

#### 4-4. コンポーネントライブラリ固有のチェック（対象コードがコンポーネントベースのフレームワークを使用している場合のみ）

**HTML 仕様にない属性（カスタム props・カスタムディレクティブ等）の DOM 要素への受け渡し**

React, Vue, Svelte 等のコンポーネントライブラリでは、コンポーネントが受け取ったデータをそのまま DOM 要素に渡してしまうと、HTML 仕様に存在しない属性が実際の DOM に出力される。これにより以下の問題が生じる:

1. ブラウザのコンソールに警告が出力される（React: `Warning: Unknown prop 'xxx' on <div> tag.`）
2. 意図しない属性が HTML に出力され、情報漏洩やマークアップの汚染につながる
3. HTML バリデーションに失敗する

**チェックすべき代表的なパターン:**

- React: `<button {...props}>` や `<li isActive={isActive}>` のように props をそのまま／個別に DOM 要素へ渡すスプレッド構文
- Vue: `inheritAttrs`（デフォルト `true`）により `$attrs` がルート要素に自動継承される
- Svelte: `<div {...$$restProps}>` のようにカスタムデータを含んだまま DOM に渡すスプレッド構文

**問題のある属性かどうかの判断基準:**

- WHATWG HTML Living Standard に定義されていない属性名
- `is` / `has` / `should` / `can` / `show` などで始まるブール的なフラグ名
- camelCase の属性名（HTML 仕様の属性は基本的に小文字。ただし React の JSX マッピング属性は除く）
- コンポーネント内部ロジックのためのデータ（`selectedIndex`, `loadingState` 等）

**問題なしと判断すべき属性:**

- `data-*` 属性（HTML Living Standard § 3.2.6.6 で定義されたカスタムデータ属性）
- `aria-*` 属性（WAI-ARIA 仕様で定義）
- React の JSX 属性マッピング: `className` → `class`, `htmlFor` → `for`, `tabIndex` → `tabindex` 等
- 標準 DOM イベントハンドラー: `onClick`, `onChange`, `onFocus` 等（React 固有の書き方だが標準イベントにマッピングされる）

**修正の方向性（参考として示す）:**

- React: `function Button({ children, isLoading, variant, ...htmlProps }) { return <button {...htmlProps}>{children}</button>; }` のようにカスタム props を分離してから渡す
- Vue: `inheritAttrs: false` を指定し、`v-bind="filteredAttrs"` で必要な属性のみ明示的に渡す

> **Claude へ**: カスタム props の混入は静的に見ただけでは判断しきれない場合がある。スプレッド構文があれば「混入リスクがある」として指摘し、明らかに DOM 要素へ直接渡されている場合は「優先度: 高」とする。

---

#### 4-5. 既知のブラウザバグの確認

レビュー対象に以下の要素・パターンが含まれる場合、該当する既知バグをレビュー結果の「既知のブラウザバグ情報」セクションに記載すること。仕様上は正しい実装であっても、特定のブラウザで問題が生じる場合があるため、参考情報として伝える。

| 要素・パターン        | バグの概要                                                         | 影響ブラウザ    | バグ報告                                                                                   |
| --------------------- | ------------------------------------------------------------------ | --------------- | ------------------------------------------------------------------------------------------ |
| `input[type="color"]` | キーボードフォーカスが当たらない、またはフォーカスが即座に奪われる | Safari (WebKit) | [WebKit Bug #194756](https://bugs.webkit.org/show_bug.cgi?id=194756)（2019年報告・未解決） |

> **Claude へ**: 上記バグに該当する要素がレビュー対象に含まれる場合、指摘事項ではなく「既知のブラウザバグ情報（参考）」セクションとして別枠で記載すること。コードの問題ではなくブラウザ側のバグであるため、優先度評価の対象外とする。

---

### ステップ 5: レビュー結果の出力

以下のフォーマットで出力すること。

> **Claude へ（ファイル参照のリンク形式）**: 問題箇所・修正案のコードブロックの直前に、対象ファイルの行番号へのリンクを必ず記載すること。形式は以下の通り（VSCode で直接開けるよう相対パスで記述する）:
>
> - 単一行: `[FileName.tsx:42](src/path/to/FileName.tsx#L42)`
> - 範囲: `[FileName.tsx:42-51](src/path/to/FileName.tsx#L42-L51)`

````
## マークアップレビュー結果

### 総評
（全体的な評価を 2〜3 文で）

### 指摘事項

#### #1 [優先度: 高 / 中 / 低] 件名

[FileName.html:42](path/to/FileName.html#L42)

**問題のあるコード:**
```html
（問題箇所）
```

**問題の説明:**
（何が問題か、なぜ問題なのか）

**根拠:**
（仕様の該当箇所を明示。例: WHATWG HTML Living Standard § 4.4.1、WAI-ARIA 1.2 § 6.3 など）

**修正案:**

```html
（修正後のコード）
```

---

### 確認事項

（仕様違反でも目標レベル内のWCAG違反でもないが、APGパターン等の一般的な期待挙動と異なり、意図的な設計判断である可能性がある場合のみ記載。修正案は付けず、意図的かどうかを問う形にとどめる）

#### #1 件名

[FileName.html:42](path/to/FileName.html#L42)

**該当コード:**
```html
（該当箇所）
```

**確認したいこと:**
（一般的な期待挙動・APGパターン等と異なる点と、意図的な設計判断の可能性を簡潔に）

### 既知のブラウザバグ情報（参考）

（レビュー対象に該当する既知バグがある場合のみ記載。コードの問題ではなくブラウザ側の不具合のため対応は任意。）

| 要素         | 概要           | 影響ブラウザ   | 報告       |
| ------------ | -------------- | -------------- | ---------- |
| （該当要素） | （バグの説明） | （ブラウザ名） | （リンク） |

### 参照仕様

（今回のレビューで参照した仕様の URL 一覧）

### 良い点

（適切に実装されている箇所があれば挙げる）
````

優先度の定義:

- **高**: 仕様違反、またはアクセシビリティを著しく損なう問題（目標 WCAG レベル内の達成基準違反を含む）
- **中**: 仕様上は違反ではないが、より適切な実装がある
- **低**: スタイルやベストプラクティスの観点での提案

目標レベルを超える WCAG 達成基準（例: 目標 AA なのに AAA の基準）への言及は優先度によらず「参考情報」セクションにまとめ、指摘事項には含めないこと。

> **Claude へ（優先度: 中 の適用条件・必須）**: 「仕様上は違反ではないが、より適切な実装がある」と判定する前に、まず対象が (1) 参照仕様（WHATWG HTML / WAI-ARIA / ARIA in HTML）に違反しておらず、(2) 目標レベル内の WCAG 達成基準にも違反していないことを確認する。その両方が成立する場合、それでもなお「より適切な実装がある」と客観的に言えるか（根拠となる仕様の推奨事項・既知の相互運用性問題等が明示できるか）を確認すること。APG パターン等の一般的な期待挙動と単に異なる、というだけでは優先度付きの指摘事項として扱う根拠にならない——多くの場合それは意図的な設計判断であり得る。仕様違反でも WCAG 違反でもなく、客観的な優劣も示せない挙動差異は、優先度付きの「指摘事項」として修正案を提案するのではなく、「確認事項」セクションで意図的な実装かどうかを問うだけにとどめること。

---

## 注意事項

- 仕様の解釈を独自に行わず、必ず参照した仕様の記述を根拠とすること
- 曖昧な場合は「仕様上は明記されていないが…」と前置きすること
- プロジェクト固有ルールがある場合、それと仕様要件が矛盾する場合は両方を明示し、ユーザーに判断を委ねること
