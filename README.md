let
    // 1. Source Data
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("i45WMjIwMjbQM9QzMDJSitWBCk/MyUxWSslNycjPzQeSMjXgWlUaM7Q1MjUEMsI8hGpkY7CwEChWjIuLz83PysgsBArxIkgGZgbGhsbmhka4dGpqYGYEUMhHqU6K9QyNNYFg7OICkE4OICkGqLg6J+g6J+g6J+i01gOQzQwNAKkjMDSgEisgGQOQzgwMAjVjozRjR1jZgJbZgUqg0YxRzR2AqkdmBqQGAI1T0ZlK9S0MDSGEgLGRsTAzMBkC8k0MDSGEpY5nBkYGAuBfFjpjJ1BwA2UvYHEuIgoZ2BpYGYEUCyL0QxQ9Pdw9B0c6PdwgK6BqQmB2AGtGqQxKDW1gUu9gU242sY+fNzA5F1gJg8M2gPzHwE=", BinaryEncoding.Base64), Compression.GZip)), let _t = ((type nullable text) meta [Serialized.Text = true]) in type table [#"Legal Entity" = _t, Product = _t, #"Agreement Type" = _t, #"Agreement Version" = _t, #"counterparty id" = _t]),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Legal Entity", type text}, {"Product", type text}, {"Agreement Type", type text}, {"Agreement Version", type text}, {"counterparty id", Int64.Type}}),

    // 2. Group by counterparty id to prepare for pivoting
    Grouped = Table.Group(#"Changed Type", {"counterparty id", "Legal Entity", "Product"}, {
        {"Agreement Type", each List.First([Agreement Type]), type text},
        {"Agreement Version", each List.First([Agreement Version]), type text}
    }),

    // 3. Pivot the "Agreement Type" column
    Pivoted_Type = Table.Pivot(Grouped, List.Distinct(Grouped[Product]), "Product", "Agreement Type", Combiner.First),

    // 4. Pivot the "Agreement Version" column
    Pivoted_Version = Table.Pivot(Grouped, List.Distinct(Grouped[Product]), "Product", "Agreement Version", Combiner.First),

    // 5. Merge the two pivoted tables based on counterparty and legal entity
    Merged = Table.NestedJoin(Pivoted_Type, {"counterparty id", "Legal Entity"}, Pivoted_Version, {"counterparty id", "Legal Entity"}, "Version", JoinKind.Inner),

    // 6. Expand the merged table to add the new columns
    Expanded = Table.ExpandTableColumn(Merged, "Version", {"REPO", "SLEB"}, {"REPO Version", "SLEB Version"}),

    // 7. Rename the columns for a clean final output
    #"Renamed Columns" = Table.RenameColumns(Expanded, {
        {"REPO", "REPO Agreement Type"}, 
        {"SLEB", "SLEB Agreement Type"}
    }),

    // 8. Reorder columns for readability
    #"Reordered Columns" = Table.ReorderColumns(#"Renamed Columns", {"counterparty id", "Legal Entity", "REPO Agreement Type", "REPO Version", "SLEB Agreement Type", "SLEB Version"})
in
    #"Reordered Columns"
