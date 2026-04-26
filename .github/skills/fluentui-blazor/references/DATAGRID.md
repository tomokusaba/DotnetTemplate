# FluentDataGrid — 日本語環境向け

`FluentDataGrid<TGridItem>` は strongly typed な table component。日本語の一覧画面では、列幅、折り返し、日付・数値 format、アクセシビリティを確認する。

## Basic usage

```razor
<FluentDataGrid Items="@customers.AsQueryable()" TGridItem="Customer">
    <PropertyColumn Property="@(c => c.Name)" Title="氏名" Sortable="true" />
    <PropertyColumn Property="@(c => c.Email)" Title="メールアドレス" />
    <PropertyColumn Property="@(c => c.CreatedAt)" Title="登録日" Format="yyyy/MM/dd" />
    <TemplateColumn Title="操作">
        <FluentButton OnClick="@(() => EditAsync(context))">
            編集
        </FluentButton>
    </TemplateColumn>
</FluentDataGrid>
```

重要: columns は `FluentDataGrid` の property ではなく child component。`PropertyColumn`、`TemplateColumn`、`SelectColumn` を grid 内に置く。

## Column types

### PropertyColumn

```razor
<PropertyColumn Property="@(p => p.Name)" Title="氏名" Sortable="true" />
<PropertyColumn Property="@(p => p.Price)" Title="単価" Format="C0" />
<PropertyColumn Property="@(p => p.Category)"
                Title="カテゴリ"
                Comparer="@StringComparer.CurrentCultureIgnoreCase" />
```

主な parameters:

- `Property`
- `Format`
- `Title`
- `Sortable`
- `SortBy`
- `Comparer`
- `IsDefaultSortColumn`
- `InitialSortDirection`
- `Class`
- `Tooltip`

### TemplateColumn

```razor
<TemplateColumn Title="状態" SortBy="@statusSort">
    <FluentBadge Appearance="Appearance.Accent"
                 BackgroundColor="@(context.IsActive ? "green" : "gray")">
        @(context.IsActive ? "有効" : "無効")
    </FluentBadge>
</TemplateColumn>
```

`context` は `TGridItem`。

### SelectColumn

```razor
<SelectColumn TGridItem="Customer"
              SelectMode="DataGridSelectMode.Multiple"
              @bind-SelectedItems="@selectedCustomers" />
```

## Data sources

`Items` と `ItemsProvider` は目的に応じて使い分ける。

### In-memory / IQueryable

```razor
<FluentDataGrid Items="@customers.AsQueryable()" TGridItem="Customer">
    ...
</FluentDataGrid>
```

小規模データや client-side filter に向く。

### Server-side / ItemsProvider

大量データ、server-side paging / sorting / filtering では `ItemsProvider` を使う。

```razor
<FluentDataGrid ItemsProvider="@customersProvider" TGridItem="Customer">
    ...
</FluentDataGrid>

@code {
    private GridItemsProvider<Customer> customersProvider = async request =>
    {
        var result = await CustomerService.GetCustomersAsync(
            request.StartIndex,
            request.Count ?? 50,
            request.GetSortByProperties().FirstOrDefault());

        return GridItemsProviderResult.From(result.Items, result.TotalCount);
    };
}
```

### EF Core adapter

```csharp
builder.Services.AddDataGridEntityFrameworkAdapter();
```

```razor
<FluentDataGrid Items="@dbContext.Customers" TGridItem="Customer">
    ...
</FluentDataGrid>
```

## Pagination

```razor
<FluentDataGrid Items="@customers.AsQueryable()"
                Pagination="@pagination"
                TGridItem="Customer">
    ...
</FluentDataGrid>

<FluentPaginator State="@pagination" />

@code {
    private PaginationState pagination = new() { ItemsPerPage = 20 };
}
```

## Virtualization

大量データでは virtualization を検討する。

```razor
<FluentDataGrid Items="@customers.AsQueryable()"
                Virtualize="true"
                ItemSize="46"
                TGridItem="Customer">
    ...
</FluentDataGrid>
```

`ItemSize` は行高さの推定値。実際の行高さとずれるとスクロール位置が不安定になる。

## Sorting

```razor
<PropertyColumn Property="@(c => c.Name)"
                Title="氏名"
                Sortable="true"
                IsDefaultSortColumn="true"
                InitialSortDirection="SortDirection.Ascending" />
```

custom sort:

```razor
<TemplateColumn Title="氏名"
                SortBy="@(GridSort<Customer>.ByAscending(c => c.LastName).ThenAscending(c => c.FirstName))">
    @context.LastName @context.FirstName
</TemplateColumn>
```

日本語文字列の並び順に要件がある場合は、culture、kana、全角・半角、濁点の扱いを確認する。

## 日本語一覧画面の注意

- `Title` は日本語にする。
- 日付は `yyyy/MM/dd` など明確な形式を使う。
- 通貨や数値は `ja-JP` culture の要件を確認する。
- 長い日本語文は列幅、折り返し、省略、tooltip を設計する。
- 操作列には icon だけでなく label / aria 相当の説明を付ける。
- server-side data access では cancellation、paging、sort、filter を下位へ伝播する。
