---
name: accessibility
description: "Web アクセシビリティ、WCAG 2.1/2.2、JIS X 8341-3、インクルーシブ UX、アクセシビリティテスト、PR レビューを日本語環境向けに支援する Skill。ASP.NET Core / Razor / Blazor / Fluent UI Blazor の UI、HTML / ARIA / CSS、keyboard / focus、フォーム、コントラスト、screen reader、日本語 UI を扱う。"
license: MIT
---

# Accessibility — 日本語環境向け Skill

この Skill は、Web UI の設計・実装・レビュー・QA でアクセシビリティを扱うときに使う。ASP.NET Core、Razor Pages、MVC、Blazor、Fluent UI Blazor、通常の HTML / CSS / JavaScript を対象に、WCAG、JIS X 8341-3、日本語 UX、支援技術での確認を実践できる形に落とし込む。

> **必須の一次情報:** WCAG / JIS / アクセシビリティ要件を判断・説明・レビューする場合は、必ず W3C WAI の WCAG 日本語ページを一次情報として参照する: https://www.w3.org/WAI/standards-guidelines/wcag/ja
>
> **重要:** この Skill は WCAG 本文、達成基準、解説書、達成方法集の完全な複製ではない。網羅判定や適合判定を行う場合は、必ず W3C WAI の一次情報とそこからリンクされる WCAG 2 文書、解説書、達成方法集、クイックリファレンスを確認する。

---

## 0. この Skill の位置づけ

- この Skill は、アクセシビリティ作業を始めるための運用手順、レビュー観点、確認フローをまとめたもの。
- WCAG の全ガイドライン・全達成基準・全達成方法を網羅したチェックリストではない。
- 「問題がありそうな領域を見つける」ための実務ガイドであり、「WCAG 適合を証明する最終根拠」にはしない。
- WCAG 適合レベル、達成基準 ID、例外条件、適用範囲、バージョン差分を判断する場合は、W3C WAI の一次情報を確認して引用する。
- W3C WAI の日本語ページには、翻訳が英語版の最新更新を反映していない可能性がある旨の注意がある。日本語ページを必須の入口としつつ、規格の厳密な判断ではリンク先の英語原文・W3C 勧告・変更履歴も確認する。

---

## 1. まず確認すること

| 確認項目 | 見る場所 / 観点 |
|---|---|
| 対象 UI | ASP.NET Core、Razor、MVC、Blazor、Fluent UI Blazor、HTML / CSS / JS |
| 適合目標 | WCAG 2.1 / 2.2、A / AA / AAA、JIS X 8341-3、組織の基準 |
| 日本語 UI | `lang="ja"`、日本語ラベル、入力、エラー文、UTF-8 |
| 操作方法 | keyboard、pointer、touch、screen reader、音声入力 |
| コンポーネント | dialog、menu、tabs、toast、DataGrid、form、chart、media |
| 動的 UI | route change、live region、async status、validation error |
| 視覚設計 | contrast、focus indicator、400% zoom、forced-colors、reduced motion |
| テスト方法 | 手動確認、NVDA / VoiceOver / Narrator、axe、pa11y、Lighthouse |

---

## 2. 必須ルール

### WCAG は W3C WAI の日本語ページを一次情報にする

達成基準 ID、レベル、用語、適合方針、WCAG 2.1 / 2.2 の違いが曖昧な場合は、推測で断定せず、次を確認してから回答する。

```text
https://www.w3.org/WAI/standards-guidelines/wcag/ja
```

MDN、WAI-ARIA Authoring Practices、Microsoft Learn、Fluent UI Blazor demo、JIS X 8341-3 などを追加参照しても、WCAG の根拠は上記 URL を優先する。

---

### WCAG 確認フロー

