import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.jdbc.DataSourceProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;
import com.impossibl.postgres.jdbc.PGConnection;

import javax.sql.DataSource;
import java.sql.DriverManager; // Import DriverManager
import java.sql.SQLException;

@Configuration
public class PgJdbcNgDataSourceConfig {

    // (Optional) If you still want a pooled DataSource for pgjdbc-ng for other operations, keep these:
    @Bean
    @ConfigurationProperties("pgjdbc-ng.datasource")
    public DataSourceProperties pgJdbcNgDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean(name = "pgJdbcNgDataSource")
    public DataSource pgJdbcNgDataSource(@Qualifier("pgJdbcNgDataSourceProperties") DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }

    @Bean(name = "pgJdbcNgTemplate")
    public JdbcTemplate pgJdbcNgTemplate(@Qualifier("pgJdbcNgDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    // --- Dedicated connection for LISTEN ---
    @Bean(name = "pgJdbcNgListenConnection")
    public PGConnection pgJdbcNgListenConnection(
        @Qualifier("pgJdbcNgDataSourceProperties") DataSourceProperties pgJdbcNgDataSourceProperties // Use properties directly
    ) throws SQLException {
        // Construct the JDBC URL from the properties
        String url = pgJdbcNgDataSourceProperties.getUrl();
        String username = pgJdbcNgDataSourceProperties.getUsername();
        String password = pgJdbcNgDataSourceProperties.getPassword();

        // Get a direct, non-pooled connection using DriverManager
        // Make sure the driver is registered (Spring Boot typically handles this)
        return (PGConnection) DriverManager.getConnection(url, username, password);
    }
}
