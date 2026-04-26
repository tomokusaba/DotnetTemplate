---
name: microsoft-code-reference
description: "Microsoft / Azure / .NET SDK の API リファレンス、公式コードサンプル、メソッド署名、パッケージ名、認証、移行、非推奨 API を日本語環境で検証する Skill。Microsoft Learn MCP、mslearn CLI、C#、PowerShell、Azure CLI、Python、TypeScript を扱う。"
license: MIT
---

# Microsoft Code Reference — 日本語環境向け Skill

この Skill は、Microsoft / Azure / .NET / C# / Microsoft Agent Framework などの SDK コードを書く前に、公式 API リファレンスと公式コードサンプルで正しさを確認するために使う。存在しないメソッド、誤った overload、古い SDK pattern、非推奨 API、間違った package 名を防ぐことを目的とする。

> **基本方針:** Microsoft 関連コードを生成・修正する前に、Microsoft Learn の公式 docs / code samples で API 名、namespace、package、method signature、認証方式、SDK version を確認する。日本語環境では `ja-jp` を優先しつつ、API 名や最新仕様は `en-us` も確認する。

---

## 1. まず確認すること

| 確認項目 | 観点 |
|---|---|
| 対象 SDK | Azure SDK、.NET BCL、ASP.NET Core、Microsoft Graph、Agent Framework など |
| 言語 | C#、PowerShell、Azure CLI、Python、TypeScript、JavaScript |
| package 名 | NuGet、npm、pip、PowerShell module |
| API 名 | class、method、property、extension method |
| namespace | `Azure.Storage.Blobs`, `Microsoft.Extensions.AI` など |
| version | SDK v11/v12、.NET 8/9/10、preview / stable |
| 認証 | `DefaultAzureCredential`、API key、managed identity、RBAC |
| 実行環境 | Windows PowerShell、Linux、Azure、GitHub Actions |
| 日本語要件 | `ja-jp` docs、UTF-8、コメント・説明文 |

---

## 2. 使う tools

| 目的 | Tool | 例 |
|---|---|---|
| API / class / method を探す | `microsoft_docs_search` | `BlobClient UploadAsync Azure.Storage.Blobs` |
| 公式コード例を探す | `microsoft_code_sample_search` | `upload blob managed identity`, language: `csharp` |
| overload / parameter を全文確認 | `microsoft_docs_fetch` | search で見つけた API reference URL |

コードを書く前の推奨フロー:

1. `microsoft_docs_search` で API / package の存在を確認する。
2. overload や parameter が重要なら `microsoft_docs_fetch` で全文を確認する。
3. `microsoft_code_sample_search` で実装パターンを確認する。
4. コードに package、using、認証、例外、制限を反映する。

---

## 3. 公式コードサンプル検索

`microsoft_code_sample_search` を使って、実際に動く公式サンプルを探す。

例:

```text
query: "upload file to blob storage", language: "csharp"
query: "DefaultAzureCredential BlobServiceClient", language: "csharp"
query: "authenticate managed identity key vault", language: "python"
query: "send message service bus", language: "javascript"
query: "IChatClient ChatClientAgent", language: "csharp"
```

使う場面:

- 新しい SDK を初めて使う。
- 初期化・認証・DI の正しい pattern を知りたい。
- compile error / runtime error の原因を公式例と比較したい。
- 古い SDK pattern から新しい SDK pattern へ移行したい。
- Azure RBAC や managed identity が関係する。

---

## 4. API lookup のクエリ例

精度を上げるため、class、method、namespace、package を含める。

```text
BlobClient UploadAsync Azure.Storage.Blobs
BlobServiceClient DefaultAzureCredential Azure.Identity
GraphServiceClient Users Microsoft.Graph
TimeProvider FakeTimeProvider Microsoft.Extensions.TimeProvider.Testing
ChatClientAgent Microsoft.Agents.AI IChatClient
IStringLocalizer ASP.NET Core localization
```

package 確認:

```text
Azure Blob Storage NuGet package Azure.Storage.Blobs
Microsoft Graph .NET SDK NuGet package
Microsoft Agent Framework Microsoft.Agents.AI NuGet
FakeTimeProvider NuGet Microsoft.Extensions.TimeProvider.Testing
```

---

## 5. 検証が必須の場面

必ず公式 docs で確認する。

- メソッド名が便利すぎる、または記憶ベースで曖昧。
- overload が多い。
- SDK v11 と v12 など major version が混在している。
- `CloudBlobClient` と `BlobServiceClient` のような旧新 SDK が混ざっている。
- package 名が曖昧。
- preview / rc package を使う。
- Azure RBAC が必要。
- managed identity を使う。
- Microsoft Agent Framework など API 変化が速い領域。
- GitHub Copilot や AI が生成した API 名が実在するか不明。

---

## 6. よくあるエラーの調査クエリ

| エラー | クエリ |
|---|---|
| method not found | `[ClassName] [MethodName] methods [Namespace]` |
| type not found | `[TypeName] NuGet package namespace` |
| wrong signature | `[ClassName] [MethodName] overloads` |
| obsolete warning | `[OldType] migration [SDK version]` |
| auth failure | `DefaultAzureCredential troubleshooting` |
| 401 / 403 | `[ServiceName] RBAC permissions managed identity` |
| package mismatch | `[PackageName] migration guide` |
| DI registration | `[ServiceName] dependency injection .NET` |

