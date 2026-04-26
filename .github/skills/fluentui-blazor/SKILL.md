---
name: fluentui-blazor
description: "Microsoft.FluentUI.AspNetCore.Components を使う Blazor アプリ開発を日本語環境向けに支援する Skill。セットアップ、providers、icons、forms、dialogs、toasts、DataGrid、layout、navigation、theme、アクセシビリティ、日本語 UI、Blazor Server / WebAssembly / Web App を扱う。"
license: MIT
---

# Fluent UI Blazor — 日本語環境向け Skill

この Skill は、Blazor アプリケーションで **Microsoft Fluent UI Blazor** (`Microsoft.FluentUI.AspNetCore.Components`) を正しく使うために使う。日本語 UI、Windows 開発、Blazor Server / WebAssembly / Blazor Web App、アクセシビリティ、フォーム検証、DataGrid、テーマ切り替えを考慮する。

> **基本方針:** Fluent UI Blazor は Blazor の static web assets と JS initializer で CSS / JS を自動読み込みする。コアライブラリ用の `<script>` や `<link>` を手動追加しない。サービスベースの UI には root layout の provider が必須。

---

## 1. まず確認すること

| 確認項目 | 見る場所 / 観点 |
|---|---|
| Hosting model | Blazor Server、Blazor WebAssembly、Blazor Web App、Blazor Hybrid |
| .NET SDK | `global.json`, `dotnet --info` |
| Fluent UI package | `Microsoft.FluentUI.AspNetCore.Components` |
| Icon package | `Microsoft.FluentUI.AspNetCore.Components.Icons` |
| DI 登録 | `Program.cs` の `AddFluentUIComponents()` |
| Provider 配置 | `MainLayout.razor` など root layout |
| 日本語 UI | ラベル、検証メッセージ、日付・数値表示、フォント |
| アクセシビリティ | aria-label、キーボード操作、contrast、focus |
| DataGrid | `Items` / `ItemsProvider`、pagination、virtualization |
| Theme | `FluentDesignTheme`、dark/light/system、localStorage |

---

## 2. 必須ルール

### コア CSS / JS を手動追加しない

`Microsoft.FluentUI.AspNetCore.Components` は static web assets と JS initializer により必要な CSS / JS を自動読み込みする。

避ける:

```html
<!-- 不要。追加しない。 -->
<link rel="stylesheet" href="_content/Microsoft.FluentUI.AspNetCore.Components/..." />
<script src="_content/Microsoft.FluentUI.AspNetCore.Components/..."></script>
```

---

### `AddFluentUIComponents()` を登録する

```csharp
builder.Services.AddFluentUIComponents();
```

設定付き:

```csharp
builder.Services.AddFluentUIComponents(options =>
{
    options.UseTooltipServiceProvider = true;
    options.ServiceLifetime = ServiceLifetime.Scoped;
});
```

ServiceLifetime の目安:

| Hosting model | ServiceLifetime |
|---|---|
| Blazor Server | `Scoped` |
| Blazor Web App Interactive | `Scoped` |
| Blazor WebAssembly standalone | `Singleton` |
| Blazor Hybrid | `Singleton` |
| `Transient` | 使用しない。`NotSupportedException` |

---

### Provider は root layout に置く

サービスベースのコンポーネントには provider が必要。ない場合、service call が失敗しても UI が出ないことがある。

```razor
<FluentToastProvider />
<FluentDialogProvider />
<FluentMessageBarProvider />
<FluentTooltipProvider />
<FluentKeyCodeProvider />
```

通常は `MainLayout.razor` の `FluentLayout` 後、または app root に近い場所へ配置する。

---

### `_Imports.razor`

```razor
@using Microsoft.FluentUI.AspNetCore.Components
@using Icons = Microsoft.FluentUI.AspNetCore.Components.Icons
```

---

## 3. References

詳細が必要な場合は次を参照する。

| Reference | 用途 |
|---|---|
| `references/SETUP.md` | NuGet、DI、providers、static web assets |
| `references/LAYOUT-AND-NAVIGATION.md` | layout、nav menu、breadcrumb、tabs |
| `references/DATAGRID.md` | FluentDataGrid、columns、ItemsProvider、pagination、virtualization |
| `references/THEMING.md` | FluentDesignTheme、design tokens、dark/light/system |

---

## 4. Icons

Icons は別 package が必要。

```powershell
dotnet add package Microsoft.FluentUI.AspNetCore.Components.Icons
```

使い方:

```razor
@using Icons = Microsoft.FluentUI.AspNetCore.Components.Icons

<FluentIcon Value="@(Icons.Regular.Size24.Save)" />
<FluentIcon Value="@(Icons.Filled.Size20.Delete)" Color="@Color.Error" />
<FluentIcon Value='@(Icon.FromImageUrl("/images/logo.png"))' />
```

