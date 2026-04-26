---
name: ef-core
description: "Entity Framework Core の DbContext 設計、Entity 設計、LINQ クエリ、性能、Migration、トランザクション、セキュリティ、xUnit テストを日本語環境向けに支援する Skill。SQL Server / SQLite / PostgreSQL、Windows/PowerShell、UTF-8、TimeProvider、公式 Microsoft Learn 参照を扱う。"
license: MIT
---

# Entity Framework Core — 日本語環境向け Skill

この Skill は、Entity Framework Core を使う .NET アプリケーションで、保守しやすく安全で性能のよいデータアクセス層を設計・実装・レビューするために使う。回答と手順は日本語を基本にし、API 名、型名、メソッド名、列名、ログの構造化キーは英語を基本にする。

> **基本方針:** DbContext は短命な Unit of Work として扱い、クエリは必要なデータだけを取得し、Migration は小さくレビュー可能にする。テストは xUnit を使い、現在時刻は `DateTime.Now` / `DateTimeOffset.Now` ではなく `TimeProvider` から取得する。

---

## 1. まず確認すること

作業前に、既存プロジェクトの方針を確認する。

| 確認項目 | 見る場所 / コマンド |
|---|---|
| .NET SDK / Target Framework | `global.json`, `dotnet --info`, `.csproj` |
| EF Core version | `Directory.Packages.props`, `.csproj`, `dotnet list package` |
| DB provider | `Microsoft.EntityFrameworkCore.SqlServer`, `Npgsql.EntityFrameworkCore.PostgreSQL`, `Microsoft.EntityFrameworkCore.Sqlite` など |
| DbContext 登録 | `Program.cs`, `AddDbContext`, `AddDbContextFactory`, `AddPooledDbContextFactory` |
| Migration 配置 | `Migrations\`, `dotnet ef migrations list` |
| 接続文字列 | `appsettings.json`, user secrets, Key Vault, 環境変数 |
| テスト方針 | `tests\`, xUnit, Testcontainers, LocalDB, SQLite in-memory |
| 時刻依存 | `DateTime.Now`, `DateTimeOffset.Now`, `TimeProvider` |
| CI/CD | GitHub Actions, Azure Pipelines, migration script / bundle |

既存の命名規則、レイヤー構成、Repository / Unit of Work 方針、監査列の扱いがある場合はそれを優先する。

---

## 2. 日本語環境でのルール

- 手順、PR 説明、レビューコメント、運用メモは日本語で説明する。
- Entity / DbContext / Migration / 列名 / index 名は英語を基本にする。
- 日本語文字列を保存・検索する場合は、DB collation、正規化、大小文字・全角半角・かな違いを明示的に検討する。
- Windows 手順では PowerShell と `\` 区切りのパスを優先する。
- 接続文字列、パスワード、アクセストークンをコードや Migration に含めない。
- 保存する日時は原則 UTC の `DateTimeOffset` を使い、表示時に `Asia/Tokyo` などへ変換する。
- 現在時刻は `TimeProvider.GetUtcNow()` を使い、テストでは `FakeTimeProvider` を使う。

---

## 3. DbContext 設計

推奨:

- DbContext は短命な Unit of Work として扱う。
- ASP.NET Core では `AddDbContext<TContext>()` の scoped 登録を基本にする。
- Console app、background job、Blazor、並列処理、テストでは `IDbContextFactory<TContext>` を検討する。
- Entity 設定は `IEntityTypeConfiguration<TEntity>` に分離する。
- `OnModelCreating` では `ApplyConfigurationsFromAssembly` を使い、設定を集約する。
- DbContext にドメインロジックや UI ロジックを入れない。
- `SaveChangesAsync` / `ToListAsync` など async API を使い、`CancellationToken` を渡す。

例:

```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

DI 登録:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

避けること:

- Singleton service に DbContext を直接注入する。
- DbContext を static / global に保持する。
- 1 つの DbContext インスタンスで並列クエリを実行する。
- `OnConfiguring` に本番接続文字列を直書きする。
- `DbContext` を repository のように巨大化させる。

---

## 4. Entity 設計

推奨:

- 主キー、外部キー、必須項目、最大長、精度、index を明示する。
- 値オブジェクトには owned entity type を検討する。
- 多対多、1 対多、1 対 1 の関係を Fluent API で明確に表現する。
- 更新競合が起こり得る集約には concurrency token / rowversion を使う。
- 監査列は UTC で保存し、`TimeProvider` 経由で値を設定する。
- 金額は `decimal` と精度指定を使う。
- 列名や制約名が運用上重要な場合は明示する。

