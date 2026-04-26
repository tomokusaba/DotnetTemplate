---
name: csharp-xunit
description: "C# / .NET の xUnit テスト作成を日本語環境で支援する Skill。Arrange-Act-Assert、Fact/Theory、データ駆動テスト、非同期テスト、モック、テスト命名、Windows/PowerShell、UTF-8、Asia/Tokyo、ロケール依存の注意点を扱う。"
license: MIT
---

# C# xUnit — 日本語環境向けテスト Skill

この Skill は、C# / .NET プロジェクトで xUnit を使った単体テスト、データ駆動テスト、例外テスト、非同期テスト、モックを含むテスト設計を支援する。回答と手順は原則として日本語で行い、日本語環境・Windows 開発・CI で安定して動くテストを優先する。

> **基本方針:** テストは仕様を説明する実行可能なドキュメント。日本語の説明はコメントや表示名に使ってよいが、テストメソッド名、型名、カテゴリ名、ログの構造化キーは CI や検索性を考えて英語を基本にする。

---

## 1. まず確認すること

作業前に、プロジェクトの既存方針を確認する。

| 確認項目 | 見る場所 / コマンド |
|---|---|
| .NET SDK バージョン | `global.json`, `dotnet --info` |
| テストプロジェクト | `*.Tests.csproj`, `tests\` 配下 |
| xUnit バージョン | `xunit`, `xunit.runner.visualstudio` |
| テスト SDK | `Microsoft.NET.Test.Sdk` |
| アサーション方針 | 標準 `Assert`、FluentAssertions など |
| モック方針 | Moq、NSubstitute、FakeItEasy など |
| CI 実行方法 | GitHub Actions、Azure Pipelines、`dotnet test` |
| カルチャ / タイムゾーン依存 | `CultureInfo`, `TimeProvider`, `DateTimeOffset` |

既存のテストスタイルがある場合は、それを優先する。新規追加時は、最小限の依存で xUnit 標準機能を使い、必要がある場合のみモック・アサーションライブラリを追加する。

---

## 2. 日本語環境でのルール

- 手順、PR 説明、テスト観点は日本語で説明する。
- テストメソッド名は英語を基本にし、`MethodName_Scenario_ExpectedBehavior` を推奨する。
- 日本語の期待値を検証するテストでは、ファイルと文字列を UTF-8 前提で扱う。
- 日付・時刻は `DateTime.Now` に依存せず、`TimeProvider`、`DateTimeOffset`、固定クロックを使う。
- ロケール依存の比較では `CultureInfo.InvariantCulture` または明示した `CultureInfo("ja-JP")` を使う。
- Windows/PowerShell での手順は `\` 区切りのパスを使う。
- テストが実行順に依存しないようにする。
- 日本語メッセージの完全一致テストは壊れやすいため、必要な場合だけ行う。通常はエラーコード、型、状態、構造化データを検証する。

---

## 3. プロジェクト構成

テストプロジェクトは本体プロジェクトと分離する。

```text
src\
  MyApp\
    MyApp.csproj
tests\
  MyApp.Tests\
    MyApp.Tests.csproj
```

一般的なパッケージ:

```xml
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="..." />
<PackageReference Include="xunit" Version="..." />
<PackageReference Include="xunit.runner.visualstudio" Version="..." />
```

テスト対象プロジェクトを参照する。

```xml
<ProjectReference Include="..\..\src\MyApp\MyApp.csproj" />
```

パッケージのバージョンは既存の `Directory.Packages.props`、`Directory.Build.props`、Central Package Management の方針を優先する。

---

## 4. テスト構造

xUnit では、単純なテストに `[Fact]`、データ駆動テストに `[Theory]` を使う。

```csharp
public sealed class CalculatorTests
{
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        var result = calculator.Add(2, 3);

