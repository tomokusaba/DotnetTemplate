---
name: dotnet-timezone
description: ".NET / C# のタイムゾーン処理を日本語環境向けに支援する Skill。TimeZoneInfo、TimeZoneConverter、NodaTime、TimeProvider、UTC 保存、Asia/Tokyo、Windows/IANA ID、DST、スケジューリング、API 設計、住所・都市からのタイムゾーン解決を扱う。"
license: MIT
---

# .NET Timezone — 日本語環境向け Skill

この Skill は、.NET / C# アプリケーションでタイムゾーン、UTC 変換、スケジューリング、DST、Windows / IANA タイムゾーン ID の違いを安全に扱うために使う。回答と手順は日本語を基本にし、日本国内運用、Windows 開発、Linux コンテナ、Azure などの混在環境を考慮する。

> **基本方針:** DB と API 境界では UTC または `DateTimeOffset` を使い、表示直前にユーザーのタイムゾーンへ変換する。現在時刻は `DateTime.Now` / `DateTimeOffset.Now` ではなく `TimeProvider` から取得する。日本向け既定の表示タイムゾーンは `Asia/Tokyo` / `Tokyo Standard Time` を候補にする。

---

## 1. まず判断すること

依頼内容を最初に分類する。

| 依頼タイプ | 対応 |
|---|---|
| 住所・都市・国からタイムゾーンを知りたい | `references/timezone-index.md` を参照し、Windows ID と IANA ID を返す |
| Windows / Linux 両対応の変換コードが欲しい | `TimeZoneConverter` を推奨 |
| 厳密な DST・繰り返し予定・カレンダー計算 | `NodaTime` を推奨 |
| API / DB 設計 | UTC 保存、`DateTimeOffset` 受け渡し、表示時変換を推奨 |
| ASP.NET Core / DI 対応 | `TimeProvider` と options による設計を推奨 |
| 既存コードレビュー | `DateTime.Now`、固定 offset、platform 固有 ID を重点確認 |

ライブラリ選択が曖昧な場合、Windows / Linux / コンテナ混在では `TimeZoneConverter` を既定にする。DST による曖昧時刻・存在しない時刻・繰り返し予定が重要な場合は `NodaTime` を優先する。

---

## 2. 日本語環境でのルール

- 説明、手順、注意点は日本語で書く。
- タイムゾーン ID、型名、メソッド名は英語の正式名称で書く。
- 日本国内の標準タイムゾーンは次を使う。
  - Windows ID: `Tokyo Standard Time`
  - IANA ID: `Asia/Tokyo`
  - UTC offset: `+09:00`
  - DST: なし
- 日本向け表示では `Asia/Tokyo` を候補にするが、ユーザー・拠点・予定ごとにタイムゾーンを保持できる設計を優先する。
- DB には日本時間の文字列ではなく UTC instant を保存する。
- `DateTimeKind.Unspecified` は、ユーザー入力のローカル時刻など意図が明確な場合だけ使う。
- Windows では Windows ID、Linux / コンテナ / NodaTime では IANA ID が基本になる。
- Azure Windows と Azure Linux では期待されるタイムゾーン ID が異なる可能性がある。

---

## 3. 参照資料

必要に応じて必ず次を参照する。

| Reference | 用途 |
|---|---|
| `references/timezone-index.md` | 主要地域の Windows ID / IANA ID / UTC offset / DST 有無 |
| `references/code-patterns.md` | .NET での実装パターン、TimeProvider、TimeZoneConverter、NodaTime、DST 対応 |

---

## 4. 住所・都市・地域からタイムゾーンを解決する

ユーザーが住所、都市、都道府県、国、ドキュメント内の地名を渡した場合:

1. 入力から地名を抽出する。
2. `references/timezone-index.md` の日本・主要地域・都市別メモを確認する。
3. 完全一致がなければ地理的に妥当な IANA ID を推定する。
4. Windows ID と IANA ID の両方を返す。
5. 必要なら copy-paste 可能な C# コードを添える。

返答形式:

```text
Location: 東京, 日本
Windows ID: Tokyo Standard Time
IANA ID: Asia/Tokyo
UTC offset: +09:00
DST: No
```

曖昧な地名は候補を出して確認する。例: `Springfield`、`San José`、`London` など。

---

## 5. タイムゾーン ID の扱い

常に Windows ID と IANA ID の両方を意識する。

| 用途 | 推奨 ID |
|---|---|
| Windows の `TimeZoneInfo.FindSystemTimeZoneById()` | Windows ID |
| Linux / Alpine / コンテナ | IANA ID |
| NodaTime | IANA ID |
| TimeZoneConverter | Windows ID / IANA ID のどちらも可 |
| 外部 API と保存値 | IANA ID を推奨 |
| 日本固定の表示 | `Asia/Tokyo` を推奨 |

クロスプラットフォームでは、`TimeZoneInfo.FindSystemTimeZoneById("Asia/Tokyo")` を直接使うのではなく、`TimeZoneConverter` の `TZConvert.GetTimeZoneInfo()` を使う。

---

## 6. コード生成方針

実装コードは `references/code-patterns.md` から最小のパターンを選ぶ。

| パターン | 使う場面 |
|---|---|
| Pattern 1: `TimeProvider` + `TimeZoneInfo` | Windows-only など platform が固定されている |
| Pattern 2: `TimeZoneConverter` | Windows / Linux / コンテナをまたぐ一般的な .NET アプリ |
| Pattern 3: `NodaTime` | DST とカレンダー計算の正確性が重要 |
| Pattern 4: `DateTimeOffset` | API、イベント、データ転送 |
| Pattern 5: ASP.NET Core persistence / presentation | DB 保存とユーザー表示の分離 |
| Pattern 6: recurring jobs | ユーザー指定のローカル時刻で繰り返し実行 |
| Pattern 7: DST ambiguous / invalid local time | DST による重複・欠落時刻を扱う |

