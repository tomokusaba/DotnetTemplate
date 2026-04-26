# Microsoft Agent Framework Workflows and Safety — 日本語環境向け

Agent Framework の workflow、tool calling、安全設計、運用観測性に関する参照。

## Workflow を使う場面

| 場面 | 理由 |
|---|---|
| 業務手順が明確 | 制御フローを明示できる |
| 複数 agent を順番に動かす | sequential orchestration |
| 複数処理を並列実行する | concurrent orchestration |
| 条件で処理先を切り替える | routing / switch |
| 人間の承認が必要 | human-in-the-loop |
| 長時間実行・再開が必要 | checkpoint / session persistence |
| 複雑な処理を単一 agent として公開したい | workflow as agent |

## Agent orchestration patterns

- Sequential: writer -> reviewer のように順番に処理する。
- Concurrent: 複数 agent が同時に分析する。
- Handoff: 状況に応じて担当 agent を移す。
- Group chat: 複数 agent が対話しながら進める。
- Magentic: lead agent が他 agent を指揮する。

パターンは問題に合わせて選ぶ。単純な処理に multi-agent を使いすぎない。

## Human approval

承認を挟むべき操作:

- DB 更新。
- GitHub issue / PR / repository の変更。
- Azure resource の作成・削除。
- メールや通知の送信。
- ファイル削除。
- 外部 API への確定操作。

承認に含める情報:

- tool name。
- 引数。
- 影響範囲。
- 取り消し可否。
- 実行者。
- 監査 ID。

## Tool safety checklist

- [ ] tool の責務が小さい
- [ ] 引数が型付けされている
- [ ] 入力検証がある
- [ ] authorization がある
- [ ] destructive operation は approval required
- [ ] timeout / cancellation がある
- [ ] retry してよい操作か区別している
- [ ] secret を返さない
- [ ] audit log がある

## Prompt injection 対策

外部入力は信頼しない。

注意する入力:

- ユーザー入力。
- Web ページ。
- ドキュメント。
- GitHub issue / PR コメント。
- メール。
- RAG 検索結果。
- tool output。

対策:

- system / developer instructions と user / external content を分ける。
- tool 実行前に authorization と policy check を行う。
- 外部文書中の「前の指示を無視せよ」などを命令として扱わない。
- sensitive tool は human approval を要求する。
- structured output を validation する。

## Observability

記録する候補:

- agent name。
- workflow name。
- model provider / deployment。
- latency。
- token usage。
- tool calls。
- approval status。
- workflow status。
- error code / exception type。

既定で記録しないもの:

- prompt 全文。
- model response 全文。
- secret。
- personal data。
- customer data。

必要な場合はマスキング、サンプリング、保存期間、閲覧権限を設計する。

## Failure handling

- model timeout。
- rate limit。
- tool validation error。
- authorization failure。
- structured output validation failure。
- workflow checkpoint restore failure。
- provider outage。

失敗時は success-shaped fallback にしない。ユーザーや呼び出し元に失敗を明示し、再試行可能かを区別する。

## 日本語運用

- ユーザー向け回答は日本語。
- tool name / JSON field / telemetry key は英語。
- prompt は日本語でもよいが、出力 schema は英語 field を推奨。
- 日本語ログ本文だけに依存せず、structured field で検索できるようにする。
