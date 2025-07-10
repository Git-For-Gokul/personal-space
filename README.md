CREATE OR REPLACE FUNCTION trg_versioning_handler()
RETURNS TRIGGER AS $$
DECLARE
    config RECORD;
    update_needed BOOLEAN := FALSE;
    changed_columns TEXT[];
    changed_json_keys TEXT[];
    new_val TEXT;
    old_val TEXT;
    row_code_type TEXT;
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

        -- Pre-detect changed JSON keys if needed
        IF 'your_json_column_name' = ANY(changed_columns) THEN
            EXECUTE format(
                $sql$
                SELECT array_agg(key)
                FROM (
                    SELECT key FROM jsonb_each_text($1.your_json_column_name)
                    EXCEPT
                    SELECT key FROM jsonb_each_text($2.your_json_column_name)
                    UNION
                    SELECT key FROM jsonb_each_text($2.your_json_column_name)
                    EXCEPT
                    SELECT key FROM jsonb_each_text($1.your_json_column_name)
                ) diff_keys
                $sql$
            )
            INTO changed_json_keys
            USING NEW, OLD;
        ELSE
            changed_json_keys := ARRAY[]::TEXT[];
        END IF;

        -- Main loop
        FOR config IN
            SELECT * FROM versioning_config
            WHERE table_name = TG_TABLE_NAME
              AND (code_type IS NULL OR code_type = row_code_type)
              AND ignore = false
        LOOP
            -- Wildcard: any column change triggers version
            IF config.column_name = '*' THEN
                update_needed := TRUE;
                EXIT;

            -- Regular column
            ELSIF config.json_key IS NULL AND config.column_name = ANY(changed_columns) THEN
                EXECUTE format(
                    'SELECT $1.%I IS DISTINCT FROM $2.%I',
                    config.column_name, config.column_name
                )
                INTO update_needed
                USING NEW, OLD;

                IF update_needed THEN
                    EXIT;
                END IF;

            -- JSON key
            ELSIF config.json_key IS NOT NULL AND config.column_name = ANY(changed_columns)
                  AND config.json_key = ANY(changed_json_keys) THEN

                EXECUTE format(
                    'SELECT ($1.%I ->> %L)', config.column_name, config.json_key
                ) INTO new_val USING NEW;

                EXECUTE format(
                    'SELECT ($1.%I ->> %L)', config.column_name, config.json_key
                ) INTO old_val USING OLD;

                IF new_val IS DISTINCT FROM old_val THEN
                    update_needed := TRUE;
                    EXIT;
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
