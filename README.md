import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.AsyncResult;
import org.springframework.stereotype.Service;

import javax.annotation.PreDestroy;
import java.util.Map;
import java.util.concurrent.*;

@Service
public class NotificationProcessingService {

    private final Map<String, String> latestNotificationMap = new ConcurrentHashMap<>();
    private final Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);
    private final long debounceDelaySeconds = 5;

    public void scheduleDebouncedProcessing(String payload) {
        String key = extractKeyFromPayload(payload);
        latestNotificationMap.put(key, payload);

        ScheduledFuture<?> existingTask = scheduledTasks.get(key);
        if (existingTask != null && !existingTask.isDone()) {
            existingTask.cancel(false);
        }

        ScheduledFuture<?> newTask = scheduler.schedule(() -> {
            String latestPayload = latestNotificationMap.remove(key);
            scheduledTasks.remove(key);
            if (latestPayload != null) {
                processNotificationAsync(latestPayload); // 🔁 Now async
            }
        }, debounceDelaySeconds, TimeUnit.SECONDS);

        scheduledTasks.put(key, newTask);
    }

    @Async("notificationExecutor")
    public void processNotificationAsync(String payload) {
        System.out.println("[Debounced] Async Processing on thread [" +
                Thread.currentThread().getName() + "]: " + payload);
        // Simulate heavy processing
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.err.println("Interrupted during async processing of: " + payload);
        }
        System.out.println("[Debounced] Finished processing: " + payload);
    }

    private String extractKeyFromPayload(String payload) {
        // Replace with actual extraction logic
        return payload; // simple passthrough if payload is the key
    }

    @PreDestroy
    public void shutdownExecutor() {
        System.out.println("Shutting down NotificationProcessingService...");
        scheduledTasks.values().forEach(future -> future.cancel(false));
        scheduledTasks.clear();
        latestNotificationMap.clear();

        scheduler.shutdown();
        try {
            if (!scheduler.awaitTermination(5, TimeUnit.SECONDS)) {
                scheduler.shutdownNow();
            }
        } catch (InterruptedException e) {
            scheduler.shutdownNow();
            Thread.currentThread().interrupt();
        }
        System.out.println("Executor shutdown complete.");
    }
}
