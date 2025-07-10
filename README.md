CREATE OR REPLACE FUNCTION trg_versioning_handler()
RETURNS TRIGGER AS $$
DECLARE
    config RECORD;
    update_needed BOOLEAN := FALSE;
    changed_columns TEXT[];
    changed_json_keys_map JSONB := '{}';
    new_val TEXT;
    old_val TEXT;
    json_keys TEXT[];
BEGIN
    -- Truncate handling (statement-level)
    IF TG_OP = 'TRUNCATE' THEN
        PERFORM update_counterparty_version(NULL);
        RETURN NULL;
    END IF;

    -- Insert/Delete handling: always version unless ignored
    IF TG_OP IN ('INSERT', 'DELETE') THEN
        update_needed := TRUE;
        -- Skip checking config here — assume INSERT/DELETE is always a version bump
        -- If you want to allow ignoring insert/delete for specific rows, add that logic here
    ELSE
        -- Get changed columns
        SELECT array_agg(skeys)
        INTO changed_columns
        FROM (
            SELECT skeys(hstore(NEW) - hstore(OLD)) AS skeys
        ) sub;

        FOR config IN
            SELECT * FROM versioning_config
            WHERE table_name = TG_TABLE_NAME AND ignore = false
        LOOP
            -- Case 1: Wildcard (trigger for any change)
            IF config.column_name = '*' THEN
                update_needed := TRUE;
                EXIT;

            -- Case 2: Regular column
            ELSIF config.json_key IS NULL AND config.column_name = ANY(changed_columns) THEN
                EXECUTE format(
                    'SELECT $1.%I IS DISTINCT FROM $2.%I',
                    config.column_name, config.column_name
                ) INTO update_needed
                USING NEW, OLD;

                IF update_needed THEN
                    EXIT;
                END IF;

            -- Case 3: JSON key inside JSON column
            ELSIF config.json_key IS NOT NULL AND config.column_name = ANY(changed_columns) THEN
                -- Only compute changed JSON keys for this column once
                IF NOT changed_json_keys_map ? config.column_name THEN
                    EXECUTE format(
                        $$SELECT jsonb_agg(key) FROM (
                            SELECT key FROM jsonb_each_text($1.%I)
                            EXCEPT
                            SELECT key FROM jsonb_each_text($2.%I)
                            UNION
                            SELECT key FROM jsonb_each_text($2.%I)
                            EXCEPT
                            SELECT key FROM jsonb_each_text($1.%I)
                        ) diff_keys$$,
                        config.column_name, config.column_name,
                        config.column_name, config.column_name
                    )
                    INTO json_keys
                    USING NEW, OLD, OLD, NEW;

                    -- Store for reuse
                    changed_json_keys_map := jsonb_set(changed_json_keys_map, ARRAY[config.column_name], to_jsonb(json_keys));
                ELSE
                    json_keys := ARRAY(SELECT jsonb_array_elements_text(changed_json_keys_map -> config.column_name));
                END IF;

                -- Only compare if this JSON key changed
                IF config.json_key = ANY(json_keys) THEN
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
            END IF;
        END LOOP;
    END IF;

    -- Update version if needed
    IF update_needed THEN
        PERFORM update_counterparty_version(NEW.counterparty_did);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
