ELSIF config.json_key IS NOT NULL AND config.column_name = ANY(changed_columns) THEN
    IF changed_json_keys_map ? config.column_name THEN
        IF EXISTS (
            SELECT 1
            FROM jsonb_array_elements_text(changed_json_keys_map -> config.column_name) AS key
            WHERE key = config.json_key
        ) THEN
            update_needed := TRUE;
            EXIT;
        END IF;
    END IF;
END IF;
