---
name: dotnet-design-pattern-review
description: ".NET / C# コードの設計パターン、SOLID、DI、Repository、Provider、Factory、Command Handler、ResourceManager、TimeProvider、Microsoft Agent Framework、xUnit テスト容易性を日本語環境向けにレビューする Skill。原則としてコード変更ではなく、具体的な改善提案を行う。"
license: MIT
---

# .NET / C# Design Pattern Review — 日本語環境向け Skill

この Skill は、.NET / C# コードの設計パターン実装、アーキテクチャ、保守性、テスト容易性、セキュリティをレビューするために使う。レビューでは、単にパターン名を当てはめるのではなく、プロジェクトの規模、既存設計、運用要件に対して妥当かを判断する。

> **基本方針:** 設計パターンは目的ではなく手段。過剰設計を避け、依存関係を明確にし、テストしやすく、変更に強い構造になっているかを日本語で具体的に指摘する。レビュー依頼の場合は、明示されない限りコード変更せず、改善提案に留める。

---

## 1. まず確認すること

| 確認項目 | 見る場所 / 観点 |
|---|---|
| ソリューション構成 | `src\`, `tests\`, `.sln`, `.csproj` |
| レイヤー分離 | Domain, Application, Infrastructure, Presentation, Worker など |
| DI 構成 | `Program.cs`, `IServiceCollection` 拡張、service lifetime |
| 設定管理 | Options pattern, validation, secret 管理 |
| テスト | xUnit、test doubles、integration tests |
| 時刻依存 | `TimeProvider` を使っているか |
| AI 操作 | Microsoft Agent Framework を使っているか |
| ローカライズ | `ResourceManager`, `.resx`, `IStringLocalizer` |
| ログ | `ILogger<T>`、構造化ログ、OpenTelemetry |
| データアクセス | Repository / Unit of Work / DbContext の扱い |

既存のアーキテクチャ方針、命名規則、`Directory.Build.props`、`.editorconfig`、README、ADR がある場合は、それを基準にレビューする。

---

## 2. レビュー出力のルール

- 日本語で、重要度が高い順に指摘する。
- 「何が問題か」「なぜ問題か」「どう直すか」をセットで書く。
- 具体的なクラス名、メソッド名、責務境界を示す。
- パターン適用が不要な場合は「使わない方がよい」と明記する。
- スタイルだけの好みは指摘しない。保守性、正しさ、テスト容易性、セキュリティに影響するものを優先する。
- コード変更を求められていないレビューでは、原則として変更せず提案のみ行う。
- 既存の設計と衝突する提案は、移行コストと段階的対応案を書く。

指摘の形式:

```markdown
**重要度: 高**
`OrderService` が注文作成、DB 保存、通知送信、ログ整形をすべて担当しており、Single Responsibility Principle に反しています。
通知送信を `IOrderNotificationService` に分離し、`OrderService` はユースケースの調整に集中させると、xUnit で通知失敗時のテストを独立して書けます。
```

---

## 3. 必須レビュー観点

### Command / Command Handler

確認すること:

- Command が入力データを表し、Handler がユースケースを実行しているか。
- Handler が UI、HTTP、CLI、DB 実装に過度に依存していないか。
- validation、authorization、logging、transaction が一貫した場所にあるか。
- `CancellationToken` が下位処理へ渡されているか。
- async handler は `Task` / `Task<T>` を返しているか。

推奨:

```csharp
public interface ICommandHandler<TCommand, TResult>
{
    Task<TResult> HandleAsync(
        TCommand command,
        CancellationToken cancellationToken);
}
```

避けること:

- Handler が巨大な god class になる。
- Command にビジネスロジックを詰め込む。
- Handler が `DateTime.Now` や外部 API を直接呼ぶ。
- validation が controller、handler、repository に重複する。

---

### Factory

確認すること:

- 複雑な生成処理が呼び出し側に漏れていないか。
- DI で解決すべきものを Factory が service locator 的に隠していないか。
- disposable resource の所有権が明確か。
- 設定値や外部依存を安全に扱っているか。

Factory が有効な場面:

- 生成に複数ステップがある。
- runtime data に応じて実装を選ぶ。
- 外部 SDK client や agent を構成する。
- options validation 後に immutable object を作る。

避けること:

- 単に `new` を隠すだけの Factory。
- `IServiceProvider` をあちこちで直接使う Factory。
- lifetime 管理を曖昧にする Factory。

---

### Provider / Adapter

確認すること:

- 外部サービス、DB、ファイル、AI、メール、決済などが抽象化されているか。
- interface が利用者視点の contract になっているか。
- provider 固有の例外や DTO が application 層に漏れていないか。
- timeout、retry、circuit breaker、rate limit を検討しているか。

AI 操作では、Provider / Adapter の内側で Microsoft Agent Framework を使う。新規設計で Semantic Kernel を直接の標準にしない。

```csharp
public interface ICustomerSupportAgent
{
    Task<SupportAnswer> AskAsync(
        SupportQuestion question,
        CancellationToken cancellationToken);
}
```

実装側で `Microsoft.Agents.AI` と `Microsoft.Extensions.AI` を使い、アプリケーション層へ provider 固有の型を漏らさない。

---

### Repository

確認すること:

- Repository が永続化の抽象として妥当か。
- EF Core の `DbContext` を隠す必要が本当にあるか。
- query と command の責務が混ざりすぎていないか。
- async API と `CancellationToken` を使っているか。
- SQL はパラメーター化されているか。

Repository が有効な場面:

- 永続化の実装を差し替える可能性がある。
- ドメインモデル中心の集約単位で保存したい。
- test double で application service を単体テストしたい。

Repository が不要な場合:

- EF Core の `DbContext` がすでに適切な Unit of Work / Repository として機能している。
- 単純な CRUD を薄く包むだけで価値がない。

---

### Strategy

確認すること:

- `switch` や `if` が増え続ける条件分岐を strategy に分離できるか。
- strategy の選択条件が明確か。
- DI で複数実装を登録し、選択ロジックを集中できるか。
- 過剰に細かい strategy 分割になっていないか。

有効な場面:

- 支払い方法、配送方法、価格計算、認可ルール、AI provider 選択など。

---

### Template Method / Pipeline

確認すること:

- 共通処理と差分処理が明確か。
- 継承より composition / pipeline の方が適切ではないか。
- base class が大きくなりすぎていないか。
- Hook method が暗黙的すぎないか。

ASP.NET Core、MediatR 風 pipeline、middleware、decorator で表現できる場合は、継承ベースの Template Method より優先する。

---

### Resource / Localization

確認すること:

- ユーザー向け文言がリソース化されているか。
- ログメッセージとエラーメッセージが混在していないか。
- `.resx` のキー命名が一貫しているか。
- 日本語と英語の fallback を考慮しているか。
- コード内に日本語文言が散らばっていないか。

候補:

- ASP.NET Core: `IStringLocalizer<T>`
- ライブラリ: `ResourceManager`
- 用途別ファイル: `ErrorMessages.resx`, `LogMessages.resx`, `ValidationMessages.resx`

---

## 4. SOLID レビュー

| 原則 | 確認すること |
|---|---|
| Single Responsibility | クラスやメソッドの変更理由が 1 つに近いか |
| Open/Closed | 変更ではなく追加で振る舞いを増やせるか |
| Liskov Substitution | 派生型や実装が contract を破っていないか |
| Interface Segregation | 利用者が不要なメンバーに依存していないか |
| Dependency Inversion | 上位方針が下位実装に直接依存していないか |

典型的な指摘:

- service が repository 実装や external SDK client を直接生成している。
- interface が大きすぎて mock / fake が作りにくい。
- domain model が infrastructure の型に依存している。
- base class に共通化しすぎて変更影響が大きい。

---

## 5. Dependency Injection レビュー

確認すること:

- constructor injection を使っているか。
- service lifetime が妥当か。
- Singleton から Scoped を参照していないか。
- options と logger が適切に注入されているか。
- `IServiceProvider` 直接使用が局所化されているか。
- `TimeProvider.System` が DI 登録され、時刻依存 service に注入されているか。

推奨登録:

```csharp
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<ICommandHandler<CreateOrderCommand, OrderResult>, CreateOrderHandler>();
```

レビューで注意するアンチパターン:

- service locator 化した `IServiceProvider`。
- static singleton。
- 依存を optional にしすぎる constructor。
- null を許す曖昧な dependency。

---

## 6. TimeProvider レビュー

現在時刻は `DateTime.Now`、`DateTime.UtcNow`、`DateTimeOffset.Now`、`DateTimeOffset.UtcNow` を直接使わず、`TimeProvider` から取得する。

確認すること:

- application service / domain service に `TimeProvider` が注入されているか。
- 保存・比較は UTC を基本にしているか。
- 表示時だけ `Asia/Tokyo` などへ変換しているか。
- xUnit テストで `FakeTimeProvider` を使っているか。
- timer を使う場合に破棄が考慮されているか。

指摘例:

```markdown
`SubscriptionService` が `DateTimeOffset.Now` に直接依存しているため、期限切れ判定の境界値テストが不安定になります。
`TimeProvider` を注入し、テストでは `FakeTimeProvider` で現在時刻を固定してください。
```

---

## 7. Microsoft Agent Framework レビュー

AI 操作がある場合は Microsoft Agent Framework の利用を前提にレビューする。

確認すること:

- `Microsoft.Agents.AI` と `Microsoft.Extensions.AI` の抽象を使っているか。
- `ChatClientAgent` や Agent Framework の orchestration に責務が集約されているか。
- application 層に provider 固有の SDK 型が漏れていないか。
- prompt、tools、structured output、model settings が分離されているか。
- prompt injection、tool abuse、data exfiltration を考慮しているか。
- secret が構成・環境変数・Key Vault などで扱われているか。
- AI 呼び出しが xUnit テストで fake / stub に差し替えられるか。
- observability と監査ログの範囲が適切か。

避けること:

- controller から直接 model API を呼ぶ。
- AI 応答を検証せずに DB 更新や外部送信へ使う。
- tool に過剰な権限を渡す。
- system prompt とユーザー入力を同じ信頼境界で扱う。
- 実モデル依存の単体テストを書く。

---

## 8. xUnit によるテスト容易性レビュー

テストは xUnit を前提にする。

確認すること:

- 依存関係が interface / abstract class / delegate で差し替え可能か。
- 外部 API、DB、時刻、乱数、ファイルシステムが制御可能か。
- async API が `Task` / `Task<T>` を返しているか。
- `CancellationToken` のテストが可能か。
- `FakeTimeProvider` で時刻依存テストを書けるか。
- テスト名が `MethodName_Scenario_ExpectedBehavior` になっているか。
- Arrange-Act-Assert で読みやすいか。

設計上の赤信号:

- constructor 内で外部通信を開始する。
- static state に依存する。
- private method のテストが必要になるほど public behavior が見えにくい。
- テストのためだけに production code のアクセス修飾子を緩めている。

---

## 9. 設定と Options レビュー

確認すること:

- strongly-typed options を使っているか。
- `ValidateDataAnnotations()`、`ValidateOnStart()` などで起動時検証しているか。
- secret と通常設定が分離されているか。
- 環境ごとの差分が管理可能か。
- AI model deployment、endpoint、timeout、retry 設定が options に整理されているか。

推奨:

```csharp
builder.Services
    .AddOptions<AgentOptions>()
    .Bind(builder.Configuration.GetSection("Agent"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

---

## 10. 非同期・リソース管理レビュー

確認すること:

- I/O 処理が async 化されているか。
- `.Result`、`.Wait()`、`GetAwaiter().GetResult()` を使っていないか。
- `CancellationToken` を受け取り、下位へ伝播しているか。
- `IDisposable` / `IAsyncDisposable` の所有権が明確か。
- stream、timer、DB connection、HTTP response が適切に破棄されているか。
- `HttpClient` は `IHttpClientFactory` を使っているか。

ライブラリコードでは `ConfigureAwait(false)` の必要性を既存方針に合わせて確認する。

---

## 11. セキュリティレビュー

確認すること:

- 入力検証が境界で行われているか。
- SQL injection を避けるためパラメーター化されているか。
- secret がソース、ログ、例外、プロンプトに出ていないか。
- 例外詳細をユーザーへ返していないか。
- AI tool execution が最小権限になっているか。
- 権限チェックが UI だけでなく server / application 層にもあるか。

AI 関連の追加観点:

- ユーザー入力が system / developer instructions を上書きできないか。
- tool 実行前に authorization があるか。
- 機密データを model に送る前に分類・マスキングしているか。

---

## 12. ドキュメントレビュー

確認すること:

- public API に XML docs があるか。
- `<param>`、`<returns>`、`<exception>` が必要十分か。
- 設計パターンを採用した理由が README / ADR にあるか。
- 日本語の運用手順があるか。
- AI 機能の制約、データ利用、監査方針が説明されているか。

パターンが複雑な場合は、コードだけでなく短い設計メモを残すことを提案する。

---

## 13. 過剰設計の検出

次の兆候があれば、パターンを減らす提案をする。

- interface が実装 1 つで、差し替え予定もテスト上の価値もない。
- Factory、Provider、Manager、Service が同じ責務を薄く包んでいるだけ。
- 継承階層が深く、処理の流れを追いにくい。
- 汎用化されすぎて domain language が失われている。
- テストが mock の設定だらけで振る舞いが読めない。
- 小規模機能なのに巨大な framework 風構造になっている。

提案は「単純化しても失われない品質」を明確にする。

---

## 14. レビュー時チェックリスト

- [ ] 責務境界が明確
- [ ] Command / Handler がユースケース単位に整理されている
- [ ] Factory が複雑な生成だけを担当している
- [ ] Provider / Adapter が外部依存を隠蔽している
- [ ] Repository が価値のある抽象になっている
- [ ] Strategy や Pipeline が条件分岐の増殖を防いでいる
- [ ] SOLID 原則に大きな違反がない
- [ ] DI lifetime が妥当
- [ ] `TimeProvider` を使い、直接の現在時刻参照がない
- [ ] AI 操作は Microsoft Agent Framework に集約されている
- [ ] テストは xUnit で書きやすい構造
- [ ] secret、個人情報、prompt injection への配慮がある
- [ ] 日本語文言は適切にリソース化されている
- [ ] public API に XML docs がある
- [ ] 過剰設計になっていない

---

## 15. よくあるレビュー指摘テンプレート

### 責務分離

```markdown
`{ClassName}` が `{ResponsibilityA}` と `{ResponsibilityB}` の両方を担当しており、変更理由が複数あります。
`{ResponsibilityB}` を `{NewServiceName}` に分離すると、`{ClassName}` は `{PrimaryUseCase}` に集中でき、xUnit で `{Scenario}` を独立して検証できます。
```

### TimeProvider

```markdown
`{MethodName}` が `{CurrentTimeApi}` を直接使っているため、時刻境界のテストが不安定になります。
`TimeProvider` を DI で注入し、テストでは `FakeTimeProvider` を使って現在時刻を固定してください。
```

### Microsoft Agent Framework

```markdown
AI 呼び出しが `{ClassName}` に直接埋め込まれており、provider 差し替え、テスト、監査が難しくなっています。
Microsoft Agent Framework を使う `{AgentServiceName}` / `{AgentAdapterName}` に集約し、application 層は domain contract にだけ依存する形にしてください。
```

### 過剰設計

```markdown
`{PatternName}` の適用によりクラス数が増えていますが、現時点では差し替え・再利用・テスト容易性の利益が小さいです。
まず `{SimplerApproach}` に単純化し、分岐や実装差し替えが増えた段階で pattern を導入する方が保守しやすいです。
```

---

## 16. 公式参照

| リソース | URL |
|---|---|
| .NET documentation | https://learn.microsoft.com/ja-jp/dotnet/ |
| C# documentation | https://learn.microsoft.com/ja-jp/dotnet/csharp/ |
| Dependency injection in .NET | https://learn.microsoft.com/dotnet/core/extensions/dependency-injection |
| Options pattern | https://learn.microsoft.com/dotnet/core/extensions/options |
| Logging in .NET | https://learn.microsoft.com/dotnet/core/extensions/logging |
| TimeProvider overview | https://learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview |
| Microsoft Agent Framework | https://learn.microsoft.com/agent-framework/overview/agent-framework-overview |
| xUnit | https://xunit.net/ |
