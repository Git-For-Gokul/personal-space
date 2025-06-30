// NotificationProcessingService.java
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import javax.annotation.PreDestroy;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

@Service
public class NotificationProcessingService {

    private final Map<String, String> latestNotificationMap = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);
    private final long debounceDelaySeconds = 5;

    public void scheduleDebouncedProcessing(String payload) {
        String key = extractKeyFromPayload(payload); // implement this based on payload
        latestNotificationMap.put(key, payload);

        scheduler.schedule(() -> {
            String latestPayload = latestNotificationMap.remove(key);
            if (latestPayload != null) {
                processNotification(latestPayload);
            }
        }, debounceDelaySeconds, TimeUnit.SECONDS);
    }

    private void processNotification(String payload) {
        System.out.println("[Debounced] Processing latest notification for payload: " + payload);
        // TODO: Handle Kafka outbox update and superseding logic here.
    }

    private String extractKeyFromPayload(String payload) {
        // Parse counterparty ID or logical key from payload
        // This is a placeholder:
        return payload.split(":")[0];
    }

    @PreDestroy
    public void shutdown() {
        scheduler.shutdown();
        try {
            if (!scheduler.awaitTermination(5, TimeUnit.SECONDS)) {
                scheduler.shutdownNow();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            scheduler.shutdownNow();
        }
        System.out.println("NotificationProcessingService shutdown complete.");
    }
}

// DataService.java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import com.impossibl.postgres.api.jdbc.PGConnection;
import com.impossibl.postgres.api.jdbc.PGNotificationListener;
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.sql.SQLException;

@Service
public class DataService {

    private final JdbcTemplate defaultJdbcTemplate;
    private final PGConnection pgJdbcNgListenConnection;
    private final NotificationProcessingService notificationProcessingService;

    public DataService(JdbcTemplate defaultJdbcTemplate,
                       @Qualifier("pgJdbcNgListenConnection") PGConnection pgJdbcNgListenConnection,
                       NotificationProcessingService notificationProcessingService) {
        this.defaultJdbcTemplate = defaultJdbcTemplate;
        this.pgJdbcNgListenConnection = pgJdbcNgListenConnection;
        this.notificationProcessingService = notificationProcessingService;
    }

    @PostConstruct
    public void setupPgJdbcNgListener() {
        try {
            pgJdbcNgListenConnection.addNotificationListener(new PGNotificationListener() {
                @Override
                public void notification(int processId, String channelName, String payload) {
                    System.out.println("RECEIVED notification (pgjdbc-ng): PID=" + processId + ", Channel=" + channelName + ", Payload=" + payload);
                    notificationProcessingService.scheduleDebouncedProcessing(payload);
                }
            });
            pgJdbcNgListenConnection.executeNativeQuery("LISTEN my_channel");
            System.out.println("Listening on 'my_channel' using pgjdbc-ng");
        } catch (SQLException e) {
            System.err.println("Error setting up pgjdbc-ng listener: " + e.getMessage());
        }
    }

    @PreDestroy
    public void cleanupPgJdbcNgListener() {
        try {
            if (pgJdbcNgListenConnection != null && !pgJdbcNgListenConnection.isClosed()) {
                pgJdbcNgListenConnection.executeNativeQuery("UNLISTEN my_channel");
                pgJdbcNgListenConnection.close();
                System.out.println("Unlistening and closing pgjdbc-ng connection.");
            }
        } catch (SQLException e) {
            System.err.println("Error cleaning up pgjdbc-ng listener: " + e.getMessage());
        }
    }

    public void sendNotificationFromDefaultDriver() {
        defaultJdbcTemplate.execute("SELECT pg_notify('my_channel', 'Hello from official pgJDBC!')");
        System.out.println("Sent notification 'Hello from official pgJDBC!' via default pgJDBC.");
    }
}
