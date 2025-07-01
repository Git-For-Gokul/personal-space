package com.example.repository;

import com.example.model.KafkaOutboxEntry;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface KafkaOutboxRepository extends JpaRepository<KafkaOutboxEntry, Long> {

    // Find all entries by counterpartyId and status
    List<KafkaOutboxEntry> findByCounterpartyIdAndStatus(String counterpartyId, String status);

    // Find all pending entries (used during service startup to resume unprocessed messages)
    List<KafkaOutboxEntry> findByStatus(String status);

    // Optional: find latest version by counterparty
    @Query("SELECT k FROM KafkaOutboxEntry k WHERE k.counterpartyId = :counterpartyId AND k.status = 'pending' ORDER BY k.version DESC")
    List<KafkaOutboxEntry> findLatestPendingByCounterparty(String counterpartyId);

    // Mark older versions as superseded
    @Modifying
    @Query("UPDATE KafkaOutboxEntry k SET k.status = 'superseded' WHERE k.counterpartyId = :counterpartyId AND k.status = 'pending' AND k.id <> :readyId")
    int markOlderAsSuperseded(String counterpartyId, Long readyId);

    // Optional: bulk mark entries as ready
    @Modifying
    @Query("UPDATE KafkaOutboxEntry k SET k.status = 'ready' WHERE k.id = :id")
    int markAsReady(Long id);
}


@Async
public void processNotificationAsync(String payload) {
    String counterpartyId = extractKeyFromPayload(payload);

    List<KafkaOutboxEntry> entries = kafkaOutboxRepository.findByCounterpartyIdAndStatus(counterpartyId, "pending");

    if (entries.isEmpty()) return;

    KafkaOutboxEntry latest = entries.stream()
        .max(Comparator.comparing(e -> e.getVersion()))
        .orElseThrow();

    // Mark all others as superseded
    entries.stream()
        .filter(e -> !e.getId().equals(latest.getId()))
        .forEach(e -> {
            e.setStatus("superseded");
            kafkaOutboxRepository.save(e);
        });

    // Round version and update both tables
    BigDecimal roundedVersion = BigDecimal.valueOf(latest.getVersion().intValue() + 1);
    latest.setVersion(roundedVersion);
    latest.setStatus("ready");
    kafkaOutboxRepository.save(latest);

    counterpartyRepository.updateVersion(counterpartyId, roundedVersion);

    // Send to Kafka
    kafkaTemplate.send("counterparty-updates", counterpartyId, latest.getPayload().toString());

    // Optionally mark as published
    latest.setStatus("published");
    kafkaOutboxRepository.save(latest);

    System.out.println("Published to Kafka: " + counterpartyId + " @ v" + roundedVersion);
}