1. W3C WAI の WCAG 日本語ページを開き、対象が WCAG 2.0 / 2.1 / 2.2 / WCAG 3 のどれかを確認する。
2. WCAG 2.2 は、知覚可能・操作可能・理解可能・堅牢という 4 原則のもとに 13 のガイドラインとテスト可能な達成基準があることを前提にする。
3. 適合レベルは A / AA / AAA を確認し、ユーザー要求や組織基準がどのレベルを求めているかを明確にする。
4. 具体的な達成基準、解説、達成方法、テスト方法が必要な場合は、W3C WAI の WCAG ページからリンクされる WCAG 2 文書、解説書、達成方法集、クイックリファレンスを参照する。
5. 指摘や修正方針には、可能な範囲で達成基準 ID、レベル、根拠 URL、確認方法を添える。
6. 日本語翻訳と英語原文・最新変更履歴に差分がある場合は、その差分を明記し、英語原文の W3C 勧告を厳密な根拠として扱う。

---

### Native first

まず semantic HTML と標準コントロールを使う。ARIA は semantic HTML で足りない場合だけ使い、native semantics を壊さない。

推奨:

```html
<button type="button">検索</button>
<label for="email">メールアドレス</label>
<input id="email" name="email" type="email" autocomplete="email">
```

避ける:

```html
<div role="button" tabindex="0" onclick="search()">検索</div>
```

---

### Keyboard and focus first

- `Tab` / `Shift+Tab` の順序が視覚順・意味順と一致する。
- `Enter` / `Space` / `Esc` / 矢印キーの標準的な挙動を壊さない。
- focus indicator を削除しない。デザインに合わせる場合も見える代替を用意する。
- dialog / drawer / popover は初期 focus、focus trap、focus restore を確認する。
- keyboard trap を作らない。閉じる手段を keyboard で提供する。

---

### Forms and errors

- すべての入力に visible label と programmatic label を用意する。
- visible label と accessible name は一致または近い表現にする。
- 入力例、制約、必須条件は入力前に分かる位置へ置く。
- error summary と inline error を必要に応じて併用する。
- `aria-describedby` で説明・エラーと control を関連付ける。
- validation error 後もユーザー入力を保持する。
- 氏名、ふりがな、郵便番号、住所、電話番号など日本語入力の揺れを考慮する。

例:

```html
<label for="postal-code">郵便番号</label>
<p id="postal-code-hint">ハイフンなしの7桁で入力してください。</p>
<input
  id="postal-code"
  name="postal-code"
  autocomplete="postal-code"
  inputmode="numeric"
  aria-describedby="postal-code-hint postal-code-error">
<p id="postal-code-error">郵便番号は7桁の数字で入力してください。</p>
```

---

### Dynamic UI and announcements

- SPA / Blazor の route change では page title 更新と必要な announcement を行う。
- 保存中、成功、失敗、非同期更新は `role="status"` / `aria-live` などで適切に伝える。
- toast だけで重要なエラーを伝えない。画面内にも残る message / validation を用意する。
- loading indicator は視覚だけでなく状態として伝わるようにする。

---

### Visual design and motion

- テキスト、非テキスト UI component、focus indicator の contrast を確認する。
- 色だけで必須、エラー、成功、選択中、状態を伝えない。
- 400% zoom で主要コンテンツと操作が使えることを確認する。
- `forced-colors` と high contrast で操作不能にならないようにする。
- `prefers-reduced-motion` を尊重し、必要な場合は animation / transition を抑制する。

---

## 3. ASP.NET Core / Blazor の観点

- root layout または app shell に適切な `lang="ja"` がある。
- `EditForm`、validation summary、field error、label の関連付けを確認する。
- Blazor component が生成する HTML を browser devtools / accessibility tree で確認する。
- `FluentButton` などの icon-only control は visible text または `aria-label` / `AriaLabel` で名前を持つ。
- Fluent UI Blazor の dialog、toast、menu、tooltip provider が root layout に配置されている。
- DataGrid は header、sort state、row / cell navigation、caption / summary、操作列の accessible name を確認する。
- route change、dialog open / close、toast、非同期更新の focus management と announcement を確認する。

---

## 4. レビュー時チェックリスト

