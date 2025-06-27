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


import org.postgresql.PGConnection;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.PreDestroy;
import java.sql.*;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@Component
public class OutboxListener implements CommandLineRunner {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    private Connection connection;
    private ExecutorService executor = Executors.newSingleThreadExecutor();

    @Override
    public void run(String... args) throws Exception {
        connection = jdbcTemplate.getDataSource().getConnection();
        PGConnection pgConnection = connection.unwrap(PGConnection.class);
        pgConnection.addNotificationListener((processId, channelName, payload) -> {
            System.out.println("🔔 New Outbox Message ID: " + payload);

            // Load message from table
            String json = jdbcTemplate.queryForObject(
                "SELECT payload FROM kafka_message_outbox WHERE id = ?", new Object[]{Integer.valueOf(payload)}, String.class);

            System.out.println("📦 Payload: " + json);

            // Simulate business logic
            // e.g., call stored proc or another service
        });

        try (Statement stmt = connection.createStatement()) {
            stmt.execute("LISTEN outbox_channel");
        }

        executor.submit(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                PGConnection pgConn = connection.unwrap(PGConnection.class);
                pgConn.getNotifications(); // triggers listener
                Thread.sleep(1000); // avoid busy waiting
            }
        });

        System.out.println("✅ Listening on 'outbox_channel'...");
    }

    @PreDestroy
    public void cleanup() throws SQLException {
        executor.shutdownNow();
        connection.close();
    }
}

