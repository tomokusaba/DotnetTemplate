# GitHub Issue Search — 日本語環境向け

Issue 検索では、単純な一覧は `gh issue list`、横断検索は `gh search issues`、複雑な field / Projects 検索は `gh api` を使う。

## 使い分け

| 目的 | 推奨 |
|---|---|
| 単一 repo の open issue 一覧 | `gh issue list` |
| テキスト検索、org 横断検索 | `gh search issues` |
| issue type、boolean、複雑な条件 | `gh search issues` または `gh api` |
| Projects field / custom field | `gh api` / GraphQL |

## 基本

```powershell
gh issue list --state open --limit 30
gh issue list --label bug --state open
gh issue list --assignee "@me" --state open
```

JSON:

```powershell
gh issue list --state open --json number,title,labels,assignees,updatedAt
```

## Search syntax

```text
repo:OWNER/REPO
org:ORG
is:open
is:closed
type:Bug
label:bug
-label:wontfix
no:label
no:assignee
assignee:@me
author:USERNAME
mentions:USERNAME
updated:<2026-01-01
created:2026-01-01..2026-01-31
linked:pr
-linked:pr
comments:>10
```

例:

```powershell
gh search issues "repo:OWNER/REPO is:open no:assignee"
gh search issues "org:ORG is:open type:Bug -linked:pr"
gh search issues "repo:OWNER/REPO is:open updated:<2026-01-01"
gh search issues "repo:OWNER/REPO label:bug comments:>10"
```

## 重複確認

新規作成前に、タイトルや重要語で検索する。

```powershell
gh search issues "repo:OWNER/REPO ログイン SSO in:title,body"
gh search issues "repo:OWNER/REPO exception timeout in:title,body"
```

## Advanced search with gh api

Issue field や複雑な条件では API を使う。

```powershell
gh api "search/issues?q=org:ORG+type:Bug+is:open&advanced_search=true" `
  --jq '.items[] | "#\(.number): \(.title)"'
```

GraphQL:

```powershell
gh api graphql -f query='{
  search(query: "org:ORG type:Bug is:open", type: ISSUE, first: 20) {
    issueCount
    nodes {
      ... on Issue {
        number
        title
        url
      }
    }
  }
}'
```

## 注意

- GitHub Search API には件数・rate limit がある。
- 検索結果は最大 1,000 件までになることがある。
- 日本語検索は表記ゆれに注意する。
- `field.name:value` は環境や API により不安定な場合がある。
