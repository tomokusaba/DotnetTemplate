---
name: gh-cli
description: "GitHub CLI (gh) を日本語環境で安全かつ効率的に使うための Skill。認証、repository、issue、pull request、review、Actions、releases、gh api、JSON/JQ、GitHub Enterprise、Windows PowerShell、文字コード、トークン管理を扱う。"
license: MIT
---

# GitHub CLI (`gh`) — 日本語環境向け Skill

この Skill は、GitHub CLI (`gh`) を使って GitHub の repository、issue、pull request、review、Actions、release、API 操作を行うときに使う。日本語環境、Windows PowerShell、GitHub Enterprise、CI、自動化スクリプトで安全に扱うことを重視する。

> **基本方針:** 破壊的操作は確認を挟み、secret や token を表示・保存しない。自動化では `--json` と `--jq` を優先し、日本語本文は UTF-8 と quoting に注意する。Windows では PowerShell 前提のコマンドを提示する。

---

## 1. まず確認すること

| 確認項目 | コマンド / 観点 |
|---|---|
| gh version | `gh --version` |
| 認証状態 | `gh auth status` |
| 対象 host | `github.com` または GitHub Enterprise hostname |
| 対象 repository | `gh repo view --json nameWithOwner,defaultBranchRef` |
| 現在 branch | `git branch --show-current` |
| remote | `git remote -v` |
| 出力形式 | human-readable か `--json` / `--jq` か |
| 日本語本文 | UTF-8、PowerShell quoting、改行の扱い |
| 操作の危険度 | delete、archive、merge、close、release publish など |

リポジトリ外から操作する場合は、`--repo owner/repo` を明示する。

---

## 2. インストールと基本設定

Windows PowerShell:

```powershell
winget install --id GitHub.cli
gh --version
```

認証:

```powershell
gh auth login
gh auth status
gh auth setup-git
```

GitHub Enterprise:

```powershell
gh auth login --hostname ghe.example.co.jp
gh auth status --hostname ghe.example.co.jp
gh auth setup-git --hostname ghe.example.co.jp
```

token を使う場合、token をコマンドライン引数に直接書かない。

```powershell
Get-Content .\token.txt | gh auth login --with-token
```

`token.txt` はコミットしない。使用後は安全に削除する。

---

## 3. 日本語環境と PowerShell の注意

- 日本語タイトル・本文は UTF-8 を前提にする。
- 長い本文は `--body-file` を使う。
- PowerShell では `'` と `"` の展開ルールに注意する。
- JSON は `ConvertFrom-Json` または `--jq` で処理する。
- secret や token を `Write-Host`、履歴、ログに出さない。
- Windows path は `.\file.md` のように書く。
- 複数行本文は here-string を使うと扱いやすい。

本文ファイルを使う例:

```powershell
@"
## 概要

日本語の本文を含む Issue です。

## 詳細

- 再現手順
- 期待結果
- 実際の結果
"@ | Set-Content -Path .\issue-body.md -Encoding utf8

gh issue create --title "不具合: 保存時にエラーが発生する" --body-file .\issue-body.md
```

---

## 4. よく使う共通オプション

| オプション | 用途 |
|---|---|
| `--repo owner/repo` | 対象 repository を明示 |
| `--json fields` | JSON 出力 |
| `--jq expression` | JSON を jq expression で抽出 |
| `--limit n` | 取得件数 |
| `--web` | ブラウザで開く |
| `--hostname host` | GitHub Enterprise host |
| `--template` | Go template で出力 |

自動化では、画面表示用出力を parse しない。必ず `--json` を使う。

```powershell
gh pr view --json number,title,state,headRefName,baseRefName
gh issue list --state open --json number,title,labels --jq '.[] | {number, title}'
```

---

## 5. Repository 操作

確認:

```powershell
gh repo view
gh repo view --json nameWithOwner,description,visibility,defaultBranchRef
gh repo list tomokusaba --limit 50
```

clone:

```powershell
gh repo clone owner/repo
gh repo clone owner/repo .\local-folder
```

作成:

```powershell
gh repo create my-repo --private --description "説明文"
gh repo create org-name/my-repo --private
gh repo create my-repo --source=. --private
```

