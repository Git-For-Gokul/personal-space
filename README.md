let
    // 1) Load your table (rename "Table1" to your actual table name)
    Source = Excel.CurrentWorkbook(){[Name="Table1"]}[Content],
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    // 2) Normalize column names (trim + lower-case)
    Normalized = Table.TransformColumnNames(Promoted, each Text.Lower(Text.Trim(_))),

    // 3) Required columns
    Required = {"legal enity","legal entity","product","agreement type","agreement version","counterparty id"},
    Cols = Table.ColumnNames(Normalized),

    // 4) Fix typo in "legal enity" → "legal entity"
    Fixed = if List.Contains(Cols,"legal enity") and not List.Contains(Cols,"legal entity") 
            then Table.RenameColumns(Normalized, {{"legal enity","legal entity"}})
            else Normalized,

    // 5) Unpivot the two agreement columns
    Unpivoted = Table.Unpivot(Fixed, {"agreement type","agreement version"}, "attribute", "value"),

    // 6) Build composite column names: product + attribute
    WithHeader = Table.AddColumn(Unpivoted, "prodattr", each [product] & " " & [attribute]),

    // 7) Keep only row key (counterparty + legal entity) and pivot fields
    Slim = Table.RemoveColumns(WithHeader, {"product","attribute"}),

    // 8) Pivot wide by prodattr, grouped by counterparty id + legal entity
    Wide = Table.Pivot(Slim, List.Distinct(Slim[prodattr]), "prodattr", "value", List.First),

    // 9) Rename for pretty headers
    Renamed = Table.RenameColumns(Wide, {
        {"REPO agreement type","REPO Agreement Type"},
        {"REPO agreement version","REPO Agreement Version"},
        {"SLEB agreement type","SLEB Agreement Type"},
        {"SLEB agreement version","SLEB Agreement Version"}
    }, MissingField.Ignore)
in
    Renamed
