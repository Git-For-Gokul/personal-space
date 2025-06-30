import org.springframework.stereotype.Service;
import java.util.Map;
import java.util.concurrent.*;

@Service
public class NotificationProcessingService {

    private final Map<String, String> latestNotificationMap = new ConcurrentHashMap<>();
    private final Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();

    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);
    private final long debounceDelaySeconds = 5;

    public void scheduleDebouncedProcessing(String payload) {
        String key = extractKeyFromPayload(payload); // Extract counterparty ID

        latestNotificationMap.put(key, payload);

        // Cancel previously scheduled task if exists
        ScheduledFuture<?> existingTask = scheduledTasks.get(key);
        if (existingTask != null && !existingTask.isDone()) {
            existingTask.cancel(false); // Don't interrupt if running, just prevent execution
        }

        // Schedule new task
        ScheduledFuture<?> newTask = scheduler.schedule(() -> {
            String latestPayload = latestNotificationMap.remove(key);
            scheduledTasks.remove(key); // Clean up task tracking map
            if (latestPayload != null) {
                processNotification(latestPayload);
            }
        }, debounceDelaySeconds, TimeUnit.SECONDS);

        // Store new task
        scheduledTasks.put(key, newTask);
    }

    private void processNotification(String payload) {
        System.out.println("[Debounced] Processing latest notification for payload: " + payload);
        // TODO: Handle Kafka outbox update and superseding logic here.
    }

    private String extractKeyFromPayload(String payload) {
        // TODO: Extract counterparty ID or unique key from the payload
        return payload; // Simple passthrough for now
    }
}
