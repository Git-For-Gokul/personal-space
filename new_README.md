// src/main/java/com/example/pgeventlisten/PgEventListenApplication.java
package com.example.pgeventlisten;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Main Spring Boot application class.
 * This class uses @SpringBootApplication to enable auto-configuration
 * and component scanning.
 */
@SpringBootApplication
public class PgEventListenApplication {

    public static void main(String[] args) {
        // Run the Spring Boot application.
        // This will initialize the Spring context and start all components,
        // including our PgNotificationListenerService.
        SpringApplication.run(PgEventListenApplication.class, args);
    }

}
```java
// src/main/java/com/example/pgeventlisten/PgNotificationListenerService.java
package com.example.pgeventlisten;

import org.postgresql.PGConnection; // Import from the standard driver
import org.postgresql.PGNotification; // Import from the standard driver
import jakarta.annotation.PreDestroy;
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
 * to listen for NOTIFY events using the standard org.postgresql:postgresql driver
 * via a polling mechanism (getNotifications()).
 * When a notification arrives, a simulated "complex task" is dispatched to a
 * separate thread pool to avoid blocking the notification polling thread.
 */
@Service
public class PgNotificationListenerService implements ApplicationRunner {

    private static final Logger logger = LoggerFactory.getLogger(PgNotificationListenerService.class);

    private final DataSource dataSource; // Spring will inject the configured DataSource
    private Connection notificationConnection; // Dedicated standard JDBC Connection
    private PGConnection pgConnection; // Unwrapped standard PGConnection
    private ExecutorService notificationPollingExecutor; // Executor for the notification polling thread
    private ExecutorService complexTaskExecutor; // New Executor for dispatching complex tasks
    private volatile boolean running = true; // Flag for graceful thread termination

    private static final int MAX_RETRY_ATTEMPTS = 5; // Maximum connection retry attempts
    private static final long RETRY_DELAY_MS = 5000; // Delay between retries in milliseconds
    private static final int NOTIFICATION_POLL_TIMEOUT_MS = 1000; // Timeout for getNotifications()
    private static final int COMPLEX_TASK_POOL_SIZE = 5; // Size of the thread pool for complex tasks
    private static final long COMPLEX_TASK_DURATION_MS = 3000; // Simulated duration of complex task

    /**
     * Constructor for dependency injection. Spring automatically injects the DataSource.
     * @param dataSource The configured JDBC DataSource.
     */
    public PgNotificationListenerService(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    /**
     * This method is called by Spring Boot after the application context is fully loaded.
     * It starts the dedicated thread for listening to PostgreSQL notifications and initializes
     * the thread pool for complex tasks.
     * @param args Application arguments.
     */
    @Override
    public void run(ApplicationArguments args) {
        notificationPollingExecutor = Executors.newSingleThreadExecutor();
        notificationPollingExecutor.submit(this::listenForNotifications);
        logger.info("PostgreSQL Notification Listener service (Standard Driver) started.");

        complexTaskExecutor = Executors.newFixedThreadPool(COMPLEX_TASK_POOL_SIZE);
        logger.info("Complex task executor initialized with {} threads.", COMPLEX_TASK_POOL_SIZE);
    }

    /**
     * The core logic for listening to notifications using polling.
     * This runs in a dedicated thread.
     */
    private void listenForNotifications() {
        int retryCount = 0;
        while (running) {
            try {
                // Establish connection if not already established or if it was closed
                if (pgConnection == null || pgConnection.isClosed()) {
                    logger.info("Attempting to establish/re-establish connection for PostgreSQL notifications...");
                    notificationConnection = dataSource.getConnection(); // Get a new connection
                    pgConnection = notificationConnection.unwrap(PGConnection.class); // Unwrap to standard PGConnection

                    // Execute the LISTEN command to subscribe to the desired channel
                    try (Statement stmt = pgConnection.createStatement()) {
                        stmt.execute("LISTEN my_channel");
                        logger.info("✅ Successfully subscribed to 'my_channel' for notifications.");
                    }
                    retryCount = 0; // Reset retry count on successful connection
                }

                // Poll for notifications with a timeout
                PGNotification[] notifications = pgConnection.getNotifications(NOTIFICATION_POLL_TIMEOUT_MS);

                if (notifications != null && notifications.length > 0) {
                    for (PGNotification notification : notifications) {
                        logger.info("🔔 Received notification from channel: {}", notification.getChannel());
                        logger.info("📌 PID: {}", notification.getPID());
                        logger.info("📦 Payload: {}", notification.getParameter());

                        // Dispatch the complex task to a separate thread pool
                        final String channel = notification.getChannel();
                        final String payload = notification.getParameter();
                        complexTaskExecutor.submit(() -> performComplexTask(channel, payload));
                        logger.info("✅ Dispatched complex task for notification (Channel: {}, Payload: {})", channel, payload);
                    }
                }

            } catch (SQLException e) {
                logger.error("SQL Error in PostgreSQL notification listener (Attempt {}/{}) for channel 'my_channel': {}",
                        ++retryCount, MAX_RETRY_ATTEMPTS, e.getMessage(), e);
                closeConnection(); // Close the problematic connection immediately

                if (retryCount >= MAX_RETRY_ATTEMPTS) {
                    logger.error("Max retry attempts reached. Stopping PostgreSQL notification listener for 'my_channel'.");
                    running = false;
                    break;
                }
                try {
                    Thread.sleep(RETRY_DELAY_MS); // Wait before attempting to reconnect
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    logger.warn("Notification listener retry delay interrupted.", ie);
                    running = false;
                    break;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                logger.warn("PostgreSQL Notification Listener thread interrupted.", e);
                running = false;
            } catch (Exception e) {
                logger.error("Unexpected error in PostgreSQL notification listener: {}", e.getMessage(), e);
                running = false;
            }
        }
        logger.info("PostgreSQL Notification Listener background thread terminated.");
    }

    /**
     * Simulates a complex, long-running task.
     * This method runs in a thread managed by the `complexTaskExecutor`.
     * @param channel The channel from which the notification originated.
     * @param payload The payload of the notification.
     */
    private void performComplexTask(String channel, String payload) {
        logger.info("⚙️ Starting complex task for Channel: {}, Payload: {}", channel, payload);
        try {
            Thread.sleep(COMPLEX_TASK_DURATION_MS); // Simulate work
            logger.info("✅ Complex task finished for Channel: {}, Payload: {}", channel, payload);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.warn("Complex task interrupted for Channel: {}, Payload: {}", channel, payload, e);
        } catch (Exception e) {
            logger.error("Error during complex task for Channel: {}, Payload: {}: {}", channel, payload, e.getMessage(), e);
        }
    }


    /**
     * Closes the dedicated notification connection if it's open.
     */
    private void closeConnection() {
        if (notificationConnection != null) {
            try {
                if (!notificationConnection.isClosed()) {
                    notificationConnection.close();
                    logger.info("PostgreSQL notification connection closed (returned to pool).");
                }
            } catch (SQLException e) {
                logger.error("Error closing notification connection: " + e.getMessage(), e);
            } finally {
                notificationConnection = null;
                pgConnection = null;
            }
        }
    }

    /**
     * This method is called by Spring when the application is shutting down.
     * It ensures a graceful shutdown of all executor services and connections.
     */
    @PreDestroy
    public void destroy() {
        logger.info("Initiating shutdown of PostgreSQL Notification Listener Service...");
        running = false; // Signal the listening loop to stop

        // Shutdown notification polling executor
        if (notificationPollingExecutor != null) {
            shutdownExecutor(notificationPollingExecutor, "Notification Polling Executor");
        }

        // Shutdown complex task executor
        if (complexTaskExecutor != null) {
            shutdownExecutor(complexTaskExecutor, "Complex Task Executor");
        }

        closeConnection();
        logger.info("PostgreSQL Notification Listener Service shutdown complete.");
    }

    /**
     * Helper method to gracefully shut down an ExecutorService.
     */
    private void shutdownExecutor(ExecutorService executor, String name) {
        executor.shutdown(); // Disable new tasks from being submitted
        try {
            // Wait for existing tasks to terminate or timeout
            if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                executor.shutdownNow(); // Forcefully shutdown if not terminated
                logger.warn("Forced shutdown of {} service.", name);
                // Optionally, wait a bit longer for forced shutdown
                if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                    logger.error("{} service did not terminate.", name);
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // Restore interrupt status
            executor.shutdownNow(); // Interrupt and shutdown immediately
            logger.warn("Interrupted while waiting for {} to terminate.", name);
        }
    }
}
```properties
# src/main/resources/application.properties

