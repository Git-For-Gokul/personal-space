CREATE TABLE kafka_message_outbox (
    id SERIAL PRIMARY KEY,
    topic VARCHAR(100),
    payload JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE OR REPLACE FUNCTION notify_outbox_insert()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('outbox_channel', NEW.id::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_notify_outbox
AFTER INSERT ON kafka_message_outbox
FOR EACH ROW EXECUTE FUNCTION notify_outbox_insert();

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.2</version>
    </dependency>
</dependencies>

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/testdb
    username: youruser
    password: yourpass

import com.impossibl.postgres.api.jdbc.PGConnection;
import com.impossibl.postgres.jdbc.PGDataSource;

import java.sql.Connection;
import java.sql.Statement;

public class PgListenApp {

    public static void main(String[] args) throws Exception {

        PGDataSource dataSource = new PGDataSource();
        dataSource.setHost("localhost");
        dataSource.setPort(5432);
        dataSource.setDatabase("your_db");
        dataSource.setUser("your_user");
        dataSource.setPassword("your_password");

        PGConnection connection = (PGConnection) dataSource.getConnection();

        // Register listener
        connection.addNotificationListener((channel, payload) -> {
            System.out.println("🔔 Received notification from channel: " + channel);
            System.out.println("📦 Payload: " + payload);
        });

        // Start listening to a channel
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("LISTEN my_channel");
        }

        System.out.println("✅ Now listening on 'my_channel'...");

        // Keep alive
        while (true) {
            Thread.sleep(1000);
        }
    }
}

