let
    // 1. Load source data
    Source = Excel.CurrentWorkbook(){[Name="Table1"]}[Content],

    // 2. Ensure column types
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

    // 3. Pivot Agreement Type
    PivotType = Table.Pivot(
        #"Changed Type",
        List.Distinct(#"Changed Type"[Product]),
        "Product",
        "Agreement Type",
        each List.First(_)
    ),

    // 4. Pivot Agreement Version
    PivotVersion = Table.Pivot(
        #"Changed Type",
        List.Distinct(#"Changed Type"[Product]),
        "Product",
        "Agreement Version",
        each List.First(_)
    ),

    // 5. Merge Type + Version on counterparty id + legal entity
    Merged = Table.NestedJoin(
        PivotType,
        {"counterparty id","Legal Entity"},
        PivotVersion,
        {"counterparty id","Legal Entity"},
        "Ver",
        JoinKind.FullOuter   // keep all rows, fill missing with null
    ),

    // 6. Expand version columns
    Expanded = Table.ExpandTableColumn(
        Merged,
        "Ver",
        {"REPO","SLEB"},
        {"REPO Version","SLEB Version"}
    ),

    // 7. Rename type columns
    Renamed = Table.RenameColumns(
        Expanded,
        {
            {"REPO", "REPO Agreement Type"},
            {"SLEB", "SLEB Agreement Type"}
        }
    ),

    // 8. Reorder final columns
    Final = Table.ReorderColumns(
        Renamed,
        {
            "counterparty id",
            "Legal Entity",
            "REPO Agreement Type",
            "REPO Version",
            "SLEB Agreement Type",
            "SLEB Version"
        }
    )
in
    Final
