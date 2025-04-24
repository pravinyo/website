---
title: "Part 2: Database Engineering Fundamentals: Sharding with Postgres"
author: pravin_tripathi
date: 2024-07-02 02:00:00 +0530
readtime: true
media_subpath: /assets/img/database-engineering-fundamental-part-2/
categories: [Blogging, Article]
mermaid: true
permalink: /database-engineering-fundamental-part-2/sharding-with-postgres/
parent: /database-engineering-fundamental-part-2/
tags: [softwareengineering, backenddevelopment,database-engineering]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Copilot
---
# Sharding with Postgres

## Simple Java Implementation for Database Sharding: URL Shortener Example

### Step 1: Table Creation

First, we'll create a simple table for our URL shortener on each shard. Here's the SQL command:

```sql
CREATE TABLE url_shortener (
    id SERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Step 2: Setting up Docker for Multiple Shards

We'll use Docker to set up three PostgreSQL instances as our shards. Create a docker-compose.yml file with the following content:

```yaml
version: '3'
services:
  shard1:
    image: postgres:13
    environment:
      POSTGRES_DB: urlshortener
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"

  shard2:
    image: postgres:13
    environment:
      POSTGRES_DB: urlshortener
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5433:5432"

  shard3:
    image: postgres:13
    environment:
      POSTGRES_DB: urlshortener
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5434:5432"
```

Run the following command to start the Docker containers:

```bash
docker-compose up -d
```

### Step 3: Java Implementation with Consistent Hashing

Now, let's implement a simple Java application that uses consistent hashing for sharding. We'll use the java-consistent-hash library for consistent hashing.

First, add the following dependencies to your pom.xml:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.3.1</version>
</dependency>
<dependency>
    <groupId>com.github.ssedano</groupId>
    <artifactId>java-consistent-hash</artifactId>
    <version>1.0.0</version>
</dependency>
```

Here's the Java code for our sharded URL shortener:

```java
import com.github.ssedano.hash.ConsistentHash;
import java.sql.*;
import java.util.*;

public class ShardedUrlShortener {
    private final ConsistentHash<String> consistentHash;
    private final Map<String, String> shardConnections;

    public ShardedUrlShortener() {
        List<String> shards = Arrays.asList("shard1", "shard2", "shard3");
        consistentHash = new ConsistentHash<>(shards, 10);
        
        shardConnections = new HashMap<>();
        shardConnections.put("shard1", "jdbc:postgresql://localhost:5432/urlshortener");
        shardConnections.put("shard2", "jdbc:postgresql://localhost:5433/urlshortener");
        shardConnections.put("shard3", "jdbc:postgresql://localhost:5434/urlshortener");
    }

    public void insertUrl(String shortCode, String longUrl) throws SQLException {
        String shard = consistentHash.get(shortCode);
        String connectionUrl = shardConnections.get(shard);

        try (Connection conn = DriverManager.getConnection(connectionUrl, "user", "password");
             PreparedStatement pstmt = conn.prepareStatement("INSERT INTO url_shortener (short_code, long_url) VALUES (?, ?)")) {
            pstmt.setString(1, shortCode);
            pstmt.setString(2, longUrl);
            pstmt.executeUpdate();
        }
    }

    public String getLongUrl(String shortCode) throws SQLException {
        String shard = consistentHash.get(shortCode);
        String connectionUrl = shardConnections.get(shard);

        try (Connection conn = DriverManager.getConnection(connectionUrl, "user", "password");
             PreparedStatement pstmt = conn.prepareStatement("SELECT long_url FROM url_shortener WHERE short_code = ?")) {
            pstmt.setString(1, shortCode);
            try (ResultSet rs = pstmt.executeQuery()) {
                if (rs.next()) {
                    return rs.getString("long_url");
                }
            }
        }
        return null;
    }

    public static void main(String[] args) {
        ShardedUrlShortener shortener = new ShardedUrlShortener();

        try {
            // Insert a URL
            shortener.insertUrl("abc123", "https://www.example.com");
            System.out.println("URL inserted successfully");

            // Retrieve the long URL
            String longUrl = shortener.getLongUrl("abc123");
            System.out.println("Retrieved long URL: " + longUrl);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

### Explanation of the Implementation

1. We create a ConsistentHash object with our three shards and a replication factor of 10.
2. The shardConnections map stores the JDBC connection URLs for each shard.
3. The insertUrl method uses consistent hashing to determine which shard to use based on the short code, then inserts the URL into the appropriate shard.
4. The getLongUrl method similarly uses consistent hashing to determine which shard to query for a given short code.
5. In the main method, we demonstrate inserting a URL and then retrieving it.

This implementation provides a simple example of database sharding using consistent hashing. In a production environment, you'd want to add more error handling, connection pooling, and possibly a caching layer for improved performance.

[Back to Parent Page]({{ page.parent }})