例:

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(order => order.Id);

        builder.Property(order => order.CustomerName)
            .HasMaxLength(100)
            .IsRequired();

        builder.Property(order => order.TotalAmount)
            .HasPrecision(18, 2);

        builder.Property(order => order.CreatedAt)
            .IsRequired();

        builder.HasIndex(order => order.CreatedAt);
    }
}
```

---

## 5. クエリ設計

推奨:

- 読み取り専用クエリでは `AsNoTracking()` を使う。
- 画面や API に必要な列だけ `Select` で projection する。
- 大きな一覧では `OrderBy` と `Skip` / `Take` でページングする。
- `Include` は必要な関連だけに限定し、過剰取得を避ける。
- N+1 問題を避けるため、必要に応じて eager loading / projection を使う。
- 複雑な画面では DTO projection を優先する。
- provider 固有関数を使う場合は、その provider でテストする。
- `IQueryable` をレイヤー外へ漏らす場合は、評価タイミングと責務境界を明確にする。

例:

```csharp
public async Task<IReadOnlyList<OrderSummary>> GetRecentOrdersAsync(
    int page,
    int pageSize,
    CancellationToken cancellationToken)
{
    return await dbContext.Orders
        .AsNoTracking()
        .OrderByDescending(order => order.CreatedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(order => new OrderSummary(
            order.Id,
            order.CustomerName,
            order.TotalAmount,
            order.CreatedAt))
        .ToListAsync(cancellationToken);
}
```

避けること:

- `ToListAsync()` 後にメモリ上で大量 filtering / sorting する。
- `Include` を深く連鎖して過剰な object graph を取得する。
- ページングで `OrderBy` を指定しない。
- 文字列連結で raw SQL を組み立てる。
- `AsEnumerable()` で意図せず client evaluation に切り替える。

---

## 6. 性能

確認ポイント:

- query plan と index 利用を確認する。
- `AsNoTracking()` / `AsNoTrackingWithIdentityResolution()` を適切に使う。
- 頻繁に実行される hot path では compiled query を検討する。
- `ExecuteUpdateAsync` / `ExecuteDeleteAsync` が使える更新・削除は検討する。
- 大量 insert / update は SaveChanges の回数と transaction 境界を見直す。
- `Include` による cartesian explosion がある場合は split query を検討する。
- ログで生成 SQL を確認する。ただし本番で sensitive data logging は有効化しない。

性能改善では、推測だけで変更しない。対象 DB の実行計画、index、データ量、生成 SQL を確認する。

---

## 7. Migration

推奨:

- Migration は小さく、1 つの意図に集中させる。
- Migration 名は英語で具体的にする。
- production 適用前に SQL script を生成してレビューする。
- 複数環境へ適用する場合は idempotent script を検討する。
- CI/CD では migration bundle も検討する。
- 列 rename / table rename は drop & create になっていないか確認する。
- data migration は量、ロック、再実行性、rollback を考慮する。
- `EnsureCreatedAsync()` と Migrations を混在させない。

PowerShell:

```powershell
dotnet tool restore
dotnet ef migrations add AddOrderAuditColumns --project src\MyApp.Infrastructure --startup-project src\MyApp.Api
dotnet ef migrations script --idempotent --project src\MyApp.Infrastructure --startup-project src\MyApp.Api --output artifacts\migrations.sql
dotnet ef migrations bundle --project src\MyApp.Infrastructure --startup-project src\MyApp.Api --output artifacts\efbundle.exe
```

注意:

- production でアプリ起動時に `Database.MigrateAsync()` を呼ぶ運用は慎重に判断する。
- EF Core 9 以降は migration locking があるが、production では SQL script / bundle のレビュー可能性を優先する。
- rollback script が必要な場合は `dotnet ef migrations script FromMigration ToMigration` を使う。

---

## 8. 変更追跡と保存

推奨:

- 1 request / 1 use case の範囲で DbContext を使う。
- `SaveChangesAsync` は必要な単位でまとめる。
- 複数集約の整合性が必要な場合は明示的な transaction を使う。
- optimistic concurrency を使う場合は `DbUpdateConcurrencyException` を設計上扱う。
- 監査列の設定は SaveChanges interceptor や application service で一貫させる。
- soft delete や tenant filter は global query filter を検討する。

例:

```csharp
public sealed class AuditableAppDbContext(
    DbContextOptions<AuditableAppDbContext> options,
    TimeProvider timeProvider)
    : DbContext(options)
{
    public override async Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        var now = timeProvider.GetUtcNow();

        foreach (var entry in ChangeTracker.Entries<IAuditableEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = now;
            }

            if (entry.State is EntityState.Added or EntityState.Modified)
            {
                entry.Entity.UpdatedAt = now;
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

---

## 9. セキュリティ

- raw SQL は必要な場合だけ使う。
- user input を文字列連結して SQL に埋め込まない。
- パラメーター化された `FromSql` / `FromSqlInterpolated` を使う。
- 接続文字列は user secrets、環境変数、Key Vault などで管理する。
- 本番 DB user は最小権限にする。
- `EnableSensitiveDataLogging()` は開発時だけに限定する。
- multi-tenant では tenant ID を必ず server-side で検証する。
- global query filter は便利だが、認可の唯一の防御線にしない。
- PII をログや例外メッセージに出さない。

---

## 10. テスト

テストは xUnit を使う。

推奨方針:

- クエリの正しさは、可能なら実際の production provider と同じ DB でテストする。
- SQL Server なら LocalDB / container、PostgreSQL なら Testcontainers を検討する。
- test double が必要な場合は、Repository 層を stub / mock する。
- SQLite in-memory は軽量な integration test に使えるが、provider 差異を理解して使う。
- EF Core InMemory provider は relational DB と挙動が異なるため、原則として新規テストでは避ける。
- `DbSet` の query mock は LINQ provider 差異を隠すため避ける。
- Migration は isolated database で適用確認する。
- 時刻依存の entity / query は `FakeTimeProvider` で固定する。

SQLite in-memory の例:

```csharp
public sealed class AppDbContextTests : IAsyncLifetime
{
    private readonly SqliteConnection connection = new("Data Source=:memory:");
    private AppDbContext dbContext = default!;

    public async Task InitializeAsync()
    {
        await connection.OpenAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(connection)
            .Options;

        dbContext = new AppDbContext(options);
        await dbContext.Database.EnsureCreatedAsync();
    }

    [Fact]
    public async Task Orders_WhenInserted_CanBeRead()
    {
        dbContext.Orders.Add(new Order("山田 太郎", 1200m));
        await dbContext.SaveChangesAsync();

        var order = await dbContext.Orders.SingleAsync();

        Assert.Equal("山田 太郎", order.CustomerName);
    }

    public async Task DisposeAsync()
    {
        await dbContext.DisposeAsync();
        await connection.DisposeAsync();
    }
}
```

注意:

- SQLite と SQL Server / PostgreSQL は文字列比較、日付関数、SQL 方言、制約、型の挙動が異なる。
- `EnsureCreatedAsync()` はテスト DB 初期化には使えるが、Migration 運用の代替ではない。
- 並列実行するテストでは DB 名や schema を分離する。

---

## 11. レビュー時チェックリスト

- [ ] DbContext の lifetime が適切
- [ ] Singleton から scoped DbContext を参照していない
- [ ] Entity 設定が `IEntityTypeConfiguration` などで整理されている
- [ ] 必須項目、最大長、精度、index、関係が明示されている
- [ ] 読み取り専用クエリで `AsNoTracking()` を検討している
- [ ] projection で必要な列だけ取得している
- [ ] ページングに安定した `OrderBy` がある
- [ ] N+1 / 過剰 Include がない
- [ ] raw SQL がパラメーター化されている
- [ ] Migration が小さく、SQL script をレビューできる
- [ ] production 接続文字列や secret が含まれていない
- [ ] `EnableSensitiveDataLogging()` が本番で有効にならない
- [ ] テストは xUnit を使っている
- [ ] EF Core InMemory provider に依存しすぎていない
- [ ] 時刻依存コードは `TimeProvider` を使っている

---

## 12. 公式参照

| リソース | URL |
|---|---|
| EF Core documentation | https://learn.microsoft.com/ef/core/ |
| DbContext configuration | https://learn.microsoft.com/ef/core/dbcontext-configuration/ |
| Efficient querying | https://learn.microsoft.com/ef/core/performance/efficient-querying |
| Testing strategy | https://learn.microsoft.com/ef/core/testing/choosing-a-testing-strategy |
| Applying migrations | https://learn.microsoft.com/ef/core/managing-schemas/migrations/applying |
| EF Core CLI tools | https://learn.microsoft.com/ef/core/cli/dotnet |
| Tracking vs. no-tracking queries | https://learn.microsoft.com/ef/core/querying/tracking |