このチェックリストは代表的な問題を拾うための非網羅リスト。WCAG 適合性の網羅確認では、W3C WAI の WCAG 文書、解説書、達成方法集、クイックリファレンスで対象達成基準を確認する。

- [ ] WCAG の根拠は W3C WAI の WCAG 日本語ページで確認している
- [ ] 対象バージョン、適合レベル、関連する達成基準 ID を確認している
- [ ] semantic HTML と native control を優先している
- [ ] interactive control に accessible name がある
- [ ] label、hint、error、summary が form control と関連付いている
- [ ] keyboard だけで到達・操作・脱出できる
- [ ] focus indicator が常に見える
- [ ] dialog / menu / toast / route change の focus と announcement が適切
- [ ] 色だけで状態を伝えていない
- [ ] contrast、400% zoom、forced-colors、reduced motion を考慮している
- [ ] `lang="ja"`、日本語入力、全角半角、長い日本語文、エラー文を確認している
- [ ] image、icon、chart、table、DataGrid に適切な代替情報や構造がある

---

## 5. テスト手順

### 手動確認

1. Keyboard only で主要フローを完走する。
2. `Tab` / `Shift+Tab` / `Enter` / `Space` / `Esc` / 矢印キーを確認する。
3. focus indicator が sticky header、overlay、scroll container に隠れないか確認する。
4. 400% zoom で主要コンテンツと操作が使えるか確認する。
5. Windows high contrast / forced-colors を確認する。
6. NVDA、Narrator、VoiceOver、TalkBack のいずれかで主要フローを smoke test する。
7. 日本語入力、全角半角、長い氏名・住所・エラー文を確認する。

### 自動確認

PowerShell:

```powershell
npx @axe-core/cli http://localhost:3000 --exit
npx pa11y http://localhost:3000 --reporter cli
npx lhci autorun --only-categories=accessibility
```

自動検査だけで完了としない。keyboard、focus、screen reader、zoom、contrast は手動確認を併用する。

---

## 6. PR レビュー出力例

```md
Accessibility review:
- Finding A11Y1
  - Severity: blocking | high | medium | low
  - File/line: ...
  - Issue: ...
  - WCAG/JIS reference: ...
  - Primary source: https://www.w3.org/WAI/standards-guidelines/wcag/ja
  - WCAG version / level checked: ...
  - Evidence: ...
  - Risk: ...
  - Suggested fix: ...
  - Verification: ...
```

指摘がない場合:

```md
Accessibility review:
- Findings: none
- Notes:
  - WCAG primary source checked: https://www.w3.org/WAI/standards-guidelines/wcag/ja
  - This was a focused review, not a full WCAG conformance audit.
```

---

## 7. 公式参照

| リソース | URL |
|---|---|
| W3C WAI WCAG 日本語ページ | https://www.w3.org/WAI/standards-guidelines/wcag/ja |
| WCAG 2 documents | https://www.w3.org/WAI/standards-guidelines/wcag/docs/ |
| WCAG 2.2 Recommendation | https://www.w3.org/TR/WCAG22/ |
| WAI-ARIA Authoring Practices | https://www.w3.org/WAI/ARIA/apg/ |
| MDN Accessibility | https://developer.mozilla.org/docs/Web/Accessibility |
| Microsoft Fluent UI Blazor | https://www.fluentui-blazor.net/ |
| Blazor documentation | https://learn.microsoft.com/ja-jp/aspnet/core/blazor/ |

---

## 8. Prompt starters

- 「この UI 差分を WCAG 2.2 AA、keyboard、focus、ARIA の観点でレビューしてください。」
- 「日本語住所フォームの label、hint、error、autocomplete を改善してください。」
- 「Blazor の dialog に focus trap、focus restore、screen reader announcement を設計してください。」
- 「DataGrid の sort、操作列、caption、keyboard navigation を確認してください。」
- 「400% zoom、forced-colors、NVDA を含むアクセシビリティ QA 手順を作ってください。」