既定 repository:

```powershell
gh repo set-default owner/repo
gh repo set-default --unset
```

危険操作:

```powershell
gh repo archive owner/repo
gh repo delete owner/repo
```

delete / archive / visibility 変更は必ずユーザー確認を挟む。

---

## 6. Issue 操作

一覧:

```powershell
gh issue list --state open --limit 20
gh issue list --label bug --state open --json number,title,labels,assignees
```

表示:

```powershell
gh issue view 123
gh issue view 123 --json number,title,body,state,labels,comments
gh issue view 123 --web
```

作成:

```powershell
gh issue create --title "不具合: ログインできない" --body-file .\issue-body.md --label bug
```

コメント:

```powershell
gh issue comment 123 --body "確認しました。再現手順を追記します。"
gh issue comment 123 --body-file .\comment.md
```

編集・close:

```powershell
gh issue edit 123 --add-label "priority:high"
gh issue close 123 --comment "対応済みのため close します。"
gh issue reopen 123
```

Issue 本文やコメントに日本語・Markdown を含む場合は `--body-file` を優先する。

---

## 7. Pull Request 操作

状態確認:

```powershell
gh pr status
gh pr list --state open --limit 20
gh pr view
gh pr view 123 --json number,title,state,author,headRefName,baseRefName,mergeable,reviewDecision
```

作成:

```powershell
gh pr create --base main --head feature/my-change --title "機能追加: 顧客一覧を追加" --body-file .\pr-body.md
```

ブラウザで確認:

```powershell
gh pr view 123 --web
```

checkout:

```powershell
gh pr checkout 123
```

diff:

```powershell
gh pr diff 123
gh pr diff 123 --name-only
```

checks:

```powershell
gh pr checks 123
gh pr checks 123 --watch
```

review:

```powershell
gh pr review 123 --comment --body "確認しました。いくつかコメントしました。"
gh pr review 123 --approve --body "LGTM です。"
gh pr review 123 --request-changes --body-file .\review.md
```

merge:

```powershell
gh pr merge 123 --squash --delete-branch
gh pr merge 123 --merge --delete-branch
gh pr merge 123 --rebase --delete-branch
```

merge は CI、review、branch protection、ユーザー意図を確認してから行う。

---

## 8. GitHub Actions

workflow 一覧:

```powershell
gh workflow list
gh workflow view "build.yml"
```

workflow 実行:

```powershell
gh workflow run "build.yml"
gh workflow run "deploy.yml" --ref main -f environment=staging
```

run 一覧:

```powershell
gh run list --limit 20
gh run list --workflow "build.yml" --branch main
gh run view 123456789
gh run view 123456789 --log
```

監視:

```powershell
gh run watch 123456789
```

再実行:

```powershell
gh run rerun 123456789
gh run rerun 123456789 --failed
```

ログ取得:

```powershell
gh run download 123456789
```

secret を含むログは共有しない。失敗調査では必要な範囲だけ抜粋する。

---

## 9. Releases

一覧・表示:

```powershell
gh release list
gh release view v1.0.0
```

作成:

```powershell
gh release create v1.0.0 .\artifacts\app.zip --title "v1.0.0" --notes-file .\release-notes.md
```

draft / prerelease:

```powershell
gh release create v1.1.0-beta --draft --prerelease --notes-file .\release-notes.md
```

upload:

```powershell
gh release upload v1.0.0 .\artifacts\installer.exe
```

release は tag、artifact、notes、対象 branch を確認してから publish する。

---

## 10. `gh api`

`gh api` は REST / GraphQL API を呼び出すための強力な escape hatch。通常の `gh pr` / `gh issue` / `gh repo` で足りる場合は専用 command を優先する。

REST:

```powershell
gh api repos/OWNER/REPO
gh api repos/OWNER/REPO/issues --paginate
```

JSON field:

```powershell
gh api repos/OWNER/REPO/issues --jq '.[].title'
```

POST:

```powershell
gh api repos/OWNER/REPO/issues `
  --method POST `
  --field title="不具合: 画面が表示されない" `
  --field body="再現手順を記載します。"
