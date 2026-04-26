# GitHub Projects V2 — 日本語環境向け

Projects V2 は GraphQL ベースで管理される。GitHub CLI では `gh api graphql` を使うことが多い。

## まず確認すること

- project owner: user / organization
- project number
- project ID
- field ID
- option ID
- issue node ID
- project item ID
- token scope: `read:project` または `project`

書き込み権限が足りない場合:

```powershell
gh auth refresh -h github.com -s project
```

## Project を番号で取得

```powershell
gh api graphql -f query='{
  organization(login: "ORG") {
    projectV2(number: 42) {
      id
      title
      number
    }
  }
}' --jq '.data.organization.projectV2'
```

## Field 一覧

```powershell
gh api graphql -f query='{
  node(id: "PROJECT_ID") {
    ... on ProjectV2 {
      fields(first: 30) {
        nodes {
          ... on ProjectV2Field { id name }
          ... on ProjectV2SingleSelectField { id name options { id name } }
          ... on ProjectV2IterationField { id name configuration { iterations { id title startDate duration } } }
        }
      }
    }
  }
}'
```

## Issue の project item ID を取得

```powershell
gh api graphql -f query='{
  repository(owner: "OWNER", name: "REPO") {
    issue(number: 123) {
      id
      projectItems(first: 10) {
        nodes {
          id
          project { id title number }
        }
      }
    }
  }
}' --jq '.data.repository.issue'
```

## Project に issue を追加

```powershell
gh api graphql -f query='mutation {
  addProjectV2ItemById(input: {
    projectId: "PROJECT_ID"
    contentId: "ISSUE_NODE_ID"
  }) {
    item { id }
  }
}'
```

## Status を更新

```powershell
gh api graphql -f query='mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "PROJECT_ID"
    itemId: "ITEM_ID"
    fieldId: "STATUS_FIELD_ID"
    value: { singleSelectOptionId: "IN_PROGRESS_OPTION_ID" }
  }) {
    projectV2Item { id }
  }
}'
```

## Date field を更新

```powershell
gh api graphql -f query='mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "PROJECT_ID"
    itemId: "ITEM_ID"
    fieldId: "TARGET_DATE_FIELD_ID"
    value: { date: "2026-04-26" }
  }) {
    projectV2Item { id }
  }
}'
```

## 注意

- project 名検索は曖昧な結果になりやすい。番号が分かるなら番号で直接引く。
- field 更新には field ID と option ID が必要。
- `read:project` では書き込みできない。`project` scope が必要。
- 変更前に現在値を取得して、意図しない上書きを避ける。
