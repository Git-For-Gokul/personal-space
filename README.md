import com.impossibl.postgres.api.jdbc.PGConnection;
import jakarta.annotation.PostConstruct; // For Spring's lifecycle callbacks (equivalent to InitializingBean's afterPropertiesSet)
import jakarta.annotation.PreDestroy; // For Spring's lifecycle callbacks (equivalent to DisposableBean's destroy)
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Service;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * A Spring @Service responsible for managing a dedicated PostgreSQL connection
 * to listen for NOTIFY events.
 * This service implements ApplicationRunner to start a background thread for listening
 * once the Spring context is fully initialized.
 * It also uses @PreDestroy for graceful shutdown and connection closure.
 */
@Service
public class PgNotificationListenerService implements ApplicationRunner {

    private static final Logger logger = LoggerFactory.getLogger(PgNotificationListenerService.class);

    private final DataSource dataSource; // Spring will inject the configured DataSource
    private Connection notificationConnection; // Dedicated connection for notifications
    private PGConnection pgConnection; // Unwrapped pgjdbc-ng specific connection
    private ExecutorService executorService; // Executor for the listening thread
    private volatile boolean running = true; // Flag for graceful thread termination

    private static final int MAX_RETRY_ATTEMPTS = 5; // Maximum connection retry attempts
    private static final long RETRY_DELAY_MS = 5000; // Delay between retries in milliseconds

    /**
     * Constructor for dependency injection. Spring automatically injects the DataSource.
     * @param dataSource The configured JDBC DataSource.
     */
    public PgNotificationListenerService(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    /**
     * This method is called by Spring Boot after the application context is fully loaded.
     * It starts the dedicated thread for listening to PostgreSQL notifications.
     * @param args Application arguments.
     */
    @Override
    public void run(ApplicationArguments args) {
        // Initialize a single-threaded executor for our listener
        executorService = Executors.newSingleThreadExecutor();
        // Submit the listening task to the executor
        executorService.submit(this::listenForNotifications);
        logger.info("PostgreSQL Notification Listener service started.");
    }

    /**
     * The core logic for listening to notifications. This runs in a dedicated thread.
     * It handles connection establishment, listener registration, LISTEN command execution,
     * and reconnection attempts upon failures.
     */
    private void listenForNotifications() {
        int retryCount = 0;
        while (running) { // Keep running as long as the application is active
            try {
                // Check if the connection is null or closed, indicating a need to re-establish
                if (pgConnection == null || pgConnection.isClosed()) {
                    logger.info("Attempting to establish/re-establish connection for PostgreSQL notifications...");
                    notificationConnection = dataSource.getConnection(); // Get a connection from the pool
                    pgConnection = notificationConnection.unwrap(PGConnection.class); // Unwrap to pgjdbc-ng PGConnection

                    // Register the notification listener.
                    // This callback will be invoked asynchronously by pgjdbc-ng when a NOTIFY event arrives.
                    pgConnection.addNotificationListener((channel, payload) -> {
                        logger.info("🔔 Received notification from channel: {}", channel);
                        logger.info("📦 Payload: {}", payload);
                        // IMPORTANT: Any complex or long-running business logic should be
                        // dispatched to another thread pool to avoid blocking the
                        // notification listener thread.
                    });

                    // Execute the LISTEN command to subscribe to the desired channel
                    try (Statement stmt = pgConnection.createStatement()) {
                        stmt.execute("LISTEN my_channel");
                        logger.info("✅ Successfully subscribed to 'my_channel' for notifications.");
                    }
                    retryCount = 0; // Reset retry count on successful connection
                }

                // Since pgjdbc-ng's addNotificationListener handles asynchronous notification
                // delivery, this thread primarily keeps the dedicated connection alive.
                // A small sleep prevents busy-waiting.
                Thread.sleep(1000);

            } catch (SQLException e) {
                // Handle SQL exceptions (e.g., connection loss, database issues)
                logger.error("SQL Error in PostgreSQL notification listener (Attempt {}/{}) for channel 'my_channel': {}",
                        ++retryCount, MAX_RETRY_ATTEMPTS, e.getMessage(), e);
                closeConnection(); // Close the problematic connection immediately

                if (retryCount >= MAX_RETRY_ATTEMPTS) {
                    logger.error("Max retry attempts reached. Stopping PostgreSQL notification listener for 'my_channel'.");
                    running = false; // Stop the listener if max retries are hit
                    break;
                }
                try {
                    Thread.sleep(RETRY_DELAY_MS); // Wait before attempting to reconnect
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt(); // Restore interrupt status
                    logger.warn("Notification listener retry delay interrupted.", ie);
                    running = false; // Exit loop if interrupted during sleep
                    break;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); // Restore interrupt status
                logger.warn("PostgreSQL Notification Listener thread interrupted.", e);
                running = false; // Exit loop on interruption
            } catch (Exception e) {
                // Catch any other unexpected exceptions during setup or unwrapping
                logger.error("Unexpected error in PostgreSQL notification listener: {}", e.getMessage(), e);
                running = false;
            }
        }
        logger.info("PostgreSQL Notification Listener background thread terminated.");
    }

    /**
     * Closes the dedicated notification connection if it's open.
     */
    private void closeConnection() {
        if (notificationConnection != null) {
            try {
                if (!notificationConnection.isClosed()) {
                    // Close the connection. Since it's obtained from a pool,
                    // this returns it to the pool (it's not physically closed).
                    // However, for a dedicated connection like this, the pool might
                    // eventually clean it up if it's idle or broken.
                    notificationConnection.close();
                    logger.info("PostgreSQL notification connection closed (returned to pool).");
                }
            } catch (SQLException e) {
                logger.error("Error closing notification connection: " + e.getMessage(), e);
            } finally {
                notificationConnection = null; // Clear references
                pgConnection = null;
            }
        }
    }

    /**
     * This method is called by Spring when the application is shutting down.
     * It ensures a graceful shutdown of the listener thread and connection.
     */
    @PreDestroy
    public void destroy() {
        logger.info("Initiating shutdown of PostgreSQL Notification Listener Service...");
        running = false; // Signal the listening loop to stop

        // Shut down the executor service and wait for its tasks to terminate
        if (executorService != null) {
            executorService.shutdown(); // Disable new tasks from being submitted
            try {
                // Wait for existing tasks to terminate or timeout
                if (!executorService.awaitTermination(10, TimeUnit.SECONDS)) {
                    executorService.shutdownNow(); // Forcefully shutdown if not terminated
                    logger.warn("Forced shutdown of PostgreSQL Notification Listener executor service.");
                    // Optionally, wait a bit longer for forced shutdown
                    if (!executorService.awaitTermination(5, TimeUnit.SECONDS)) {
                        logger.error("Executor service did not terminate.");
                    }
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); // Restore interrupt status
                executorService.shutdownNow(); // Interrupt and shutdown immediately
                logger.warn("Interrupted while waiting for listener thread to terminate.");
            }
        }

        closeConnection(); // Ensure the dedicated connection is closed
        logger.info("PostgreSQL Notification Listener Service shutdown complete.");
    }
}
```properties
# src/main/resources/application.properties

# Database connection properties (standard Spring Boot properties)
spring.datasource.url=jdbc:postgresql://localhost:5432/your_database_name
spring.datasource.username=your_username
spring.datasource.password=your_password
spring.datasource.driver-class-name=com.impossibl.postgres.jdbc.PGDriver # Specify pgjdbc-ng driver

# You can optionally configure connection pooling properties here as well, e.g.,
# spring.datasource.hikari.maximum-pool-size=10
# spring.datasource.hikari.minimum-idle=2
```xml