# Database connection properties (standard Spring Boot properties)
# Changed driver-class-name for the standard PostgreSQL JDBC driver.
# Added sslmode=require and ssl=true to enable SSL for the connection.
spring.datasource.url=jdbc:postgresql://localhost:5432/your_database_name?sslmode=require&ssl=true
spring.datasource.username=your_username
spring.datasource.password=your_password
spring.datasource.driver-class-name=org.postgresql.Driver # Specify the standard PostgreSQL driver

# You can optionally configure connection pooling properties here as well, e.g.,
# spring.datasource.hikari.maximum-pool-size=10
# spring.datasource.hikari.minimum-idle=2
```xml
<!-- pom.xml (add these dependencies to your Spring Boot project's pom.xml) -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0)"
         xmlns:xsi="[http://www.w3.org/2001/XMLSchema-instance](http://www.w3.org/2001/XMLSchema-instance)"
         xsi:schemaLocation="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0) [http://maven.apache.org/xsd/maven-4.0.0.xsd](http://maven.apache.org/xsd/maven-4.0.0.xsd)">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.7</version> <!-- Use a recent stable Spring Boot version -->
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>com.example</groupId>
    <artifactId>pg-event-listen</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>pg-event-listen</name>
    <description>Spring Boot application for PostgreSQL NOTIFY/LISTEN</description>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web Starter (includes Tomcat, Spring MVC, logging, etc.) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Boot JDBC Starter (includes DataSource auto-configuration) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>

        <!-- Standard PostgreSQL JDBC driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Jakarta Annotations for PostConstruct and PreDestroy (Spring Boot 3+) -->
        <dependency>
            <groupId>jakarta.annotation</groupId>
            <artifactId>jakarta.annotation-api</artifactId>
            <version>2.1.1</version> <!-- Consistent with Spring Boot 3+ -->
        </dependency>

        <!-- Spring Boot Test Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
