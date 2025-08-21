let
    // 1. Source Data
    Source = Excel.CurrentWorkbook(){[Name="Table1"]}[Content],

    #"Changed Type" = Table.TransformColumnTypes(Source,
        {{"Legal Entity", type text}, 
         {"Product", type text}, 
         {"Agreement Type", type text}, 
         {"Agreement Version", type text}, 
         {"counterparty id", Int64.Type}}),

    // 2. Create combined column for pivot (e.g., "REPO Agreement Type", "REPO Agreement Version")
    WithCombinedColumns = Table.TransformColumns(#"Changed Type", {
        {"Product", each _, type text}
    }),
    AddTypeCol = Table.AddColumn(WithCombinedColumns, "PivotKey_Type", each [Product] & " Agreement Type"),
    AddVersionCol = Table.AddColumn(AddTypeCol, "PivotKey_Version", each [Product] & " Agreement Version"),

    // 3. Unpivot into a tall table with the new pivot keys
    Unpivoted = Table.UnpivotOtherColumns(AddVersionCol, {"counterparty id","Legal Entity","Product"}, "PivotKey", "Value"),

    // 4. Pivot so each counterparty+LE becomes one row with all columns
    Pivoted = Table.Pivot(
        Unpivoted, 
        List.Distinct(Unpivoted[PivotKey]), 
        "PivotKey", 
        "Value", 
        each List.First(_)
    ),

    // 5. Reorder
    Final = Table.ReorderColumns(Pivoted, {"counterparty id","Legal Entity",
        "REPO Agreement Type","REPO Agreement Version",
        "SLEB Agreement Type","SLEB Agreement Version"})
in
    Final
