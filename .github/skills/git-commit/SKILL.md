---
name: git-commit
description: "Git commit を日本語環境で安全に作成するための Skill。Conventional Commits、差分分析、論理単位の staging、コミットメッセージ生成、PowerShell、UTF-8、機密情報チェック、フック失敗時の対応、Co-authored-by などの trailer を扱う。"
license: MIT
---

# Git Commit — 日本語環境向け Skill

この Skill は、Git の変更差分を分析し、論理的でレビューしやすい commit を作成するために使う。Conventional Commits を基本にしつつ、日本語チームで読みやすく、CI・リリースノート・履歴追跡に使いやすいメッセージを作る。

> **基本方針:** 1 commit は 1 つの論理変更にする。差分を確認してから必要なファイルだけ stage し、機密情報を含めない。破壊的操作や history rewrite は明示指示なしに行わない。

---

## 1. まず確認すること

| 確認項目 | コマンド / 観点 |
|---|---|
| 作業ツリー状態 | `git --no-pager status --short` |
| staged diff | `git --no-pager diff --staged` |
| unstaged diff | `git --no-pager diff` |
| 変更ファイル一覧 | `git --no-pager diff --name-status` |
| 現在 branch | `git branch --show-current` |
| 直近履歴 | `git --no-pager log --oneline -n 10` |
| 既存の commit 形式 | Conventional Commits か、独自形式か |
| hook / CI | pre-commit、lint-staged、commit-msg hook など |
| 機密情報 | `.env`、token、key、接続文字列、証明書 |

作業ツリーに他者やユーザーの未関連変更がある場合、それを勝手に stage / revert しない。

---

## 2. Conventional Commit 形式

基本形式:

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

例:

```text
feat(auth): add SSO callback validation
fix(api): handle empty customer id
docs(readme): add Japanese setup steps
test(order): add expiry boundary tests
```

breaking change:

```text
feat(api)!: remove deprecated customer endpoint

BREAKING CHANGE: `/api/customers/legacy` is no longer supported.
```

---

## 3. Commit type の選び方

| Type | 用途 |
|---|---|
| `feat` | 新機能 |
| `fix` | 不具合修正 |
| `docs` | ドキュメントのみ |
| `style` | formatting、空白、コード意味に影響しない変更 |
| `refactor` | 機能追加でも修正でもない内部構造改善 |
| `perf` | パフォーマンス改善 |
| `test` | テスト追加・更新 |
| `build` | build system、dependencies、package 設定 |
| `ci` | CI / workflow 設定 |
| `chore` | メンテナンス、雑多な作業 |
| `revert` | 既存 commit の revert |

迷った場合:

- ユーザーに見える機能追加なら `feat`
- バグを直すなら `fix`
- テストだけなら `test`
- README や Skill など文書だけなら `docs`
- 振る舞いを変えないコード整理なら `refactor`

---

## 4. 日本語環境でのメッセージ方針

推奨は、type / scope / description を英語で書き、必要な補足を body に日本語で書く形式。

```text
docs(skills): add Japanese GitHub Issues guidance

日本語本文の扱い、Issue type、Projects V2、依存関係の記載方針を追加。
```

理由:

- Conventional Commits と release tooling に合わせやすい。
- 英語 scope は検索しやすい。
- 日本語 body はチーム内の意図共有に向く。

ただし、プロジェクトが日本語 commit subject を採用している場合は既存規約を優先する。

```text
docs(skills): GitHub Issues の日本語運用ガイドを追加
```

---

## 5. 差分分析ワークフロー

1. status を確認する。
2. staged diff があれば staged diff を優先する。
3. staged が空なら unstaged diff を確認する。
4. 変更が複数の論理単位に分かれる場合、commit を分割する。
5. 機密情報や生成物が混入していないか確認する。
6. commit type、scope、description を決める。
7. commit を作成する。

PowerShell:

```powershell
git --no-pager status --short
git --no-pager diff --staged
git --no-pager diff
git --no-pager diff --name-status
```

---

## 6. Staging 方針

論理単位ごとに stage する。

```powershell
git add .\skills\github-issues\SKILL.md
git add .\skills\github-issues\references\templates.md
```

まとめて stage してよい例:

- 1 つの feature に必要な実装・テスト・docs
- 1 つの Skill とその references
- 1 つの bug fix と regression test

分けるべき例:

- bug fix と unrelated refactor
- dependency update と feature implementation
- generated file と手書き修正
- 複数機能の追加

interactive staging:

```powershell
git add -p
```

ただし、interactive 操作が不安定な環境では、ファイル単位で明示的に stage する。

---

## 7. 機密情報チェック

commit 前に必ず確認する。

含めないもの:

- `.env`
- API key
- access token
- connection string
- private key
- certificate
- password
- cookie / session ID
- 個人情報や顧客データ
- 内部 endpoint や認証情報付き URL

確認例:

```powershell
git --no-pager diff --staged
```

怪しいファイル:

