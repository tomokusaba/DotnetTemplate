---
name: csharp-docs
description: "C# / .NET の XML ドキュメントコメント、API ドキュメント、サンプルコード、C# 13 / C# 14 など新しい言語仕様の説明を日本語環境向けに支援する Skill。日本語の文体、UTF-8、cref、inheritdoc、DocFX、NuGet 公開、LangVersion 互換性を扱う。"
license: MIT
---

# C# ドキュメント — 日本語環境向け Skill

この Skill は、C# / .NET の型、メンバー、API、サンプルコードを XML ドキュメントコメントや Markdown で説明するときに使う。日本語チームで読みやすく、IDE 補完・DocFX・NuGet パッケージ・CI でも扱いやすいドキュメントを作ることを目的とする。

> **基本方針:** ドキュメントは利用者の判断を助ける仕様情報を書く。実装を読めば分かることを繰り返すのではなく、「何をするか」「いつ使うか」「何を返すか」「どの条件で例外になるか」「言語バージョンや実行環境の前提」を明確にする。

---

## 1. まず確認すること

| 確認項目 | 見る場所 / コマンド |
|---|---|
| .NET SDK | `global.json`, `dotnet --info` |
| C# 言語バージョン | `.csproj` の `LangVersion`, SDK 既定値 |
| XML docs 出力 | `.csproj` の `GenerateDocumentationFile` |
| 警告方針 | `NoWarn`, `WarningsAsErrors`, CS1591 |
| ドキュメント生成 | DocFX, Sandcastle, README, Wiki, Learn 風 Markdown |
| 公開対象 | 社内 API、OSS、NuGet、ライブラリ、アプリ内部 |
| 表記言語 | 日本語のみ、日英併記、英語 API + 日本語説明 |
| 新言語機能 | C# 13 / C# 14 を使える SDK・CI・IDE か |

既存のスタイル、`.editorconfig`、`Directory.Build.props`、CI の警告設定がある場合はそれを優先する。

---

## 2. 日本語環境での記述ルール

- 説明文は日本語を基本にする。
- 型名、メソッド名、パラメーター名、キーワード、例外型、構造化ログキーは英語のまま記載する。
- XML ドキュメントタグは標準タグを使う。独自タグは既存ツールが対応している場合だけ使う。
- ファイルは UTF-8 を前提にする。
- 日本語文は簡潔にし、1 文を長くしすぎない。
- 「します」「返します」など、利用者視点で現在形にする。
- 実装詳細は必要な場合のみ `<remarks>` に分ける。
- 外部公開ライブラリでは、グローバル利用者がいる場合に英語版ドキュメントも検討する。
- 新しい C# 構文をサンプルに使う場合は、必要な .NET SDK / C# バージョンを明記する。

---

## 3. XML ドキュメントの基本

公開 API には XML ドキュメントコメントを付ける。複雑な内部 API や再利用される internal メンバーにも付ける。

```csharp
/// <summary>
/// 指定されたユーザー ID に対応するユーザーを取得します。
/// </summary>
/// <param name="userId">取得するユーザーの ID。</param>
/// <param name="cancellationToken">操作をキャンセルするためのトークン。</param>
/// <returns>ユーザーが存在する場合はユーザー情報。それ以外の場合は <see langword="null" />。</returns>
/// <exception cref="ArgumentException"><paramref name="userId" /> が空です。</exception>
public Task<User?> GetUserAsync(
    string userId,
    CancellationToken cancellationToken = default);
```

よく使うタグ:

| タグ | 用途 |
|---|---|
| `<summary>` | 型やメンバーの短い説明 |
| `<remarks>` | 補足、制約、使用上の注意、実装上の重要事項 |
| `<param>` | メソッド引数の説明 |
| `<paramref>` | 説明文中で引数名を参照 |
| `<typeparam>` | ジェネリック型引数の説明 |
| `<typeparamref>` | 説明文中で型引数を参照 |
| `<returns>` | 戻り値の説明 |
| `<value>` | プロパティ値の説明 |
| `<exception>` | 直接送出される例外 |
| `<see cref>` | 型やメンバーへの参照 |
| `<see langword>` | `null`, `true`, `false`, `default` などのキーワード |
| `<seealso>` | 関連 API への参照 |
| `<inheritdoc />` | 基底型・インターフェイスの説明を継承 |
| `<example>` | 使用例 |
| `<code>` | コード例 |
| `<c>` | インラインコード |

