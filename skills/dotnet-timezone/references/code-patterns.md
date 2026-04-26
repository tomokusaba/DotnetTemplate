# .NET Timezone Code Patterns — 日本語環境向け

このファイルは、.NET / C# でタイムゾーンを扱うときの実装パターン集。現在時刻が必要なコードでは `DateTime.Now` や `DateTimeOffset.Now` ではなく `TimeProvider` を使う。

## Pattern 1: TimeProvider + TimeZoneInfo

Windows-only など、実行 OS と timezone ID 形式が固定されている場合だけ使う。

```csharp
public sealed class WindowsTimeZoneService(TimeProvider timeProvider)
{
    public DateTimeOffset GetTokyoNow()
    {
        DateTimeOffset utcNow = timeProvider.GetUtcNow();
        TimeZoneInfo tokyoTimeZone =
            TimeZoneInfo.FindSystemTimeZoneById("Tokyo Standard Time");

        return TimeZoneInfo.ConvertTime(utcNow, tokyoTimeZone);
    }
}
```

Linux、コンテナ、Windows / Linux 混在では `TimeZoneConverter` または `NodaTime` を使う。

## Pattern 2: Cross-platform with TimeZoneConverter

Windows / Linux / コンテナをまたぐ .NET アプリでは既定で推奨する。

```xml
<PackageReference Include="TimeZoneConverter" Version="6.*" />
```

```csharp
using TimeZoneConverter;

public sealed class UserTimeService(TimeProvider timeProvider)
{
    public DateTimeOffset ConvertUtcNowToUserTime(string userTimeZoneId)
    {
        TimeZoneInfo timeZone = TZConvert.GetTimeZoneInfo(userTimeZoneId);
        return TimeZoneInfo.ConvertTime(timeProvider.GetUtcNow(), timeZone);
    }
}
```

`TZConvert.GetTimeZoneInfo()` は Windows ID と IANA ID の両方を受け取れる。

```csharp
TimeZoneInfo tokyoFromIana = TZConvert.GetTimeZoneInfo("Asia/Tokyo");
TimeZoneInfo tokyoFromWindows = TZConvert.GetTimeZoneInfo("Tokyo Standard Time");
```

## Pattern 3: NodaTime

DST、カレンダー計算、繰り返し予定の正確性が重要な場合に使う。

```xml
<PackageReference Include="NodaTime" Version="3.*" />
```

```csharp
using NodaTime;

public sealed class NodaTimeZoneService(IClock clock)
{
    public ZonedDateTime GetTokyoNow()
    {
        DateTimeZone tokyoZone = DateTimeZoneProviders.Tzdb["Asia/Tokyo"];
        Instant now = clock.GetCurrentInstant();
        return now.InZone(tokyoZone);
    }
}
```

ローカル時刻を厳密にゾーンへ割り当てる例:

```csharp
DateTimeZone newYorkZone = DateTimeZoneProviders.Tzdb["America/New_York"];
LocalDateTime localDateTime = new(2026, 3, 8, 2, 30, 0);

ZonedDateTime zoned = newYorkZone.AtLeniently(localDateTime);
Instant instant = zoned.ToInstant();
```

曖昧時刻・無効時刻に対して業務ルールがある場合は、`AtStrictly`、`AtLeniently`、resolver を明示的に選ぶ。

## Pattern 4: DateTimeOffset for APIs

サービスやプロセス境界を越える値には `DateTimeOffset` を優先する。

```csharp
public sealed record EventDto(
    string EventId,
    DateTimeOffset OccurredAt);
```

UTC を返す API:

```csharp
public sealed class EventService(TimeProvider timeProvider)
{
    public EventDto CreateEvent(string eventId)
    {
        return new EventDto(eventId, timeProvider.GetUtcNow());
    }
}
```

ユーザー表示用変換:

```csharp
using TimeZoneConverter;

public static DateTimeOffset ToUserTime(
    DateTimeOffset utcInstant,
    string userTimeZoneId)
{
    TimeZoneInfo timeZone = TZConvert.GetTimeZoneInfo(userTimeZoneId);
    return TimeZoneInfo.ConvertTime(utcInstant, timeZone);
}
```

## Pattern 5: ASP.NET Core persistence and presentation

DB には UTC を保存し、画面や API response でユーザータイムゾーンへ変換する。

```csharp
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<UserTimeService>();
```

