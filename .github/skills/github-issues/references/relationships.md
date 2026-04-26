# GitHub Issue Relationships — 日本語環境向け

Issue 間の関係は、作業順序と進捗把握に重要。GitHub の sub-issues / dependencies 機能が使える場合はそれを使い、使えない場合でも本文・コメントで明示する。

## 関係の種類

| 関係 | 意味 |
|---|---|
| Parent | 親 issue / epic |
| Sub-issue | 親にぶら下がる作業 issue |
| Depends on / blocked by | この issue が別 issue の完了待ち |
| Blocks / blocking | この issue が別 issue をブロックしている |
| Related | 関連するが依存関係ではない |
| Duplicate | 重複 |

## 本文での表現

```markdown
## 関連 Issue

- Parent: #100
- Depends on: #101
- Blocks: #102
- Related: #103
```

## コメントでの更新

```markdown
依存関係を更新しました。

- `#101` が完了するまで本 Issue は着手できません。
- 本 Issue が完了すると `#102` が着手可能になります。
```

## sub-issues の分け方

親 issue:

- 目的
- 範囲
- 完了条件
- sub-issues 一覧

sub-issue:

- 1 つの明確な作業単位
- 担当者を割り当てられる粒度
- 完了条件が判断できる粒度

## 例

```markdown
## 概要

顧客管理機能を追加します。

## Sub-issues

- [ ] #201 顧客一覧 API を追加
- [ ] #202 顧客一覧 UI を追加
- [ ] #203 顧客検索条件を追加
- [ ] #204 xUnit テストを追加
```

## 注意

- 依存関係は循環させない。
- 1 issue に複数の独立作業を詰め込みすぎない。
- blocked 状態は理由と解除条件を書く。
- GitHub の専用機能がない場合でも、本文のリンクだけで最低限追跡できるようにする。