---

## 4. `<summary>` の書き方

`<summary>` は、1 文または短い 2 文で「何をするか」を書く。

推奨:

```csharp
/// <summary>
/// 注文の合計金額を計算します。
/// </summary>
public Money CalculateTotal();
```

避ける例:

```csharp
/// <summary>
/// CalculateTotal メソッドです。
/// </summary>
```

型の場合:

```csharp
/// <summary>
/// 注文の作成と状態遷移を管理します。
/// </summary>
public sealed class OrderService
{
}
```

コンストラクターの場合:

```csharp
/// <summary>
/// <see cref="OrderService" /> クラスの新しいインスタンスを初期化します。
/// </summary>
public OrderService(IOrderRepository repository)
{
}
```

---

## 5. メソッドのドキュメント

### パラメーター

`<param>` は型名を繰り返さず、意味と制約を書く。

```csharp
/// <param name="amount">税込金額を計算する対象の税抜金額。</param>
```

Boolean の場合:

```csharp
/// <param name="includeInactive">
/// 非アクティブなユーザーを含める場合は <see langword="true" />。それ以外の場合は <see langword="false" />。
/// </param>
```

`out` パラメーターの場合:

```csharp
/// <param name="value">
/// このメソッドが戻るとき、変換に成功した値が格納されます。このパラメーターは初期化されていないものとして扱われます。
/// </param>
```

### 戻り値

戻り値は「何が返るか」を書く。戻り値の型名だけを繰り返さない。

```csharp
/// <returns>
/// 変換に成功した場合は <see langword="true" />。それ以外の場合は <see langword="false" />。
/// </returns>
public bool TryParse(string text, out int value);
```

---

## 6. プロパティのドキュメント

読み取り専用:

```csharp
/// <summary>
/// ユーザーの表示名を取得します。
/// </summary>
public string DisplayName { get; }
```

読み書き可能:

```csharp
/// <summary>
/// 監査ログを有効にするかどうかを示す値を取得または設定します。
/// </summary>
/// <value>
/// 監査ログを有効にする場合は <see langword="true" />。それ以外の場合は <see langword="false" />。既定値は <see langword="false" /> です。
/// </value>
public bool EnableAuditLog { get; set; }
```

プロパティが副作用を持つ場合は、`<remarks>` に明記する。

---

## 7. 例外のドキュメント

メンバーが直接送出する例外は `<exception>` に書く。ネストした内部実装から送出される例外は、利用者が遭遇しやすく対応すべきものだけを書く。

```csharp
/// <exception cref="ArgumentNullException"><paramref name="options" /> が <see langword="null" /> です。</exception>
/// <exception cref="InvalidOperationException">必要な接続情報が構成されていません。</exception>
public Client CreateClient(ClientOptions options);
```

避ける表現:

```xml
<exception cref="ArgumentNullException">options が null の場合にスローされます。</exception>
```

推奨:

```xml
<exception cref="ArgumentNullException"><paramref name="options" /> が <see langword="null" /> です。</exception>
```

---

## 8. `<see>` と `<inheritdoc />`

型、メンバー、パラメーター、キーワードは XML タグで参照する。

```csharp
/// <summary>
/// <see cref="IClock" /> を使用して現在時刻を取得します。
/// </summary>
```

インターフェイスや基底クラスと同じ意味の実装では `<inheritdoc />` を使う。

```csharp
/// <inheritdoc />
public sealed class SystemClock : IClock
{
    /// <inheritdoc />
    public DateTimeOffset GetUtcNow() => DateTimeOffset.UtcNow;
}
```

挙動が基底定義と異なる場合は、`<inheritdoc />` だけで済ませず、差分を `<remarks>` に書く。

---

## 9. サンプルコードと `<example>`

サンプルは実行可能に近い形で、対象 API の典型的な使い方を示す。