例:

```text
BlobClient UploadAsync overloads Azure.Storage.Blobs
DefaultAzureCredential troubleshooting Azure Identity
Azure Key Vault RBAC permissions SecretClient
Microsoft Graph Users request adapter v5 migration
```

---

## 7. 日本語ページと英語ページの使い分け

日本語環境では `ja-jp` の Microsoft Learn を優先する。

ただし、次の場合は `en-us` も確認する。

- API reference の翻訳が遅れている。
- preview / latest 機能。
- package version が新しい。
- 例外名・method 名・parameter 名を正確に確認したい。
- 日本語訳で意味が曖昧。

回答では、API 名とコードは英語の正式名で書く。

---

## 8. コード生成ルール

Microsoft 関連コードを提示するとき:

- package 名を明記する。
- `using` / import を明記する。
- 認証方式を明記する。
- secret を直書きしない。
- Azure では可能なら `DefaultAzureCredential` と managed identity を優先する。
- RBAC role が必要なら触れる。
- async API を優先する。
- .NET の時刻依存コードは `TimeProvider` を使う。
- テスト例は xUnit を使う。
- preview API は preview と明記する。
- 古い SDK からの移行なら旧 API と新 API を区別する。

例:

```text
NuGet package: Azure.Storage.Blobs
Authentication: DefaultAzureCredential
Required role: Storage Blob Data Contributor
```

---

## 9. .NET / Azure SDK の注意

典型的な混同:

| 古い / 誤りやすい | 新しい / 確認すべき |
|---|---|
| `CloudBlobClient` | `BlobServiceClient` |
| `Microsoft.Azure.*` legacy packages | `Azure.*` modern SDK |
| connection string 直書き | `DefaultAzureCredential` / managed identity |
| sync I/O | async API |
| `DateTime.Now` | `TimeProvider.GetUtcNow()` |
| 実 model に依存する unit test | fake / stub + xUnit |

SDK major version により API が大きく変わるため、version を必ず確認する。

---

## 10. CLI 代替手段

Microsoft Learn MCP が使えない場合は `mslearn` CLI を使う。

PowerShell:

```powershell
npx @microsoft/learn-cli search "BlobClient UploadAsync Azure.Storage.Blobs"
npx @microsoft/learn-cli code-search "upload blob managed identity" --language csharp
npx @microsoft/learn-cli fetch "https://learn.microsoft.com/ja-jp/dotnet/api/azure.storage.blobs.blobclient"
```

global install:

```powershell
npm install -g @microsoft/learn-cli
mslearn search "ChatClientAgent Microsoft.Agents.AI"
mslearn code-search "DefaultAzureCredential BlobServiceClient" --language csharp
mslearn fetch "https://learn.microsoft.com/ja-jp/dotnet/"
```

JSON:

```powershell
mslearn search "FakeTimeProvider TimeProvider" --json | ConvertFrom-Json
```

---

## 11. 回答形式

コード参照を求められたら、必要に応じて次の形で答える。

````markdown
**推奨 API:** `BlobClient.UploadAsync`
**Package:** `Azure.Storage.Blobs`
**Namespace:** `Azure.Storage.Blobs`
**認証:** `DefaultAzureCredential`
**注意:** `CloudBlobClient` は旧 SDK の型です。

```csharp
// code
```

公式参照:
- https://learn.microsoft.com/...
````

不確かな API は、確認できるまで断言しない。

---

## 12. セキュリティ

- token、connection string、API key をコード例に含めない。
- `"<connection-string>"` のような placeholder を使う。
- Azure 本番環境では managed identity と RBAC を優先する。
- サンプルの tenant ID、subscription ID、endpoint は placeholder にする。
- エラーログを扱う場合は secret を除去する。

---

## 13. レビュー時チェックリスト

- [ ] 公式 docs で API / method / package を確認した
- [ ] overload と parameter を確認した
- [ ] 公式コードサンプルを確認した
- [ ] SDK version / package version が明確
- [ ] 古い SDK pattern と混在していない
- [ ] secret をコードに含めていない
- [ ] 認証方式と RBAC を考慮している
- [ ] async / cancellation を考慮している
- [ ] .NET では xUnit と TimeProvider 方針に合っている
- [ ] 公式 URL を示している

---

## 14. 公式参照

| リソース | URL |
|---|---|
| Microsoft Learn | https://learn.microsoft.com/ja-jp/ |
| .NET API browser | https://learn.microsoft.com/dotnet/api/ |
| Azure SDK for .NET | https://learn.microsoft.com/dotnet/azure/sdk/ |
| Azure SDK releases | https://azure.github.io/azure-sdk/releases/latest/ |
| Microsoft Learn CLI | https://www.npmjs.com/package/@microsoft/learn-cli |
| Azure Identity | https://learn.microsoft.com/dotnet/api/overview/azure/identity-readme |
| Microsoft.Extensions.AI | https://learn.microsoft.com/dotnet/ai/microsoft-extensions-ai |
