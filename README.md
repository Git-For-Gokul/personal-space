let
    // 1. Source Data
    Source = Excel.CurrentWorkbook(){[Name="Table1"]}[Content],

    #"Changed Type" = Table.TransformColumnTypes(Source,
        {{"Legal Entity", type text}, 
         {"Product", type text}, 
         {"Agreement Type", type text}, 
         {"Agreement Version", type text}, 
         {"counterparty id", Int64.Type}}),

    // 2. Unpivot Agreement Type + Version into rows
    Unpivoted = Table.Unpivot(#"Changed Type", {"Agreement Type","Agreement Version"}, "Attribute", "Value"),

    // 3. Build new column name like "REPO Agreement Type" or "SLEB Agreement Version"
    AddedCustom = Table.AddColumn(Unpivoted, "PivotKey", each [Product] & " " & [Attribute]),

    // 4. Remove now-redundant columns
    RemovedExtra = Table.RemoveColumns(AddedCustom, {"Product","Attribute"}),

    // 5. Pivot so each counterparty+LE has its own row with the right set of columns
    Pivoted = Table.Pivot(
        RemovedExtra,
        List.Distinct(RemovedExtra[PivotKey]),
        "PivotKey",
        "Value",
        each List.First(_)
    ),

    // 6. Reorder columns (will only work if they exist — otherwise they’ll just stay at the end)
    Final = Table.ReorderColumns(Pivoted, {
        "counterparty id","Legal Entity",
        "REPO Agreement Type","REPO Agreement Version",
        "SLEB Agreement Type","SLEB Agreement Version"
    }, MissingField.Ignore)
in
    Final
