    // 9) Merge the two pivoted tables on both keys using NestedJoin
    Merged = Table.NestedJoin(
        Pivoted_Type_Renamed,
        {"counterparty id","Legal Entity"},
        Pivoted_Version_Renamed,
        {"counterparty id","Legal Entity"},
        "VersionTable",
        JoinKind.FullOuter
    ),

    // 10) Expand the merged table to pull in version columns only
    Expanded = Table.ExpandTableColumn(
        Merged,
        "VersionTable",
        List.RemoveItems(Table.ColumnNames(Pivoted_Version_Renamed), {"counterparty id","Legal Entity"})
    ),

    // 11) Reorder columns (only include those that exist)
    DesiredOrder = {"counterparty id","Legal Entity"} 
                   & List.Transform(ProductList, each _ & " Agreement Type")
                   & List.Transform(ProductList, each _ & " Version"),

    FinalOrder = List.Select(DesiredOrder, each List.Contains(Table.ColumnNames(Expanded), _)),

    Final = Table.ReorderColumns(Expanded, FinalOrder)