サードパーティライブラリを勧める場合は、必ず NuGet package を明記する。

---

## 7. 推奨デフォルト

### アプリケーションコード

- 現在時刻は `TimeProvider.GetUtcNow()`。
- 保存は UTC。
- API 境界では `DateTimeOffset` を優先。
- 表示時にユーザーの IANA time zone ID で変換。
- Windows / Linux 両対応は `TimeZoneConverter`。

```xml
<PackageReference Include="TimeZoneConverter" Version="6.*" />
```

```csharp
using TimeZoneConverter;

public sealed class UserTimeService(TimeProvider timeProvider)
{
    public DateTimeOffset GetUserLocalNow(string userTimeZoneId)
    {
        TimeZoneInfo timeZone = TZConvert.GetTimeZoneInfo(userTimeZoneId);
        return TimeZoneInfo.ConvertTime(timeProvider.GetUtcNow(), timeZone);
    }
}
```

### DI 登録

```csharp
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<UserTimeService>();
```

### テスト

```xml
<PackageReference Include="Microsoft.Extensions.TimeProvider.Testing" Version="..." />
```

```csharp
using Microsoft.Extensions.Time.Testing;

var timeProvider = new FakeTimeProvider(
    new DateTimeOffset(2026, 4, 26, 0, 0, 0, TimeSpan.Zero));
```

---

## 8. API と DB 設計

推奨:

- 作成日時、更新日時、イベント発生日時は UTC で保存する。
- ユーザーの表示タイムゾーンは IANA ID で保存する。
- API では `DateTimeOffset` を使い、offset を含める。
- 「毎日 9:00」などの予定は UTC instant だけでなく、local time と timezone ID を保存する。

例:

```csharp
public sealed record ScheduledJob(
    TimeOnly LocalTime,
    string TimeZoneId,
    DateTimeOffset? LastRunAtUtc);
```

避ける:

- DB に `2026/04/26 09:00 JST` のような表示文字列だけを保存する。
- offset `+09:00` だけを保存してタイムゾーン ID を失う。
- `DateTime.Now` を保存する。
- すべてのユーザーが日本時間であると仮定する。

---

## 9. スケジューリングと DST

繰り返し予定では、UTC 変換済みの単発時刻だけでは不十分な場合がある。

保存すべき情報:

- ユーザーが入力したローカル日付・時刻
- タイムゾーン ID
- 繰り返しルール
- 次回実行 UTC
- DST の曖昧時刻・無効時刻の扱い方

DST がある地域では、ローカル時刻が存在しない、または 2 回存在することがある。`references/code-patterns.md` の Pattern 7 を使って `IsInvalidTime()` / `IsAmbiguousTime()` を確認する。

日本は DST がないが、海外拠点・海外ユーザー・海外 SaaS 連携がある場合は DST を前提に設計する。

---

## 10. よくある落とし穴

- `TimeZoneInfo.FindSystemTimeZoneById()` に渡せる ID は OS によって異なる。
- Linux コンテナでは Windows ID が見つからないことがある。
- `DateTime.Now` をサーバーコードで使うとテストと運用で不安定になる。
- DB にローカル時刻を保存すると、利用者の地域変更や DST で破綻しやすい。
- 固定 offset `+09:00` はタイムゾーンではない。
- `DateTimeKind.Unspecified` を意図なく使うと変換バグの原因になる。
- DST の切り替わりでは、存在しないローカル時刻と重複するローカル時刻が発生する。
- 日本時間だけで設計すると、将来の海外展開や外部 API 連携で修正範囲が大きくなる。

---

## 11. レビュー時チェックリスト

- [ ] 現在時刻は `TimeProvider` から取得している
- [ ] `DateTime.Now` / `DateTimeOffset.Now` を使っていない
- [ ] 保存値は UTC または offset 付き instant
- [ ] 表示時にユーザーのタイムゾーンへ変換している
- [ ] Windows ID と IANA ID の違いを考慮している
- [ ] Linux / コンテナで動くコードになっている
- [ ] 日本の既定値は `Asia/Tokyo` / `Tokyo Standard Time`
- [ ] ユーザーや拠点ごとのタイムゾーンを保存できる
- [ ] DST の曖昧時刻・無効時刻を考慮している
- [ ] 繰り返し予定で local time と timezone ID を保持している
- [ ] xUnit テストで `FakeTimeProvider` を使える

---

## 12. 回答形式

住所・地域の依頼:

1. 解決した location block を返す。
2. 推奨実装を 1 文で述べる。
3. C# コードを最小構成で添える。

コード・設計の依頼:

1. 推奨アプローチを 1 文で述べる。
2. 関係する Windows ID / IANA ID を示す。
3. 最小コードを提示する。
4. 必要な NuGet package を明記する。
5. 該当する落とし穴を 1 つ以上添える。

---

## 13. 公式参照

| リソース | URL |
|---|---|
| TimeZoneInfo | https://learn.microsoft.com/dotnet/api/system.timezoneinfo |
| TimeProvider | https://learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview |
| Testing with FakeTimeProvider | https://learn.microsoft.com/dotnet/core/extensions/timeprovider-testing |
| TimeZoneConverter | https://github.com/mattjohnsonpint/TimeZoneConverter |
| NodaTime | https://nodatime.org/ |
| IANA Time Zone Database | https://www.iana.org/time-zones |