パターン:

```text
Icons.[Variant].[Size].[Name]
```

- Variant: `Regular`, `Filled`
- Size: `Size12`, `Size16`, `Size20`, `Size24`, `Size28`, `Size32`, `Size48`

文字列ベースの icon name は使わない。Fluent UI Blazor の icon は strongly typed class を使う。
独自画像を icon として使う場合は、文字列名ではなく `Icon.FromImageUrl(...)` を使う。

---

## 5. 日本語 UI とアクセシビリティ

日本語環境では次を意識する。

- `Label`、`Placeholder`、button text、dialog title、toast message は日本語にする。
- 日付・数値・通貨は `ja-JP` culture を明示またはアプリ設定に従う。
- 日本語は英語より長くなりやすいため、固定幅 UI で折り返しや overflow を確認する。
- `aria-label`、`Title`、`Tooltip` は日本語利用者に分かる表現にする。
- icon-only button には必ず label / aria 相当の説明を付ける。
- keyboard navigation と focus outline を壊さない。
- color だけで状態を表現しない。`FluentBadge`、text、icon を併用する。
- validation message は `FluentValidationMessage` / `FluentValidationSummary` で Fluent styling に揃える。

---

## 6. List component binding

`FluentSelect<TOption>`、`FluentCombobox<TOption>`、`FluentListbox<TOption>`、`FluentAutocomplete<TOption>` は標準 `<InputSelect>` と同じ書き方ではない。

使うもの:

- `Items`
- `OptionText`
- `OptionValue`
- `SelectedOption` / `SelectedOptionChanged`
- `SelectedOptions` / `SelectedOptionsChanged`

```razor
<FluentSelect Items="@prefectures"
              OptionText="@(p => p.Name)"
              OptionValue="@(p => p.Code)"
              @bind-SelectedOption="@selectedPrefecture"
              Label="都道府県" />
```

避ける:

```razor
@* 誤り: InputSelect の pattern を使わない *@
<FluentSelect @bind-Value="@selectedValue">
    <option value="13">東京都</option>
</FluentSelect>
```

---

## 7. FluentAutocomplete

`FluentAutocomplete` の注意点:

- 検索入力 text は `ValueText` を使う。`Value` は obsolete。
- `OnOptionsSearch` で候補を filter する。
- 既定は `Multiple="true"`。

```razor
<FluentAutocomplete TOption="Person"
                    OnOptionsSearch="@OnSearch"
                    OptionText="@(p => p.FullName)"
                    @bind-SelectedOptions="@selectedPeople"
                    Label="担当者を検索" />

@code {
    private IReadOnlyList<Person> allPeople = [];
    private IEnumerable<Person> selectedPeople = [];

    private void OnSearch(OptionsSearchEventArgs<Person> args)
    {
        args.Items = allPeople.Where(p =>
            p.FullName.Contains(args.Text, StringComparison.CurrentCultureIgnoreCase));
    }
}
```

日本語検索では、ひらがな・カタカナ、全角・半角、濁点などを扱う要件があるか確認する。必要なら検索用の正規化を別 service に分離する。

---

## 8. Dialog service pattern

`<FluentDialog>` の表示状態を手動 toggle するのではなく、`IDialogService` を使う。

content component:

```csharp
public partial class EditCustomerDialog : IDialogContentComponent<Customer>
{
    [Parameter]
    public Customer Content { get; set; } = default!;

    [CascadingParameter]
    public FluentDialog Dialog { get; set; } = default!;

    private async Task SaveAsync()
    {
        await Dialog.CloseAsync(Content);
    }

    private async Task CancelAsync()
    {
        await Dialog.CancelAsync();
    }
}
```

呼び出し:

```csharp
[Inject]
private IDialogService DialogService { get; set; } = default!;

private async Task ShowEditDialogAsync(Customer customer)
{
    var dialog = await DialogService.ShowDialogAsync<EditCustomerDialog, Customer>(
        customer,
        new DialogParameters
        {
            Title = "顧客情報の編集",
            PrimaryAction = "保存",
            SecondaryAction = "キャンセル",
            Width = "560px",
            PreventDismissOnOverlayClick = true,
        });

    var result = await dialog.Result;
    if (!result.Cancelled && result.Data is Customer updatedCustomer)
    {
        customer = updatedCustomer;
    }
}
```

簡易 dialog:

```csharp
await DialogService.ShowConfirmationAsync("削除しますか？", "削除", "キャンセル");
await DialogService.ShowSuccessAsync("保存しました。");
await DialogService.ShowErrorAsync("保存に失敗しました。");
```

---

## 9. Toast notifications