        // Assert
        Assert.Equal(5, result);
    }
}
```

### Arrange-Act-Assert

| フェーズ | 内容 |
|---|---|
| Arrange | 入力、依存関係、テスト対象を準備する |
| Act | テスト対象の操作を 1 つ実行する |
| Assert | 期待される結果を検証する |

1 つのテストでは 1 つの振る舞いに集中する。複数の振る舞いをまとめて検証しない。

---

## 5. 命名規則

推奨パターン:

```text
MethodName_Scenario_ExpectedBehavior
```

例:

```csharp
public void Parse_ValidJapaneseDate_ReturnsDateOnly()
public void CreateOrder_EmptyItems_ThrowsValidationException()
public async Task GetUserAsync_UserDoesNotExist_ReturnsNull()
```

テストクラスは対象クラス名に `Tests` を付ける。

| 対象 | テストクラス |
|---|---|
| `UserService` | `UserServiceTests` |
| `OrderValidator` | `OrderValidatorTests` |
| `JapaneseDateParser` | `JapaneseDateParserTests` |

---

## 6. データ駆動テスト

複数の入力と期待値を検証する場合は `[Theory]` を使う。

### InlineData

```csharp
public sealed class TaxCalculatorTests
{
    [Theory]
    [InlineData(1000, 100)]
    [InlineData(2500, 250)]
    [InlineData(0, 0)]
    public void CalculateConsumptionTax_ValidAmount_ReturnsTax(
        int amount,
        int expectedTax)
    {
        var calculator = new TaxCalculator();

        var result = calculator.CalculateConsumptionTax(amount);

        Assert.Equal(expectedTax, result);
    }
}
```

### MemberData

日本語文字列や複雑な型を扱う場合は `MemberData` を使う。

```csharp
public sealed class NameFormatterTests
{
    public static TheoryData<string, string, string> JapaneseNames =>
        new()
        {
            { "山田", "太郎", "山田 太郎" },
            { "佐藤", "花子", "佐藤 花子" }
        };

    [Theory]
    [MemberData(nameof(JapaneseNames))]
    public void Format_JapaneseName_ReturnsFullName(
        string familyName,
        string givenName,
        string expected)
    {
        var formatter = new NameFormatter();

        var result = formatter.Format(familyName, givenName);

        Assert.Equal(expected, result);
    }
}
```

### ClassData

大きなデータセットや再利用するデータは `ClassData` を検討する。ただし、読みづらくなる場合は `TheoryData` を優先する。

---

## 7. アサーション

標準の xUnit アサーションを優先する。

| 目的 | アサーション |
|---|---|
| 値の一致 | `Assert.Equal(expected, actual)` |
| 参照の一致 | `Assert.Same(expected, actual)` |
| 真偽値 | `Assert.True(value)`, `Assert.False(value)` |
| null 確認 | `Assert.Null(value)`, `Assert.NotNull(value)` |
| コレクション | `Assert.Contains`, `Assert.DoesNotContain`, `Assert.Collection` |
| 例外 | `Assert.Throws<T>`, `Assert.ThrowsAsync<T>` |
| 正規表現 | `Assert.Matches`, `Assert.DoesNotMatch` |

複雑なオブジェクトの可読性を高める必要がある場合のみ、既存方針に合わせて FluentAssertions などを使う。

---

## 8. 例外テスト

例外の型を優先して検証する。メッセージ全文の完全一致は、ローカライズで壊れやすいため避ける。

```csharp
[Fact]
public void Create_EmptyName_ThrowsArgumentException()
{
    var exception = Assert.Throws<ArgumentException>(
        () => User.Create(""));

    Assert.Equal("name", exception.ParamName);
}
```

非同期の場合:

```csharp
[Fact]
public async Task GetAsync_InvalidId_ThrowsArgumentException()
{
    var repository = new UserRepository();

    var exception = await Assert.ThrowsAsync<ArgumentException>(
        () => repository.GetAsync(""));

    Assert.Equal("id", exception.ParamName);
}
```

---

## 9. 非同期テスト

- テストメソッドは `async Task` にする。
- `async void` は使わない。
- `.Result` や `.Wait()` はデッドロックや例外ラップの原因になるため避ける。
- タイムアウトが必要な場合は `CancellationToken` やテストフレームワークの timeout 方針を確認する。

```csharp
[Fact]
public async Task SendAsync_ValidRequest_ReturnsAccepted()
{
    var client = new NotificationClient();

    var result = await client.SendAsync("user@example.com", "こんにちは");

    Assert.Equal(NotificationStatus.Accepted, result.Status);
}
```

---

## 10. セットアップと後片付け

xUnit ではテストクラスのコンストラクターが各テストごとに実行される。

```csharp
public sealed class FileExporterTests : IDisposable
{
    private readonly string _tempDirectory;

    public FileExporterTests()
    {
        _tempDirectory = Path.Combine(Path.GetTempPath(), Path.GetRandomFileName());
        Directory.CreateDirectory(_tempDirectory);
    }

    [Fact]
    public void Export_ValidData_WritesUtf8File()
    {
        var exporter = new FileExporter(_tempDirectory);

        exporter.Export("report.txt", "日本語");

        var path = Path.Combine(_tempDirectory, "report.txt");
        Assert.Equal("日本語", File.ReadAllText(path, Encoding.UTF8));
    }

