CREATE OR REPLACE FUNCTION increment_counterparty_version_selected_columns()
RETURNS TRIGGER AS $$
DECLARE
    cp_id BIGINT;
    config RECORD;
    update_needed BOOLEAN := FALSE;
    old_val TEXT;
    new_val TEXT;
BEGIN
    IF TG_OP = 'DELETE' THEN
        cp_id := OLD.counterparty_dbid;
        update_needed := TRUE;  -- Always version on DELETE

    ELSIF TG_OP = 'INSERT' THEN
        cp_id := NEW.counterparty_dbid;
        update_needed := TRUE;  -- Always version on INSERT

    ELSIF TG_OP = 'UPDATE' THEN
        cp_id := NEW.counterparty_dbid;

        FOR config IN
            SELECT * FROM versioning_config
            WHERE table_name = TG_TABLE_NAME
        LOOP
            -- Case 1: Simple column
            IF config.json_key IS NULL THEN
                EXECUTE format('SELECT $1.%I IS DISTINCT FROM $2.%I', config.column_name, config.column_name)
                INTO update_needed
                USING NEW, OLD;

                IF update_needed THEN
                    EXIT;
                END IF;

            -- Case 2: JSON column + key
            ELSE
                EXECUTE format('SELECT ($1.%I ->> $2) IS DISTINCT FROM ($2.%I ->> $2)',
                               config.column_name, config.column_name)
                INTO update_needed
                USING NEW, config.json_key, OLD, config.json_key;

                IF update_needed THEN
                    EXIT;
                END IF;
            END IF;
        END LOOP;
    END IF;

    IF update_needed THEN
        INSERT INTO counterparty_version(counterparty_dbid, version, last_updated)
        VALUES (cp_id, 1, CURRENT_TIMESTAMP)
        ON CONFLICT (counterparty_dbid)
        DO UPDATE SET
            version = counterparty_version.version + 1,
            last_updated = CURRENT_TIMESTAMP;
    END IF;

    RETURN CASE WHEN TG_OP = 'DELETE' THEN OLD ELSE NEW END;
END;
$$ LANGUAGE plpgsql;


CREATE TRIGGER trg_version_increment
AFTER INSERT OR UPDATE OR DELETE ON counterparty_info
FOR EACH ROW
EXECUTE FUNCTION increment_counterparty_version_selected_columns();


-- Monitor a regular column
INSERT INTO versioning_config(table_name, column_name)
VALUES ('counterparty_info', 'name');

-- Monitor a JSON key
INSERT INTO versioning_config(table_name, column_name, json_key)
VALUES ('counterparty_info', 'data_json', 'status');


CREATE TABLE versioning_config (
    table_name TEXT NOT NULL,
    column_name TEXT DEFAULT '*',   -- '*' means all columns
    json_key TEXT,                 -- Optional; only applies if column_name is a JSON column
    PRIMARY KEY (table_name, column_name, json_key)
);

-- Check if wildcard is present
SELECT EXISTS (
    SELECT 1 FROM versioning_config
    WHERE table_name = TG_TABLE_NAME
    AND column_name = '*'
) INTO update_needed;

-- If no wildcard, check column-level config
IF NOT update_needed THEN
    FOR config IN ...
    LOOP
        -- Same as before: check individual column or json_key
        ...
    END LOOP;
END IF;

Im designing the kafka message outbox,

CREATE TABLE kafka_message_outbox (
    id                  BIGSERIAL PRIMARY KEY,
    topic_name          TEXT NOT NULL,
    key                 TEXT,
    message_headers     JSONB,
    payload             JSONB NOT NULL,

    created_by          TEXT,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    status              TEXT NOT NULL DEFAULT 'PENDING', -- PENDING, SENT, FAILED
    sent_at             TIMESTAMPTZ,
    error_message       TEXT,
    retry_count         INT DEFAULT 0
);


CREATE INDEX idx_outbox_topic_status_created_at 
ON kafka_message_outbox (topic_name, status, created_at);