```

GraphQL:

```powershell
gh api graphql `
  -f query='query { viewer { login } }'
```

注意:

- destructive API は必ず確認する。
- token scope を最小限にする。
- response に secret や個人情報が含まれないか確認する。
- `--paginate` は大量取得になるため rate limit に注意する。

---

## 11. JSON / JQ / PowerShell

`--json` で取得し、必要な field だけ扱う。

```powershell
gh pr list --state open --json number,title,author `
  --jq '.[] | "\(.number): \(.title) (@\(.author.login))"'
```

PowerShell object として扱う:

```powershell
$prs = gh pr list --state open --json number,title,author | ConvertFrom-Json
$prs | ForEach-Object {
    "{0}: {1}" -f $_.number, $_.title
}
```

日本語をファイルに保存:

```powershell
gh issue view 123 --json title,body | Out-File -FilePath .\issue.json -Encoding utf8
```

---

## 12. GitHub Enterprise

Enterprise host を使う場合:

```powershell
gh auth login --hostname ghe.example.co.jp
gh repo view owner/repo --hostname ghe.example.co.jp
```

多くの command では `--repo HOST/OWNER/REPO` ではなく、host は認証・環境・remote から解決される。曖昧な場合は対象 repository の remote と `gh auth status` を確認する。

環境変数:

```powershell
$env:GH_HOST = "ghe.example.co.jp"
```

CI では `GH_TOKEN` または `GITHUB_TOKEN` を使う。

```powershell
$env:GH_TOKEN = $env:GITHUB_TOKEN
gh auth status
```

token はログに出さない。

---

## 13. セキュリティと安全操作

必ず守ること:

- `gh auth token` の出力を共有しない。
- token を command line 引数、README、issue、PR、ログに書かない。
- `delete`、`archive`、`merge`、`close`、`release create`、`workflow run deploy` は確認してから実行する。
- `--yes` は自動化以外で安易に使わない。
- public repository への投稿内容に機密情報がないか確認する。
- `gh api --paginate` で大量データを取得する場合は rate limit と情報範囲を確認する。
- CI の `GITHUB_TOKEN` は必要最小権限にする。

危険操作の例:

```powershell
gh repo delete owner/repo --yes
gh pr merge 123 --admin
gh release delete v1.0.0 --yes
```

これらはユーザーの明示指示なしに実行しない。

---

## 14. トラブルシュート

| 症状 | 確認すること |
|---|---|
| `gh` が見つからない | `gh --version`、PATH、winget install |
| 認証エラー | `gh auth status`、host、token scope |
| repo が解決されない | `git remote -v`、`gh repo set-default`、`--repo owner/repo` |
| GraphQL / API で権限不足 | token scope、organization SSO、Enterprise policy |
| 日本語が文字化けする | PowerShell 7、UTF-8、`--body-file`、`Out-File -Encoding utf8` |
| PR 作成で head/base が違う | current branch、`--base`、`--head` |
| Actions が見えない | repository permissions、Actions enabled、workflow file path |
| `--jq` が期待通りでない | `--json` field、配列/オブジェクト構造を確認 |

---

## 15. レビュー時チェックリスト

- [ ] 対象 repository / host が明確
- [ ] 認証状態と token scope が適切
- [ ] secret や token を出力していない
- [ ] 日本語本文は `--body-file` などで安全に扱っている
- [ ] 自動化は `--json` / `--jq` を使っている
- [ ] destructive operation は確認を挟んでいる
- [ ] CI では `GITHUB_TOKEN` / `GH_TOKEN` の権限が最小限
- [ ] `--repo owner/repo` で曖昧さを避けている
- [ ] PowerShell の quoting と改行を考慮している

---

## 16. 公式参照

| リソース | URL |
|---|---|
| GitHub CLI manual | https://cli.github.com/manual/ |
| GitHub CLI repository | https://github.com/cli/cli |
| GitHub REST API | https://docs.github.com/rest |
| GitHub GraphQL API | https://docs.github.com/graphql |
| GitHub Actions | https://docs.github.com/actions |
| GitHub Enterprise Server | https://docs.github.com/enterprise-server |
