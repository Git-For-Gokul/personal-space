let
    // 1) Load table (rename "Table1" if needed)
    Source   = Excel.CurrentWorkbook(){[Name="Table1"]}[Content],
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    // 2) Normalizer (lowercase, trim, remove NBSP and spaces/punct)
    NormalizeKey = (s as text) as text =>
        let
            s0 = Text.From(s),
            s1 = Text.Replace(s0, Character.FromNumber(160), " "),
            s2 = Text.Trim(s1),
            s3 = Text.Lower(s2),
            s4 = Text.Select(s3, List.Combine({{"a".."z"},{"0".."9"}}))
        in s4,

    // 3) Build lookup from original headers -> normalized keys
    Cols    = Table.ColumnNames(Promoted),
    KeyMap  = Table.FromColumns({Cols, List.Transform(Cols, each NormalizeKey(_))}, {"Original","Key"}),

    // 4) Helper to fetch original column name by acceptable normalized key(s)
    FindCol = (keys as list) as text =>
        let hit = Table.SelectRows(KeyMap, each List.Contains(keys, [Key]))
        in if Table.RowCount(hit) > 0 then hit{0}[Original] else error "Missing column for keys: " & Text.Combine(keys, "/"),

    // 5) Resolve required columns (case/space tolerant)
    LegalCol = FindCol({"legalentity"}),
    ProdCol  = FindCol({"product"}),
    TypeCol  = FindCol({"agreementtype"}),
    VersCol  = FindCol({"agreementversion"}),
    CpidCol  = FindCol({"counterpartyid"}),

    // 6) Standardize names
    Std = Table.RenameColumns(
        Promoted,
        {
            {LegalCol, "Legal Entity"},
            {ProdCol,  "Product"},
            {TypeCol,  "Agreement Type"},
            {VersCol,  "Agreement Version"},
            {CpidCol,  "counterparty id"}
        },
        MissingField.Ignore
    ),

    // 7) Unpivot the two agreement columns (won’t error now)
    Unpivoted = Table.Unpivot(Std, {"Agreement Type","Agreement Version"}, "Attribute", "Value"),

    // 8) Build composite column name = Product + Attribute
    WithHeader = Table.AddColumn(Unpivoted, "ProdAttr", each [Product] & " " & [Attribute]),

    // 9) Keep row keys: counterparty id + Legal Entity
    Slim = Table.RemoveColumns(WithHeader, {"Product","Attribute"}),

    // 10) Pivot wide; List.First handles duplicates
    Wide = Table.Pivot(Slim, List.Distinct(Slim[ProdAttr]), "ProdAttr", "Value", List.First),

    // 11) Pretty headers
    Renamed = Table.RenameColumns(Wide, {
        {"REPO Agreement Type","REPO Agreement Type"},
        {"REPO Agreement Version","REPO Agreement Version"},
        {"SLEB Agreement Type","SLEB Agreement Type"},
        {"SLEB Agreement Version","SLEB Agreement Version"}
    }, MissingField.Ignore)
in
    Renamed
