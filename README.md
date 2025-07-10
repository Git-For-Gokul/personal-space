CREATE OR REPLACE FUNCTION trg_versioning_handler()
RETURNS TRIGGER AS $$
DECLARE
    update_needed BOOLEAN := FALSE;
    changed_columns TEXT[];
    changed_json_keys_map JSONB := '{}';
    json_col TEXT;
    changed_keys TEXT[];
BEGIN
    IF TG_OP = 'TRUNCATE' THEN
        PERFORM update_counterparty_version(NULL);
        RETURN NULL;
    END IF;

    IF TG_OP IN ('INSERT', 'DELETE') THEN
        -- If there's a wildcard rule or column-specific rule, consider it a change
        SELECT EXISTS (
            SELECT 1 FROM versioning_config
            WHERE table_name = TG_TABLE_NAME
              AND (column_name = '*' OR json_key IS NULL OR json_key IS NOT NULL)
        ) INTO update_needed;

    ELSE
        -- Get changed columns
        SELECT array_agg(skeys)
        INTO changed_columns
        FROM (
            SELECT skeys(hstore(NEW) - hstore(OLD)) AS skeys
        ) sub;

        -- Build map of changed json keys per json column
        FOR json_col IN
            SELECT DISTINCT column_name
            FROM versioning_config
            WHERE table_name = TG_TABLE_NAME
              AND json_key IS NOT NULL
        LOOP
            IF json_col = ANY(changed_columns) THEN
                EXECUTE format(
                    $sql$
                    SELECT array_agg(key)
                    FROM (
                        SELECT key, value FROM jsonb_each_text($1.%I)
                        EXCEPT
                        SELECT key, value FROM jsonb_each_text($2.%I)
                        UNION
                        SELECT key, value FROM jsonb_each_text($2.%I)
                        EXCEPT
                        SELECT key, value FROM jsonb_each_text($1.%I)
                    ) changed
                    $sql$,
                    json_col, json_col, json_col, json_col
                )
                INTO changed_keys
                USING NEW, OLD;

                IF changed_keys IS NOT NULL THEN
                    changed_json_keys_map := jsonb_set(
                        changed_json_keys_map,
                        ARRAY[json_col],
                        to_jsonb(changed_keys)
                    );
                END IF;
            END IF;
        END LOOP;

        -- Shortcut 1: wildcard column matches
        SELECT EXISTS (
            SELECT 1
            FROM versioning_config
            WHERE table_name = TG_TABLE_NAME
              AND column_name = '*'
        ) INTO update_needed;

        -- Shortcut 2: any direct column changes (non-JSON)
        IF NOT update_needed THEN
            SELECT EXISTS (
                SELECT 1
                FROM versioning_config
                WHERE table_name = TG_TABLE_NAME
                  AND json_key IS NULL
                  AND column_name = ANY(changed_columns)
            ) INTO update_needed;
        END IF;

        -- Shortcut 3: any configured JSON keys were changed
        IF NOT update_needed THEN
            SELECT EXISTS (
                SELECT 1
                FROM versioning_config vc
                WHERE vc.table_name = TG_TABLE_NAME
                  AND vc.json_key IS NOT NULL
                  AND vc.column_name = ANY(changed_columns)
                  AND EXISTS (
                      SELECT 1
                      FROM jsonb_array_elements_text(changed_json_keys_map -> vc.column_name) AS key
                      WHERE key = vc.json_key
                  )
            ) INTO update_needed;
        END IF;
    END IF;

    IF update_needed THEN
        PERFORM update_counterparty_version(NEW.counterparty_did);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
