# Fluent UI Blazor Setup — 日本語環境向け

## NuGet packages

| Package | 用途 |
|---|---|
| `Microsoft.FluentUI.AspNetCore.Components` | core component library。必須 |
| `Microsoft.FluentUI.AspNetCore.Components.Icons` | strongly typed icons。推奨 |
| `Microsoft.FluentUI.AspNetCore.Components.Emojis` | emoji components。任意 |
| `Microsoft.FluentUI.AspNetCore.Components.DataGrid.EntityFrameworkAdapter` | DataGrid EF Core adapter。任意 |
| `Microsoft.FluentUI.AspNetCore.Components.DataGrid.ODataAdapter` | DataGrid OData adapter。任意 |

PowerShell:

```powershell
dotnet add package Microsoft.FluentUI.AspNetCore.Components
dotnet add package Microsoft.FluentUI.AspNetCore.Components.Icons
```

## Program.cs

```csharp
builder.Services.AddFluentUIComponents();
```

設定付き:

```csharp
builder.Services.AddFluentUIComponents(options =>
{
    options.UseTooltipServiceProvider = true;
    options.RequiredLabel = new MarkupString("<span aria-hidden=\"true\">*</span>");
    options.ServiceLifetime = ServiceLifetime.Scoped;
});
```

## ServiceLifetime

| Hosting model | ServiceLifetime |
|---|---|
| Blazor Server | `Scoped` |
| Blazor Web App Interactive | `Scoped` |
| Blazor WebAssembly standalone | `Singleton` |
| Blazor Hybrid / MAUI | `Singleton` |
| `Transient` | 使用不可。`NotSupportedException` |

## MainLayout.razor example

```razor
@inherits LayoutComponentBase
@using Icons = Microsoft.FluentUI.AspNetCore.Components.Icons

<FluentLayout>
    <FluentHeader Height="50">
        <FluentStack Orientation="Orientation.Horizontal"
                     HorizontalAlignment="HorizontalAlignment.SpaceBetween"
                     VerticalAlignment="VerticalAlignment.Center"
                     Width="100%">
            <span>業務アプリ</span>
            <FluentButton Appearance="Appearance.Stealth"
                          IconStart="@(Icons.Regular.Size20.Settings)">
                設定
            </FluentButton>
        </FluentStack>
    </FluentHeader>

    <FluentStack Orientation="Orientation.Horizontal"
                 HorizontalGap="0"
                 Style="height: 100%;">
        <FluentNavMenu Width="250"
                       Collapsible="true"
                       Title="メイン ナビゲーション">
            <FluentNavLink Href="/"
                           Icon="@(Icons.Regular.Size20.Home)"
                           Match="NavLinkMatch.All">
                ホーム
            </FluentNavLink>
            <FluentNavLink Href="/customers"
                           Icon="@(Icons.Regular.Size20.People)">
                顧客
            </FluentNavLink>
        </FluentNavMenu>

        <FluentBodyContent>
            <FluentStack Orientation="Orientation.Vertical" Style="padding: 1rem;">
                @Body
            </FluentStack>
        </FluentBodyContent>
    </FluentStack>
</FluentLayout>

<FluentToastProvider />
<FluentDialogProvider />
<FluentMessageBarProvider />
<FluentTooltipProvider />
<FluentKeyCodeProvider />

<FluentDesignTheme Mode="DesignThemeModes.System"
                   OfficeColor="OfficeColor.Teams"
                   StorageName="myapp-theme" />
```

## _Imports.razor

```razor
@using Microsoft.FluentUI.AspNetCore.Components
@using Icons = Microsoft.FluentUI.AspNetCore.Components.Icons
```

## Static web assets

コアライブラリ用の `<link>` / `<script>` は手動追加しない。

- CSS は static web assets で自動提供される。
- JS は Blazor JS initializer で自動読み込みされる。
- Component-specific JS は必要に応じて lazy-load される。
- `_content/Microsoft.FluentUI.AspNetCore.Components/` から配信される。

## Providers

| Provider | 必要な service |
|---|---|
| `FluentToastProvider` | `IToastService` |
| `FluentDialogProvider` | `IDialogService` |
| `FluentMessageBarProvider` | `IMessageService` |
| `FluentTooltipProvider` | `ITooltipService` |
| `FluentKeyCodeProvider` | `IKeyCodeService` |

provider がない場合、service call が失敗しても画面に何も出ないことがある。
