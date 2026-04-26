---
name: github-issues
description: "GitHub Issues の作成、更新、検索、triage、ラベル、assignee、milestone、issue type、Projects、sub-issues、dependencies、コメント、テンプレート運用を日本語環境向けに支援する Skill。gh CLI、gh api、MCP、Windows PowerShell、UTF-8、日本語本文を扱う。"
license: MIT
---

# GitHub Issues — 日本語環境向け Skill

この Skill は、GitHub Issues を使った bug report、feature request、task、triage、更新、検索、Projects 管理を日本語環境で行うために使う。日本語本文、PowerShell、GitHub CLI (`gh`)、GitHub Enterprise、Issue types、Projects V2、関連 Issue のリンクを安全に扱う。

> **基本方針:** Issue は「後から読める作業単位」にする。日本語で簡潔に、再現手順・期待結果・実際の結果・受け入れ条件・依存関係を明確に書く。作成前に repository context を確認し、secret や個人情報を本文・コメント・ログへ含めない。

---

## 1. まず確認すること

| 確認項目 | コマンド / 観点 |
|---|---|
| 対象 repository | `gh repo view --json nameWithOwner,defaultBranchRef` |
| 認証状態 | `gh auth status` |
| issue の種類 | Bug、Feature、Task、Epic など |
| 既存の類似 issue | `gh issue list` / `gh issue view` / search |
| 既存 labels | `gh label list` |
| milestones | `gh api repos/OWNER/REPO/milestones` |
| issue types | organization に設定されているか |
| Projects | 追加先 project、Status / Priority など |
| 日本語本文 | UTF-8、Markdown、`--body-file` |
| 機密情報 | token、接続文字列、個人情報、ログ全文 |

リポジトリ外から操作する場合は、必ず `--repo OWNER/REPO` を明示する。

---

## 2. 参照資料

必要に応じて次を参照する。

| Reference | 用途 |
|---|---|
| `references/templates.md` | Bug / Feature / Task / Minimal の日本語テンプレート |
| `references/search.md` | Issue 検索、GitHub search syntax、`gh api` advanced search |
| `references/projects.md` | Projects V2、Status / Priority / field 更新 |
| `references/relationships.md` | sub-issues、dependencies、blocked-by / blocking、関連リンク |

---

## 3. 操作方針

### 読み取り

通常は `gh issue` または MCP の read/search/list tools を使う。CLI では `--json` / `--jq` を優先する。

```powershell
gh issue list --state open --limit 20 --json number,title,labels,assignees
gh issue view 123 --json number,title,body,state,labels,comments
```

### 作成・更新

基本的な作成は `gh issue create` でよい。issue type を設定する場合や細かい API 操作が必要な場合は `gh api` を使う。

```powershell
gh issue create --title "不具合: ログインできない" --body-file .\issue-body.md --label bug
```

issue type を設定する場合:

```powershell
gh api repos/OWNER/REPO/issues `
  -X POST `
  -f title="ログイン画面で SSO 実行時にエラーになる" `
  -f type="Bug" `
  -f body=@issue-body.md `
  --jq '{number, html_url}'
```

> 注意: `gh issue create` が issue type を直接扱えない場合がある。type が必要なら `gh api` を使う。

---

## 4. 日本語 Issue の書き方

推奨:

- タイトルは具体的で短くする。
- Issue type を使える場合、タイトルに `[Bug]` などの重複 prefix を付けない。
- 本文は Markdown の見出しで構造化する。
- 再現手順は番号付きリストにする。
- 受け入れ条件は checkbox にする。
- ログは必要最小限に抜粋し、secret を除去する。
- 日本語だけでなく、API 名・画面名・例外型・ファイル名は正確に書く。

避ける:

- 「動かない」「修正する」だけの曖昧なタイトル。
- 個人情報や token を含むログ貼り付け。
- 複数の独立した問題を 1 issue にまとめる。
- 再現条件・期待結果がない bug report。

---

## 5. Issue type と labels

Issue types が organization に設定されている場合は、分類には type を優先する。

| 種類 | 推奨 issue type | labels の例 |
|---|---|---|
| 不具合 | `Bug` | `priority:high`, `area:auth` |
| 機能追加 | `Feature` | `area:ui`, `needs-design` |
| 作業 | `Task` | `refactor`, `test` |
| 大きな取り組み | `Epic` | `roadmap` |
| ドキュメント | `Task` または `Documentation` | `documentation` |

type と同義の label (`bug`, `enhancement`) を重複して付けるかは、リポジトリ運用に合わせる。既存運用がある場合はそちらを優先する。

issue types の確認:

```powershell
gh api graphql -f query='{
  organization(login: "ORG") {
    issueTypes(first: 20) {
      nodes { name }
    }
  }
}' --jq '.data.organization.issueTypes.nodes[].name'
```

---

## 6. Issue 作成ワークフロー

1. 対象 repository を確認する。
2. 類似 issue を検索する。
3. Bug / Feature / Task のどれかを判断する。
4. `references/templates.md` から本文を作る。
5. labels、assignees、milestone、project、issue type を必要に応じて設定する。
6. `gh issue create` または `gh api` で作成する。
7. 作成後に issue URL を報告する。

PowerShell で本文ファイルを作る例:

```powershell
@"
## 概要