```csharp
using TimeZoneConverter;

public sealed class AuditService(
    TimeProvider timeProvider,
    UserTimeService userTimeService)
{
    public AuditLog CreateLog(string userId)
    {
        return new AuditLog
        {
            UserId = userId,
            CreatedAtUtc = timeProvider.GetUtcNow()
        };
    }

    public DateTimeOffset ToDisplayTime(
        DateTimeOffset createdAtUtc,
        string userTimeZoneId)
    {
        return userTimeService.ConvertUtcToUserTime(createdAtUtc, userTimeZoneId);
    }
}

public sealed class UserTimeService
{
    public DateTimeOffset ConvertUtcToUserTime(
        DateTimeOffset utcInstant,
        string userTimeZoneId)
    {
        TimeZoneInfo timeZone = TZConvert.GetTimeZoneInfo(userTimeZoneId);
        return TimeZoneInfo.ConvertTime(utcInstant, timeZone);
    }
}
```

## Pattern 6: Scheduling and recurring jobs

ユーザーが入力した「毎日 9:00」などの予定では、local time と timezone ID を保存する。

```csharp
public sealed record RecurringSchedule(
    TimeOnly LocalTime,
    string TimeZoneId,
    string CronExpression);
```

単発のローカル時刻を UTC へ変換する:

```csharp
using TimeZoneConverter;

TimeZoneInfo timeZone = TZConvert.GetTimeZoneInfo("Asia/Tokyo");
DateTime scheduledLocal = new(2026, 4, 26, 9, 0, 0, DateTimeKind.Unspecified);
DateTime scheduledUtc = TimeZoneInfo.ConvertTimeToUtc(scheduledLocal, timeZone);
```

Hangfire 例:

```csharp
using TimeZoneConverter;

TimeZoneInfo timeZone = TZConvert.GetTimeZoneInfo("Asia/Tokyo");

RecurringJob.AddOrUpdate(
    "morning-job",
    () => DoWork(),
    "0 9 * * *",
    new RecurringJobOptions { TimeZone = timeZone });
```

## Pattern 7: Ambiguous and invalid DST times

DST がある地域では、ローカル時刻が存在しない、または 2 回存在することがある。

```csharp
using TimeZoneConverter;

TimeZoneInfo timeZone = TZConvert.GetTimeZoneInfo("America/New_York");
DateTime localTime = new(2026, 11, 1, 1, 30, 0, DateTimeKind.Unspecified);

if (timeZone.IsInvalidTime(localTime))
{
    // 業務ルール例: 存在しない時刻は 1 時間後へ寄せる。
    localTime = localTime.AddHours(1);
}

if (timeZone.IsAmbiguousTime(localTime))
{
    TimeSpan[] offsets = timeZone.GetAmbiguousTimeOffsets(localTime);
    TimeSpan selectedOffset = offsets.Min();
    DateTimeOffset resolved = new(localTime, selectedOffset);
}
```

曖昧時刻の解決方法は業務要件として明文化する。

## Pattern 8: xUnit with FakeTimeProvider

時刻依存テストでは `FakeTimeProvider` を使い、実時間に依存しない。

```xml
<PackageReference Include="Microsoft.Extensions.TimeProvider.Testing" Version="..." />
```

```csharp
using Microsoft.Extensions.Time.Testing;

public sealed class UserTimeServiceTests
{
    [Fact]
    public void ConvertUtcNowToUserTime_WhenTokyo_ReturnsJst()
    {
        // Arrange
        var utcNow = new DateTimeOffset(2026, 4, 26, 0, 0, 0, TimeSpan.Zero);
        var timeProvider = new FakeTimeProvider(utcNow);
        var service = new UserTimeService(timeProvider);

        // Act
        DateTimeOffset result = service.ConvertUtcNowToUserTime("Asia/Tokyo");

        // Assert
        Assert.Equal(TimeSpan.FromHours(9), result.Offset);
        Assert.Equal(9, result.Hour);
    }
}
```

## Common mistakes

| Wrong | Better |
|---|---|
| `DateTime.Now` in server code | `TimeProvider.GetUtcNow()` |
| `DateTimeOffset.Now` in business logic | Inject `TimeProvider` |
| Storing local timestamps in the database | Store UTC and convert for display |
| Hardcoding offsets such as `+09:00` | Store timezone ID such as `Asia/Tokyo` |
| Using `FindSystemTimeZoneById("Asia/Tokyo")` on Windows | Use `TZConvert.GetTimeZoneInfo("Asia/Tokyo")` |
| Comparing local `DateTime` values from different zones | Compare UTC instants or use `DateTimeOffset` |
| Creating `DateTime` without intentional kind semantics | Use UTC, offset, or deliberate `Unspecified` |

## Decision guide

- Use `TimeZoneInfo` only when OS and timezone ID format are controlled.
- Use `TimeZoneConverter` for most cross-platform applications.
- Use `NodaTime` when DST arithmetic or calendar accuracy is central.
- Use `DateTimeOffset` for APIs and serialized timestamps.
- Use `TimeProvider` for current time and tests.