    public void Dispose()
    {
        Directory.Delete(_tempDirectory, recursive: true);
    }
}
```

共有コンテキスト:

| 用途 | xUnit 機能 |
|---|---|
| 1 テストクラス内で共有 | `IClassFixture<T>` |
| 複数テストクラスで共有 | `ICollectionFixture<T>` |
| テストごとの初期化 | コンストラクター |
| テストごとの後片付け | `IDisposable` / `IAsyncLifetime` |

---

## 11. モックと分離

外部依存はインターフェイスや抽象化を通して差し替える。

モックを使う対象:

- 外部 API
- メール送信
- 現在時刻
- ファイルシステム
- DB / キャッシュ
- ランダム値や ID 採番

避けること:

- 実ネットワークに依存する単体テスト
- 現在時刻や実タイムゾーンに依存するテスト
- テスト実行順に依存する共有状態
- 日本語 OS / 英語 OS の違いで壊れるカルチャ未指定処理

既存プロジェクトにモックライブラリがなければ、まず手書き fake / stub を検討する。追加する場合はチームの標準に合わせる。

---

## 12. カルチャ・タイムゾーン・文字コード

日本語環境では次の観点を明示的にテストする。

| 観点 | 推奨 |
|---|---|
| 日付表示 | `CultureInfo("ja-JP")` を明示 |
| 内部保存 | UTC または `DateTimeOffset` |
| 現在時刻 | `TimeProvider` や固定クロック |
| 数値 / 通貨 | `CultureInfo` を明示 |
| 文字コード | UTF-8 を明示 |
| 改行 | `Environment.NewLine` または正規化 |
| パス | `Path.Combine` を使う |

```csharp
[Fact]
public void FormatDate_JapaneseCulture_ReturnsJapaneseDate()
{
    var culture = CultureInfo.GetCultureInfo("ja-JP");
    var date = new DateOnly(2026, 4, 26);

    var result = date.ToString("yyyy年M月d日", culture);

    Assert.Equal("2026年4月26日", result);
}
```

---

## 13. カテゴリと診断出力

カテゴリ分けには `[Trait]` を使う。

```csharp
[Trait("Category", "Unit")]
[Fact]
public void Validate_ValidInput_ReturnsSuccess()
{
    // ...
}
```

テスト中の診断出力には `ITestOutputHelper` を使う。

```csharp
public sealed class ImporterTests
{
    private readonly ITestOutputHelper _output;

    public ImporterTests(ITestOutputHelper output)
    {
        _output = output;
    }

    [Fact]
    public void Import_InvalidRow_WritesDiagnosticOutput()
    {
        _output.WriteLine("invalid row を検証します。");
    }
}
```

CI ログに機密情報や個人情報を出力しない。

---

## 14. 実行コマンド

Windows PowerShell:

```powershell
dotnet restore
dotnet build
dotnet test
```

特定プロジェクトだけ実行:

```powershell
dotnet test .\tests\MyApp.Tests\MyApp.Tests.csproj
```

フィルター実行:

```powershell
dotnet test --filter "FullyQualifiedName~UserServiceTests"
dotnet test --filter "Category=Unit"
```

詳細ログ:

```powershell
dotnet test --logger "console;verbosity=detailed"
```

---

## 15. レビュー時のチェックリスト

- [ ] テスト名が振る舞いを説明している
- [ ] Arrange-Act-Assert が読み取りやすい
- [ ] 1 テストが 1 つの振る舞いに集中している
- [ ] テストが実行順に依存していない
- [ ] 外部 API、時刻、乱数、ファイル、DB などの依存が制御されている
- [ ] 日本語文字列、UTF-8、改行、パス区切りの違いが考慮されている
- [ ] カルチャやタイムゾーンが必要に応じて明示されている
- [ ] 例外テストがメッセージ全文ではなく型やパラメーターを検証している
- [ ] `async Task` を使い、`.Result` / `.Wait()` を避けている
- [ ] `dotnet test` で安定して通る

---

## 16. よくある依頼への対応

### テストを新規追加する

1. 既存のテスト構成と命名規則を確認
2. 対象クラスに対応する `*Tests` クラスを作成
3. 主要な正常系、異常系、境界値を洗い出す
4. 単純なケースは `[Fact]`、複数入力は `[Theory]` にする
5. `dotnet test` で確認する

### 既存テストを改善する

1. 何を検証しているかをテスト名に反映
2. 不要な複数アサーションを分割
3. 実時刻・実ファイル・実ネットワーク依存を排除
4. データ駆動テストで重複を整理
5. CI で不安定な要因を取り除く

### 日本語入力のテストを追加する

1. UTF-8 で保存されていることを確認
2. `CultureInfo("ja-JP")` が必要か判断
3. 全角・半角、ひらがな・カタカナ、濁点、サロゲートペアを必要に応じて含める
4. メッセージ全文ではなく仕様上重要な値を検証
5. Windows と Linux CI の差を考慮する
