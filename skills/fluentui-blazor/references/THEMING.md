# Fluent UI Blazor Theming — 日本語環境向け

## FluentDesignTheme

アプリ全体の theme には `FluentDesignTheme` を root に置く。

```razor
<FluentDesignTheme Mode="DesignThemeModes.System"
                   OfficeColor="OfficeColor.Teams"
                   StorageName="myapp-theme" />
```

## Parameters

| Parameter | Type | 既定値 | 説明 |
|---|---|---|---|
| `Mode` | `DesignThemeModes` | `System` | `Light`、`Dark`、`System` |
| `CustomColor` | `string?` | null | accent color。例: `#0078D4` |
| `OfficeColor` | `OfficeColor?` | null | `Teams`、`Word`、`Excel` など |
| `NeutralBaseColor` | `string?` | null | neutral palette base |
| `StorageName` | `string?` | null | localStorage に保存する key |
| `Direction` | `LocalizationDirection?` | null | `Ltr` または `Rtl` |
| `OnLuminanceChanged` | `EventCallback<LuminanceChangedEventArgs>` | | light / dark 変更時 |
| `OnLoaded` | `EventCallback<LoadedEventArgs>` | | storage から theme 読み込み時 |

日本語は通常 `LocalizationDirection.Ltr`。

## Two-way binding

```razor
<FluentDesignTheme @bind-Mode="@themeMode"
                   @bind-OfficeColor="@officeColor"
                   @bind-CustomColor="@customColor"
                   StorageName="myapp-theme" />

<FluentSelect Items="@(Enum.GetValues<DesignThemeModes>())"
              @bind-SelectedOption="@themeMode"
              OptionText="@(m => m.ToString())"
              Label="テーマ" />

@code {
    private DesignThemeModes themeMode = DesignThemeModes.System;
    private OfficeColor? officeColor = OfficeColor.Teams;
    private string? customColor;
}
```

日本語 UI では enum 表示をそのまま使うより、表示名を日本語に mapping する。

## JS interop dependency

`FluentDesignTheme` と design tokens は JS interop に依存する。server-side pre-rendering 中には動かない。

避ける:

```csharp
protected override void OnInitialized()
{
    // design token 操作をここで行わない。
}
```

推奨:

```csharp
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        // design token 操作はここで行う。
    }
}
```

## FluentDesignSystemProvider

subtree に theme を限定したい場合に使う。

```razor
<FluentDesignSystemProvider AccentBaseColor="#0078D4"
                            NeutralBaseColor="#808080"
                            BaseLayerLuminance="0.95">
    <FluentButton Appearance="Appearance.Accent">
        テーマ適用ボタン
    </FluentButton>
</FluentDesignSystemProvider>
```

## OfficeColor presets

`Teams`, `Word`, `Excel`, `PowerPoint`, `Outlook`, `OneNote`, `Loop`, `Planner`, `SharePoint`, `Stream`, `Sway`, `Viva`, `VivaEngage`, `VivaInsights`, `VivaLearning`, `VivaTopics`

## 日本語環境での theme 注意

- dark mode で日本語フォントの可読性と contrast を確認する。
- color だけで状態を表さない。
- OfficeColor や CustomColor は企業 brand guideline と整合させる。
- `StorageName` はアプリごとに一意にする。
- theme selector の表示名は日本語化する。
- high contrast / forced colors の動作を確認する。
