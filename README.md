let
    // 1. Define the source data as a table
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("i45WMjIwMjbQM9QzMDJSitWBCk/MyUxWSslNycjPzQeSMjXgWlUaM7Q1MjUEMsI8hGpkY7CwEChWjIuLz83PysgsBArxIkgGZgbGhsbmhka4dGpqYGYEUMhHqU6K9QyNNYFg7OICkE4OICkGqLg6J+g6J+g6J+i01gOQzQwNAKkjMDSgEisgGQOQzgwMAjVjozRjR1jZgJbZgUqg0YxRzR2AqkdmBqQGAI1T0ZlK9S0MDSGEgLGRsTAzMBkC8k0MDSGEpY5nBkYGAuBfFjpjJ1BwA2UvYHEuIgoZ2BpYGYEUCyL0QxQ9Pdw9B0c6PdwgK6BqQmB2AGtGqQxKDW1gUu9gU242sY+fNzA5F1gJg8M2gPzHwE=", BinaryEncoding.Base64), Compression.GZip)), let _t = ((type nullable text) meta [Serialized.Text = true]) in type table [#"Legal Entity" = _t, Product = _t, #"Agreement Type" = _t, #"Agreement Version" = _t, #"counterparty id" = _t]),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Legal Entity", type text}, {"Product", type text}, {"Agreement Type", type text}, {"Agreement Version", type text}, {"counterparty id", Int64.Type}}),

    // 2. Pivot the 'Agreement Type' column
    #"Pivoted Agreement Type" = Table.Pivot(
        #"Changed Type", 
        List.Distinct(#"Changed Type"[Product]), 
        "Product", 
        "Agreement Type", 
        Combiner.First),

    // 3. Pivot the 'Agreement Version' column
    #"Pivoted Agreement Version" = Table.Pivot(
        #"Changed Type", 
        List.Distinct(#"Changed Type"[Product]), 
        "Product", 
        "Agreement Version", 
        Combiner.First),

    // 4. Merge the two pivoted tables
    MergedTables = Table.NestedJoin(
        #"Pivoted Agreement Type", 
        {"counterparty id"}, 
        #"Pivoted Agreement Version", 
        {"counterparty id"}, 
        "NewColumn", 
        JoinKind.Inner),

    // 5. Expand the merged table to get the new columns
    #"Expanded NewColumn" = Table.ExpandTableColumn(
        MergedTables, 
        "NewColumn", 
        {"REPO", "SLEB"}, 
        {"REPO Version", "SLEB Version"}),

    // 6. Rename columns to be more descriptive
    #"Renamed Columns" = Table.RenameColumns(
        #"Expanded NewColumn",
        {
            {"REPO", "REPO Agreement Type"}, 
            {"SLEB", "SLEB Agreement Type"}
        }),

    // 7. Reorder columns for a clean final output
    #"Reordered Columns" = Table.ReorderColumns(
        #"Renamed Columns",
        {
            "counterparty id", 
            "REPO Agreement Type", 
            "REPO Version", 
            "SLEB Agreement Type", 
            "SLEB Version"
        })
in
    #"Reordered Columns"