ログイン画面で SSO を実行するとエラーになります。

## 再現手順

1. ログイン画面を開く
2. 「SSO でサインイン」を選択する
3. 認証完了後のリダイレクトを待つ

## 期待結果

ダッシュボードへ遷移します。

## 実際の結果

エラー画面が表示されます。
"@ | Set-Content -Path .\issue-body.md -Encoding utf8

gh issue create --title "SSO ログイン後にエラー画面が表示される" --body-file .\issue-body.md --label bug
```

---

## 7. Issue 更新

既存 issue を更新するときは、まず現在の内容を取得する。

```powershell
gh issue view 123 --json number,title,body,state,labels,assignees
```

コメント追加:

```powershell
gh issue comment 123 --body-file .\comment.md
```

ラベル追加:

```powershell
gh issue edit 123 --add-label "priority:high"
```

assignee:

```powershell
gh issue edit 123 --add-assignee "@me"
gh issue edit 123 --remove-assignee "octocat"
```

close:

```powershell
gh issue close 123 --comment "対応済みのため close します。"
```

本文全体を更新する場合は、上書きで既存情報を消さないよう注意する。

---

## 8. 検索と triage

よく使う検索:

```powershell
gh issue list --state open --label bug --json number,title,updatedAt
gh issue list --assignee "@me" --state open --json number,title,labels
gh search issues "repo:OWNER/REPO is:open no:assignee"
gh search issues "org:ORG is:open type:Bug -linked:pr"
```

古い issue:

```powershell
gh search issues "repo:OWNER/REPO is:open updated:<2026-01-01"
```

重複確認:

```powershell
gh search issues "repo:OWNER/REPO ログイン SSO in:title,body"
```

複雑な検索や Projects fields は `references/search.md` を参照する。

---

## 9. Projects V2

Projects に追加・更新する場合は、project number、field、option ID を確認する。

代表的な流れ:

1. project を特定する。
2. field 一覧を取得する。
3. issue の project item ID を取得する。
4. Status / Priority / Target date などを更新する。

書き込みには `project` scope が必要な場合がある。

```powershell
gh auth refresh -h github.com -s project
```

詳細は `references/projects.md` を参照する。

---

## 10. sub-issues / dependencies / related issues

大きな作業は親 issue と sub-issues に分ける。

本文で明示する例:

```markdown
## 関連 Issue

- Parent: #100
- Depends on: #101
- Blocks: #102
- Related: #103
```

GitHub の sub-issues / dependencies 機能が使える場合は API / UI で設定する。使えない場合でも、本文やコメントに明確なリンクを残す。

詳細は `references/relationships.md` を参照する。

---

## 11. 画像・ログ・添付

- 画像やログは必要最小限にする。
- token、cookie、メールアドレス、顧客名、接続文字列を除去する。
- UI 不具合はスクリーンショットが有効。
- 文字化けを避けるため UTF-8 の Markdown ファイルから投稿する。
- 長大ログは gist や artifact ではなく、必要箇所の抜粋を優先する。

---

## 12. セキュリティ

Issue には機密情報を書かない。

含めないもの:

- access token
- API key
- connection string
- password
- cookie / session ID
- 個人情報
- 顧客データ
- private endpoint
- 内部 IP / network 構成の詳細

セキュリティ脆弱性の可能性がある場合は、public issue ではなく repository の Security Advisories やチームの脆弱性報告フローを使う。

---

## 13. レビュー時チェックリスト

- [ ] 対象 repository が明確
- [ ] 類似 issue を確認した
- [ ] タイトルが具体的
- [ ] type / label / milestone / assignee が適切
- [ ] Bug は再現手順、期待結果、実際の結果がある
- [ ] Feature は動機、提案、受け入れ条件がある
- [ ] Task は目的、詳細、チェックリストがある
- [ ] 関連 issue、依存関係、blocker が書かれている
- [ ] 日本語本文が UTF-8 で扱われている
- [ ] secret や個人情報を含まない
- [ ] 作成後に URL を報告する

---

## 14. 公式参照

| リソース | URL |
|---|---|
| GitHub Issues | https://docs.github.com/issues |
| GitHub CLI issue manual | https://cli.github.com/manual/gh_issue |
| GitHub REST Issues API | https://docs.github.com/rest/issues/issues |
| GitHub GraphQL API | https://docs.github.com/graphql |
| GitHub Projects | https://docs.github.com/issues/planning-and-tracking-with-projects |
