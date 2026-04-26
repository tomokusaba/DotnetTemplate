# Fluent UI Blazor Layout and Navigation — 日本語環境向け

## FluentLayout

root layout container として使う。

```razor
<FluentLayout Orientation="Orientation.Vertical">
    <FluentHeader>...</FluentHeader>
    <FluentBodyContent>...</FluentBodyContent>
    <FluentFooter>...</FluentFooter>
</FluentLayout>
```

## FluentHeader / FluentFooter

```razor
<FluentHeader Height="50">
    <FluentStack Orientation="Orientation.Horizontal"
                 HorizontalAlignment="HorizontalAlignment.SpaceBetween"
                 VerticalAlignment="VerticalAlignment.Center">
        <span>業務アプリ</span>
        <FluentButton>設定</FluentButton>
    </FluentStack>
</FluentHeader>
```

## FluentStack

flexbox layout。日本語ラベルは長くなりやすいため、`Wrap` や幅を確認する。

```razor
<FluentStack Orientation="Orientation.Horizontal"
             HorizontalGap="10"
             VerticalGap="10"
             Wrap="true"
             Width="100%">
    <FluentButton>検索</FluentButton>
    <FluentButton>条件をクリア</FluentButton>
</FluentStack>
```

## FluentGrid / FluentGridItem

12 column responsive grid。

```razor
<FluentGrid Spacing="3" Justify="JustifyContent.Center" AdaptiveRendering="true">
    <FluentGridItem xs="12" sm="6" md="4" lg="3">
        <FluentCard>カード 1</FluentCard>
    </FluentGridItem>
    <FluentGridItem xs="12" sm="6" md="4" lg="3">
        <FluentCard>カード 2</FluentCard>
    </FluentGridItem>
</FluentGrid>
```

## FluentNavMenu

collapsible navigation menu。`Title` は accessibility 用の説明として日本語で設定する。

```razor
<FluentNavMenu Width="250"
               Collapsible="true"
               @bind-Expanded="@menuExpanded"
               Title="メイン ナビゲーション"
               CollapsedChildNavigation="true">
    <FluentNavLink Href="/"
                   Icon="@(Icons.Regular.Size20.Home)"
                   Match="NavLinkMatch.All">
        ホーム
    </FluentNavLink>
    <FluentNavGroup Title="管理"
                    Icon="@(Icons.Regular.Size20.Shield)"
                    @bind-Expanded="@adminExpanded">
        <FluentNavLink Href="/admin/users">ユーザー</FluentNavLink>
        <FluentNavLink Href="/admin/roles">ロール</FluentNavLink>
    </FluentNavGroup>
</FluentNavMenu>
```

主な parameters:

- `Width`
- `Collapsible`
- `Expanded` / `ExpandedChanged`
- `CollapsedChildNavigation`
- `CustomToggle`
- `Title`

## FluentNavLink

```razor
<FluentNavLink Href="/reports"
               Icon="@(Icons.Regular.Size20.Document)"
               Match="NavLinkMatch.Prefix"
               Tooltip="レポート一覧を開きます">
    レポート
</FluentNavLink>
```

icon-only や collapse 時に tooltip / label が伝わるか確認する。

## FluentBreadcrumb

```razor
<FluentBreadcrumb>
    <FluentBreadcrumbItem Href="/">ホーム</FluentBreadcrumbItem>
    <FluentBreadcrumbItem Href="/customers">顧客</FluentBreadcrumbItem>
    <FluentBreadcrumbItem>詳細</FluentBreadcrumbItem>
</FluentBreadcrumb>
```

## FluentTabs

```razor
<FluentTabs @bind-ActiveTabId="@activeTab">
    <FluentTab Id="summary" Label="概要">
        概要コンテンツ
    </FluentTab>
    <FluentTab Id="history" Label="履歴">
        履歴コンテンツ
    </FluentTab>
</FluentTabs>
```

## 日本語 UI の注意

- nav label が長い場合は折り返し・省略・tooltip を設計する。
- keyboard navigation と focus visibility を維持する。
- `Title`、`Tooltip`、`aria` 相当の説明は日本語利用者に分かる表現にする。
- mobile / narrow layout で nav collapse の動作を確認する。
