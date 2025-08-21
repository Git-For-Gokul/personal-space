let
    // 1) Load source
    Source = Excel.CurrentWorkbook(){[Name="Table1"]}[Content],

    // 2) Ensure types
    #"Changed Type" = Table.TransformColumnTypes(
        Source,
        {
            {"Legal Entity", type text},
            {"Product", type text},
            {"Agreement Type", type text},
            {"Agreement Version", type text},
            {"counterparty id", Int64.Type}
        }
    ),

    // 3) Normalize Product values (trim + uppercase) to avoid product-name mismatches
    #"Normalize Product" = Table.TransformColumns(#"Changed Type", {{"Product", each Text.Upper(Text.Trim(_)), type text}}),

    // 4) Group to one row per (counterparty id, Legal Entity, Product)
    Grouped = Table.Group(
        #"Normalize Product",
        {"counterparty id","Legal Entity","Product"},
        {
            {"Agreement Type", each List.First([Agreement Type]), type text},
            {"Agreement Version", each List.First([Agreement Version]), type text}
        }
    ),

    // 5) Get product list (e.g. {"REPO","SLEB"})
    ProductList = List.Distinct(Grouped[Product]),

    // 6) Pivot Agreement Type (creates columns like REPO, SLEB with types)
    Pivoted_Type = Table.Pivot(
        Grouped,
        ProductList,
        "Product",
        "Agreement Type",
        List.First
    ),

    // 7) Pivot Agreement Version (creates columns like REPO, SLEB with versions)
    Pivoted_Version = Table.Pivot(
        Grouped,
        ProductList,
        "Product",
        "Agreement Version",
        List.First
    ),

    // 8) Rename pivoted columns to descriptive names:
    //    e.g. REPO -> "REPO Agreement Type", REPO (in version table) -> "REPO Version"
    RenameTypePairs = List.Transform(ProductList, each {_, _ & " Agreement Type"}),
    RenameVersionPairs = List.Transform(ProductList, each {_, _ & " Version"}),

    Pivoted_Type_Renamed = Table.RenameColumns(Pivoted_Type, RenameTypePairs, MissingField.Ignore),
    Pivoted_Version_Renamed = Table.RenameColumns(Pivoted_Version, RenameVersionPairs, MissingField.Ignore),

    // 9) Merge the two pivoted tables on both keys (use FullOuter to keep all rows)
    Merged = Table.Join(
        Pivoted_Type_Renamed,
        {"counterparty id","Legal Entity"},
        Pivoted_Version_Renamed,
        {"counterparty id","Legal Entity"},
        JoinKind.FullOuter
    ),

    // 10) Reorder columns (only include those that exist)
    DesiredOrder = {"counterparty id","Legal Entity"} 
                   & List.Transform(ProductList, each _ & " Agreement Type")
                   & List.Transform(ProductList, each _ & " Version"),

    FinalOrder = List.Select(DesiredOrder, each List.Contains(Table.ColumnNames(Merged), _)),

    Final = Table.ReorderColumns(Merged, FinalOrder)
in
    Final
