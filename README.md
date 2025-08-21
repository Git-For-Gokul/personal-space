let
    // 1) Load table (replace "Table1" with your actual table name/range)
    Source = Excel.CurrentWorkbook(){[Name="Table1"]}[Content],

    // 2) Promote headers
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    // 3) Unpivot Agreement Type + Agreement Version
    Unpivoted = Table.Unpivot(Promoted, {"Agreement Type","Agreement Version"}, "Attribute", "Value"),

    // 4) Build composite column names: Product + Attribute
    WithHeader = Table.AddColumn(Unpivoted, "ProdAttr", each [Product] & " " & [Attribute]),

    // 5) Keep only row keys (Counterparty ID + Legal Entity) + pivot fields
    Slim = Table.RemoveColumns(WithHeader, {"Product","Attribute"}),

    // 6) Pivot wide (one row per Counterparty ID + Legal Entity)
    Wide = Table.Pivot(Slim, List.Distinct(Slim[ProdAttr]), "ProdAttr", "Value", List.First),

    // 7) Rename for pretty headers
    Renamed = Table.RenameColumns(Wide, {
        {"REPO Agreement Type","REPO Agreement Type"},
        {"REPO Agreement Version","REPO Agreement Version"},
        {"SLEB Agreement Type","SLEB Agreement Type"},
        {"SLEB Agreement Version","SLEB Agreement Version"}
    }, MissingField.Ignore)
in
    Renamed
