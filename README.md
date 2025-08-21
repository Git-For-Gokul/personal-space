let
    // 1. Load data
    Source = Excel.CurrentWorkbook(){[Name="Table1"]}[Content],

    // 2. Set types
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

    // 3. Group by counterparty + legal entity + product
    Grouped = Table.Group(
        #"Changed Type",
        {"counterparty id","Legal Entity","Product"},
        {
            {"Agreement Type", each Text.Combine(List.Distinct([Agreement Type]), ", "), type text},
            {"Agreement Version", each Text.Combine(List.Distinct([Agreement Version]), ", "), type text}
        }
    ),

    // 4. Pivot on product (REPO / SLEB)
    Pivoted = Table.Pivot(
        Grouped,
        List.Distinct(Grouped[Product]),
        "Product",
        {"Agreement Type","Agreement Version"}
    ),

    // 5. Flatten column names
    Renamed = Table.RenameColumns(
        Pivoted,
        {
            {"REPO.Agreement Type", "REPO Agreement Type"},
            {"REPO.Agreement Version", "REPO Agreement Version"},
            {"SLEB.Agreement Type", "SLEB Agreement Type"},
            {"SLEB.Agreement Version", "SLEB Agreement Version"}
        },
        MissingField.Ignore
    )
in
    Renamed