```csharp
/// <example>
/// 次の例では、注文の合計金額を計算します。
/// <code language="csharp">
/// var calculator = new OrderCalculator();
/// var total = calculator.CalculateTotal(order);
/// Console.WriteLine(total);
/// </code>
/// </example>
```

サンプルコードでは次を守る。

- 対象プロジェクトの C# バージョンでコンパイルできる構文にする。
- 新しい構文を使う場合は、必要な C# / .NET バージョンを説明する。
- シークレット、実接続文字列、個人情報を含めない。
- 日本語文字列を含める場合は UTF-8 前提で扱う。

---

## 10. C# 13 の新しい言語仕様に関する言及

C# 13 は .NET 9 でサポートされる。ドキュメントやサンプルで C# 13 機能を使う場合は、対象 SDK、`LangVersion`、CI の対応状況を確認する。

主な機能:

| 機能 | ドキュメント時の注意 |
|---|---|
| `params` コレクション | 配列以外の `Span<T>`、`ReadOnlySpan<T>`、コレクション型を受け取れることを説明する |
| `System.Threading.Lock` | `lock` 対象が新しい `Lock` 型である場合のセマンティクスを明記する |
| `\e` エスケープシーケンス | ANSI escape などで使う場合、C# 13 以降が必要と書く |
| メソッドグループ自然型の改善 | オーバーロードや型推論の説明で、旧バージョンとの差分に注意する |
| オブジェクト初期化子での `^` インデックス | サンプルが C# 13 以降専用であることを明記する |
| async / iterator での `ref` / `unsafe` 緩和 | `await` / `yield` 境界を越えられない制約を書く |
| `allows ref struct` | ジェネリック制約として ref safety が関係することを説明する |
| `ref struct` のインターフェイス実装 | boxing できない制約や呼び出し方法を補足する |
| partial プロパティ / インデクサー | generator や partial 型で使う場合、定義宣言と実装宣言を説明する |
| オーバーロード解決優先順位 | ライブラリ作成者向け機能として互換性意図を書く |
| `field` キーワード | C# 13 ではプレビュー扱い。C# 14 以降との差分と名前衝突に注意する |

例:

```csharp
/// <summary>
/// 指定された値を連結して表示します。
/// </summary>
/// <remarks>
/// この API は C# 13 の <c>params</c> コレクションを使用するため、.NET 9 SDK 以降が必要です。
/// </remarks>
public void Concat<T>(params ReadOnlySpan<T> items)
{
}
```

---

## 11. C# 14 の新しい言語仕様に関する言及

C# 14 は .NET 10 でサポートされる。新しい構文をドキュメントや README のサンプルに使う場合は、読者が .NET 10 SDK と対応 IDE を使えることを前提として明記する。

主な機能:

| 機能 | ドキュメント時の注意 |
|---|---|
| 拡張メンバー | 拡張メソッドだけでなく拡張プロパティ、静的拡張メンバー、演算子を説明できる |
| null 条件付き割り当て | 左辺に `?.` / `?[]` を使えること、右辺評価が null で抑止されることを明記する |
| `nameof(List<>)` | バインドされていないジェネリック型を使うサンプルは C# 14 以降と書く |
| `Span<T>` / `ReadOnlySpan<T>` の暗黙変換 | 配列・Span 間の変換が自然になるが、所有権や寿命の説明を省かない |
| 単純ラムダパラメーターの修飾子 | `out result` など型省略時の修飾子を説明する |
| `field` でサポートされるプロパティ | 明示的なバッキングフィールド不要。ただし `field` という識別子との衝突に注意する |
| partial イベント / コンストラクター | source generator や partial 型の API で定義宣言と実装宣言を説明する |
| ユーザー定義複合代入演算子 | 演算子の副作用、戻り値、互換性を明記する |
| ファイルベースアプリ用プリプロセッサ | 単一ファイルサンプルで使う場合、通常プロジェクトとの差分を書く |

例:

```csharp
/// <summary>
/// メッセージを取得または設定します。
/// </summary>
/// <remarks>
/// このプロパティは C# 14 の <c>field</c> キーワードを使用して入力値を検証します。
/// 型内に <c>field</c> という名前のメンバーがある場合は、名前衝突に注意してください。
/// </remarks>
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

null 条件付き割り当ての説明例:

```csharp
/// <remarks>
/// C# 14 以降では、対象が <see langword="null" /> でない場合だけ代入するために
/// <c>customer?.Order = order</c> の形式を使用できます。
/// </remarks>
```

---

## 12. 新言語機能を使うときの互換性方針

新しい C# 仕様を使う場合は、ドキュメントに次を含める。

- 必要な .NET SDK と C# バージョン
- `LangVersion` を明示しているか
- CI / IDE / analyzer が対応しているか
- 旧バージョン利用者向けの代替コードが必要か
- ライブラリ公開時にターゲットフレームワークとパッケージ利用者へ影響があるか

README や API docs では、必要に応じて次のように書く。

```markdown
このサンプルは C# 14 / .NET 10 SDK 以降を前提にしています。
C# 13 / .NET 9 以前を対象にする場合は、`field` キーワードではなく明示的なバッキングフィールドを使用してください。
```

---

## 13. 日本語 API ドキュメントの表記

推奨表記:

| 用語 | 推奨 |
|---|---|
| null | `<see langword="null" />` |
| true / false | `<see langword="true" />` / `<see langword="false" />` |
| cancellation token | キャンセル トークン |
| async | 非同期 |
| exception | 例外 |
| property | プロパティ |
| parameter | パラメーター |
| collection | コレクション |
| overload | オーバーロード |
| source generator | ソース ジェネレーター |

日本語の助詞でコード要素を囲むと読みやすい。

```xml
<paramref name="name" /> が空の場合は例外を送出します。
```

---

## 14. ドキュメント品質チェック

レビュー時は次を確認する。

- [ ] 公開 API に `<summary>` がある
- [ ] メソッドに必要な `<param>`、`<typeparam>`、`<returns>` がある
- [ ] 直接送出する例外が `<exception>` に書かれている
- [ ] `null`、`true`、`false` は `<see langword>` で表現している
- [ ] パラメーター参照は `<paramref>` を使っている
- [ ] 型やメンバー参照は `<see cref>` を使っている
- [ ] サンプルコードは対象 C# バージョンでコンパイルできる
- [ ] C# 13 / C# 14 機能を使う場合、必要 SDK と互換性を書いている
- [ ] 日本語文字列、UTF-8、タイムゾーン、カルチャの前提が必要に応じて書かれている
- [ ] シークレットや個人情報をサンプルに含めていない
- [ ] DocFX や XML docs 生成で警告が出ない

---

## 15. よくある依頼への対応

### XML ドキュメントを追加する

1. 公開 API と利用者が迷う internal API を洗い出す
2. `<summary>`、`<param>`、`<returns>`、`<exception>` を追加する
3. 必要に応じて `<remarks>` と `<example>` を追加する
4. `cref` と `paramref` が解決できるか確認する
5. `dotnet build` で XML doc 警告を確認する

### README に C# サンプルを書く

1. 対象 .NET / C# バージョンを明記する
2. 最小構成で動くサンプルにする
3. C# 13 / 14 の新構文を使う場合は代替案も検討する
4. 日本語説明は短く、コードは英語識別子にする
5. 機密値はプレースホルダーにする

### 新しい言語仕様を説明する

1. Microsoft Learn の日本語ページと英語ページの差分を確認する
2. 対応 SDK、`LangVersion`、プレビュー機能かどうかを明記する
3. 旧構文との違いを短いコード例で示す
4. 互換性、破壊的変更、名前衝突、制約を説明する
5. 必要なら公式仕様や proposal へのリンクを添える

---

## 16. 公式参照

| リソース | URL |
|---|---|
| C# ドキュメント | https://learn.microsoft.com/ja-jp/dotnet/csharp/ |
| C# 13 の新機能 | https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-13 |
| C# 14 の新機能 | https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14 |
| C# 言語リファレンス | https://learn.microsoft.com/ja-jp/dotnet/csharp/language-reference/ |
| C# 言語仕様 | https://learn.microsoft.com/ja-jp/dotnet/csharp/language-reference/language-specification/ |
| Roslyn | https://github.com/dotnet/roslyn |
| C# language design | https://github.com/dotnet/csharplang |