```csharp
[Inject]
private IToastService ToastService { get; set; } = default!;

ToastService.ShowSuccess("保存しました。");
ToastService.ShowError("保存に失敗しました。");
ToastService.ShowWarning("入力内容を確認してください。");
ToastService.ShowInfo("更新があります。");
```

`FluentToastProvider` の主な parameters:

- `Position`
- `Timeout`
- `MaxToastCount`

日本語メッセージは短くし、詳細は画面内の message bar や validation に出す。

---

## 10. Forms

通常フォームは標準 `EditForm` と Fluent form components を使う。`FluentEditForm` は主に `FluentWizard` step ごとの validation で使う。

```razor
<EditForm Model="@model" OnValidSubmit="HandleSubmitAsync">
    <DataAnnotationsValidator />

    <FluentTextField @bind-Value="@model.Name"
                     Label="氏名"
                     Required />

    <FluentSelect Items="@categories"
                  OptionText="@(c => c.Label)"
                  @bind-SelectedOption="@model.Category"
                  Label="カテゴリ" />

    <FluentValidationSummary />

    <FluentButton Type="ButtonType.Submit" Appearance="Appearance.Accent">
        保存
    </FluentButton>
</EditForm>
```

Fluent styling を維持するため、標準 `ValidationMessage` / `ValidationSummary` ではなく `FluentValidationMessage` / `FluentValidationSummary` を優先する。

---

## 11. DataGrid

`FluentDataGrid<TGridItem>` は strongly typed な table component。columns は property ではなく child component として定義する。

```razor
<FluentDataGrid Items="@customers.AsQueryable()" TGridItem="Customer">
    <PropertyColumn Property="@(c => c.Name)" Title="氏名" Sortable="true" />
    <PropertyColumn Property="@(c => c.Email)" Title="メールアドレス" />
    <PropertyColumn Property="@(c => c.CreatedAt)" Title="登録日" Format="yyyy/MM/dd" />
    <TemplateColumn Title="操作">
        <FluentButton OnClick="@(() => EditAsync(context))">編集</FluentButton>
    </TemplateColumn>
</FluentDataGrid>
```

大量データでは `ItemsProvider`、pagination、virtualization を検討する。詳細は `references/DATAGRID.md` を参照する。

---

## 12. Theme

`FluentDesignTheme` を root に置く。

```razor
<FluentDesignTheme Mode="DesignThemeModes.System"
                   OfficeColor="OfficeColor.Teams"
                   StorageName="myapp-theme" />
```

Design token は JS interop に依存するため、`OnInitialized` ではなく `OnAfterRenderAsync` で操作する。

```csharp
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        // design token 操作はここで行う。
    }
}
```

---

## 13. トラブルシュート

| 症状 | 確認すること |
|---|---|
| Toast / Dialog が表示されない | root layout に provider があるか |
| Tooltip が出ない | `UseTooltipServiceProvider` と `FluentTooltipProvider` |
| Icon が見つからない | Icons package、`@using Icons = ...`、strongly typed icon |
| Select が bind できない | `@bind-Value` ではなく `SelectedOption` / `Items` pattern |
| Autocomplete が動かない | `OnOptionsSearch`、`ValueText`、`Multiple` |
| Theme が反映されない | `FluentDesignTheme` の配置、JS interop、pre-rendering |
| DataGrid の列が出ない | `PropertyColumn` / `TemplateColumn` を child component にしているか |
| Blazor WASM で service lifetime 問題 | `ServiceLifetime.Singleton` を検討 |

---

## 14. レビュー時チェックリスト

- [ ] `AddFluentUIComponents()` が登録されている
- [ ] core CSS / JS を手動追加していない
- [ ] root layout に必要な providers がある
- [ ] Icons package と strongly typed icon を使っている
- [ ] list components が `Items` / `OptionText` / `SelectedOption` pattern
- [ ] Dialog は `IDialogService` pattern
- [ ] Toast は `IToastService` と `FluentToastProvider`
- [ ] 通常フォームは `EditForm` + Fluent form components
- [ ] 日本語ラベル、validation、aria / tooltip が適切
- [ ] DataGrid は child columns、必要に応じて pagination / virtualization
- [ ] theme 操作は render 後に行っている
- [ ] Blazor hosting model に合った service lifetime

---

## 15. 公式参照

| リソース | URL |
|---|---|
| Fluent UI Blazor GitHub | https://github.com/microsoft/fluentui-blazor |
| Fluent UI Blazor demo | https://www.fluentui-blazor.net/ |
| Microsoft Fluent UI | https://developer.microsoft.com/fluentui |
| Blazor documentation | https://learn.microsoft.com/ja-jp/aspnet/core/blazor/ |