```text
.env
*.pfx
*.pem
*.key
credentials.json
appsettings.Production.json
```

機密情報を見つけた場合は commit せず、ユーザーに知らせて除去する。

---

## 8. Commit 実行

single-line:

```powershell
git commit -m "docs(skills): add Japanese Git commit guidance"
```

body 付き:

```powershell
git commit -m "docs(skills): add Japanese Git commit guidance" `
  -m "Conventional Commits、staging、機密情報チェック、PowerShell での運用方針を追加。"
```

footer / trailer 付き:

```powershell
git commit -m "docs(skills): add Japanese Git commit guidance" `
  -m "Conventional Commits、staging、機密情報チェック、PowerShell での運用方針を追加。" `
  -m "Refs #123"
```

プロジェクトや作業指示で trailer が指定されている場合は、commit message の末尾に追加する。

```text
Co-authored-by: Name <name@example.com>
Signed-off-by: Name <name@example.com>
Refs #123
```

---

## 9. Description の書き方

英語 subject の場合:

- imperative mood を使う。
- 先頭は小文字の動詞を基本にする。
- 72 文字以内を目安にする。
- 末尾に句点を付けない。

良い例:

```text
feat(auth): add SSO callback validation
fix(order): handle expired subscription boundary
docs(timezone): add Japanese TimeProvider patterns
```

避ける例:

```text
fixed bug
update files
WIP
changes
ログイン修正
```

日本語 subject の場合:

- 具体的に「何をしたか」を書く。
- 「修正」「対応」だけで終わらせない。
- scope と type は残す。

```text
fix(auth): SSO コールバックの空 state を検証する
docs(skills): GitHub CLI の日本語運用ガイドを追加
```

---

## 10. Scope の決め方

scope は変更範囲を短く表す。

例:

| 変更範囲 | scope |
|---|---|
| 認証 | `auth` |
| API | `api` |
| UI | `ui` |
| テスト | `test` |
| CI | `ci` |
| docs | `docs` |
| Skills | `skills` |
| Timezone | `timezone` |
| Fluent UI Blazor | `fluentui` |
| GitHub Issues | `issues` |

scope が不自然なら省略してよい。

```text
docs: add PR template
```

---

## 11. Issue 参照

footer に関連 Issue を書く。

```text
Refs #123
Closes #123
Fixes #123
Related to #123
```

自動 close したい場合だけ `Closes` / `Fixes` を使う。単なる関連なら `Refs` にする。

---

## 12. Hook 失敗時

commit hook が失敗したら、原則として `--no-verify` で回避しない。

対応:

1. hook の出力を読む。
2. lint / format / test の失敗を修正する。
3. 必要なファイルを再 stage する。
4. commit を再実行する。

```powershell
git --no-pager status --short
git add .\path\to\fixed-file.cs
git commit -m "fix(scope): correct validation"
```

`--no-verify` はユーザーが明示的に求めた場合だけ使う。

---

## 13. 安全プロトコル

やらないこと:

- 明示指示なしに `git reset --hard` しない。
- 明示指示なしに `git push --force` しない。
- main / master に force push しない。
- ユーザーの未関連変更を stage しない。
- 機密情報を commit しない。
- hook を勝手に skip しない。
- commit amend を勝手に行わない。
- git config を勝手に変更しない。

必要ならユーザーに確認すること:

- 複数の論理変更があり commit 分割が必要。
- staged に未関連変更が含まれている。
- destructive operation が必要。
- commit message の規約が不明。
- hook 失敗を回避したいという要望がある。

---

## 14. よくあるパターン

### docs だけ

```text
docs(skills): add Japanese GitHub Issues guidance
```

### bug fix + test

```text
fix(auth): reject expired login token

Add xUnit coverage for expired and malformed token cases.
```

### feature

```text
feat(search): add saved query filter
```

### refactor

```text
refactor(order): extract price calculation service
```

### build / dependency

```text
build(deps): update Fluent UI Blazor package
```

### breaking change

```text
feat(api)!: require timezone id for recurring schedules

BREAKING CHANGE: Recurring schedule creation now requires an IANA timezone ID.
```

---

## 15. レビュー時チェックリスト

- [ ] 変更は 1 つの論理単位にまとまっている
- [ ] 未関連変更を含めていない
- [ ] staged diff を確認した
- [ ] 機密情報を含んでいない
- [ ] type が適切
- [ ] scope が適切
- [ ] description が具体的
- [ ] Issue 参照が適切
- [ ] 必要な trailer が末尾にある
- [ ] hook を勝手に skip していない
- [ ] 破壊的 git 操作をしていない

---

## 16. 公式参照

| リソース | URL |
|---|---|
| Conventional Commits | https://www.conventionalcommits.org/ |
| Git commit documentation | https://git-scm.com/docs/git-commit |
| GitHub commit guidelines | https://docs.github.com/pull-requests/committing-changes-to-your-project |
| GitHub secret scanning | https://docs.github.com/code-security/secret-scanning |
