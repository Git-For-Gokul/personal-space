# Configure properties for the default Spring Task Executor
spring.task.execution.pool.core-size=5       # Min threads always active
spring.task.execution.pool.max-size=20       # Max threads when queue is full
spring.task.execution.pool.queue-capacity=100 # Capacity of the queue for tasks
spring.task.execution.thread-name-prefix=notification-processor-
spring.task.execution.shutdown.await-termination=true
spring.task.execution.shutdown.await-termination-period=30s

// In your main Spring Boot application class (e.g., YourApplication.java)
@SpringBootApplication
@EnableAsync // <--- Add this!
public class YourApplication {
    public static void main(String[] args) {
        SpringApplication.run(YourApplication.class, args);
    }
}

public class NotificationEvent {
    private int processId;
    private String channelName;
    private String payload;
    private long receivedTimestamp; // To track order if needed

    public NotificationEvent(int processId, String channelName, String payload) {
        this.processId = processId;
        this.channelName = channelName;
        this.payload = payload;
        this.receivedTimestamp = System.currentTimeMillis();
    }

    // Getters
    public int getProcessId() { return processId; }
    public String getChannelName() { return channelName; }
    public String getPayload() { return payload; }
    public long getReceivedTimestamp() { return receivedTimestamp; }

    @Override
    public String toString() {
        return "NotificationEvent{" +
               "processId=" + processId +
               ", channelName='" + channelName + '\'' +
               ", payload='" + payload + '\'' +
               ", receivedTimestamp=" + receivedTimestamp +
               '}';
    }
}

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

@Service
public class NotificationProcessingService {

    // Use LinkedBlockingQueue for an unbounded queue.
    // For a bounded queue, use new ArrayBlockingQueue<>(capacity).
    private final BlockingQueue<NotificationEvent> notificationQueue = new LinkedBlockingQueue<>();
    private final AtomicBoolean running = new AtomicBoolean(true);

    // This is where your heavy task logic would reside.
    // Make sure this method runs asynchronously.
    @Async // Will use the default TaskExecutor or a custom one if defined
    public void processNotification(NotificationEvent event) {
        System.out.println("Processing notification (Thread: " + Thread.currentThread().getName() + "): " + event.getChannelName() + " - " + event.getPayload());
        // Simulate a heavy task
        try {
            Thread.sleep(2000); // e.g., 2 seconds of work
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.err.println("Notification processing interrupted for: " + event.getPayload());
        }
        System.out.println("Finished processing notification: " + event.getPayload());
    }

    // Method to add notifications to the queue
    public void enqueueNotification(NotificationEvent event) {
        try {
            notificationQueue.put(event); // .put() blocks if the queue is full (for BlockingQueue)
            System.out.println("Enqueued notification: " + event.getPayload() + ". Queue size: " + notificationQueue.size());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.err.println("Failed to enqueue notification: " + event.getPayload());
        }
    }

    // Consumer loop to continuously take from the queue and submit to async processing
    @Bean
    public ApplicationRunner queueConsumer() {
        return args -> {
            while (running.get()) {
                try {
                    // Try to take an event from the queue. Wait for a short period.
                    NotificationEvent event = notificationQueue.poll(100, TimeUnit.MILLISECONDS);
                    if (event != null) {
                        processNotification(event); // Call the @Async method
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    System.out.println("Notification queue consumer interrupted.");
                    break;
                } catch (Exception e) {
                    System.err.println("Error while consuming notification: " + e.getMessage());
                    // Log the error, maybe push back to a dead-letter queue or handle
                }
            }
            System.out.println("Notification queue consumer stopped.");
        };
    }

    @PreDestroy
    public void shutdown() {
        running.set(false); // Signal the consumer loop to stop
        // You might want to wait for pending tasks in the queue or executor to finish
        System.out.println("Shutting down NotificationProcessingService. Remaining tasks in queue: " + notificationQueue.size());
    }
}

// In your DataService.java
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
    private final NotificationProcessingService notificationProcessingService; // Inject the new service

    public DataService(JdbcTemplate defaultJdbcTemplate,
                       @Qualifier("pgJdbcNgListenConnection") PGConnection pgJdbcNgListenConnection,
                       NotificationProcessingService notificationProcessingService) { // Inject it
        this.defaultJdbcTemplate = defaultJdbcTemplate;
        this.pgJdbcNgListenConnection = pgJdbcNgListenConnection;
        this.notificationProcessingService = notificationProcessingService; // Assign
    }

    public void performRegularDatabaseOperation() {
        Integer count = defaultJdbcTemplate.queryForObject("SELECT COUNT(*) FROM your_table", Integer.class);
        System.out.println("Regular DB operation (official pgJDBC): Count = " + count);
    }

    @PostConstruct
    public void setupPgJdbcNgListener() {
        try {
            pgJdbcNgListenConnection.addNotificationListener(new PGNotificationListener() {
                @Override
                public void notification(int processId, String channelName, String payload) {
                    System.out.println("RECEIVED notification (pgjdbc-ng): PID=" + processId + ", Channel=" + channelName + ", Payload=" + payload + " - Enqueuing...");
                    // Immediately enqueue the notification, don't do heavy work here
                    notificationProcessingService.enqueueNotification(new NotificationEvent(processId, channelName, payload));
                }
            });
            pgJdbcNgListenConnection.executeNativeQuery("LISTEN my_channel");
            System.out.println("Listening on 'my_channel' using pgjdbc-ng");
        } catch (SQLException e) {
            System.err.println("Error setting up pgjdbc-ng listener: " + e.getMessage());
            // Handle error appropriately
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



