CREATE OR REPLACE FUNCTION trg_versioning_handler()
RETURNS TRIGGER AS $$
DECLARE
    config RECORD;
    update_needed BOOLEAN := FALSE;
    changed_columns TEXT[];
    changed_json_keys_map JSONB := '{}';
    row_code_type TEXT;
    json_col TEXT;
    changed_keys TEXT[];
BEGIN
    IF TG_OP = 'TRUNCATE' THEN
        PERFORM update_counterparty_version(NULL);
        RETURN NULL;
    END IF;

    row_code_type := COALESCE(NEW.code_type, OLD.code_type);

    IF TG_OP IN ('INSERT', 'DELETE') THEN
        FOR config IN
            SELECT * FROM versioning_config
            WHERE table_name = TG_TABLE_NAME
              AND (code_type IS NULL OR code_type = row_code_type)
              AND ignore = false
        LOOP
            update_needed := TRUE;
            EXIT;
        END LOOP;

    ELSE
        -- Get changed columns
        SELECT array_agg(skeys)
        INTO changed_columns
        FROM (
            SELECT skeys(hstore(NEW) - hstore(OLD)) AS skeys
        ) sub;

        -- For each distinct JSON column in config, compute changed keys
        FOR json_col IN
            SELECT DISTINCT column_name
            FROM versioning_config
            WHERE table_name = TG_TABLE_NAME
              AND json_key IS NOT NULL
              AND ignore = false
              AND (code_type IS NULL OR code_type = row_code_type)
        LOOP
            IF json_col = ANY(changed_columns) THEN
                EXECUTE format(
                    $sql$
                    SELECT array_agg(key)
                    FROM (
                        SELECT key FROM jsonb_each_text($1.%I)
                        EXCEPT
                        SELECT key FROM jsonb_each_text($2.%I)
                        UNION
                        SELECT key FROM jsonb_each_text($2.%I)
                        EXCEPT
                        SELECT key FROM jsonb_each_text($1.%I)
                    ) diff_keys
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

        -- Now check against config
        FOR config IN
            SELECT * FROM versioning_config
            WHERE table_name = TG_TABLE_NAME
              AND (code_type IS NULL OR code_type = row_code_type)
              AND ignore = false
        LOOP
            IF config.column_name = '*' THEN
                update_needed := TRUE;
                EXIT;

            ELSIF config.json_key IS NULL AND config.column_name = ANY(changed_columns) THEN
                update_needed := TRUE;
                EXIT;

            ELSIF config.json_key IS NOT NULL AND config.column_name = ANY(changed_columns) THEN
                IF changed_json_keys_map ? config.column_name THEN
                    IF config.json_key = ANY (
                        (changed_json_keys_map -> config.column_name)::TEXT[]
                    ) THEN
                        update_needed := TRUE;
                        EXIT;
                    END IF;
                END IF;
            END IF;
        END LOOP;
    END IF;

    IF update_needed THEN
        PERFORM update_counterparty_version(NEW.counterparty_did);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
