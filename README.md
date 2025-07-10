RAISE NOTICE 'Changed keys for column %: %', json_col, changed_keys;
RAISE NOTICE 'Evaluating config: column=%, json_key=%, changed_keys_map=%',
             config.column_name, config.json_key, changed_json_keys_map;
RAISE NOTICE 'Matched config for version bump: column=%, json_key=%',
             config.column_name, config.json_key;
