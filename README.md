# UPI Offline Mesh — Secure Deferred-Settlement Demo

A Java 17 / Spring Boot 3.3 backend and software-simulated mesh demonstrating how an offline payment instruction can be carried through untrusted intermediary devices and later uploaded by an internet-connected bridge. The system combines hybrid RSA-OAEP + AES-256-GCM encryption, SHA-256 ciphertext hashing, concurrent idempotency, freshness validation, transactional settlement, and a simulated gossip network. It includes an interactive dashboard so the complete flow can be demonstrated locally without real Bluetooth hardware.

> **Project scope:** This is a software demo of mesh-routed deferred settlement. The mesh is simulated; it is not a production UPI or real-time offline payment system.

## Key Features

- **Offline payment packet model** using `PaymentInstruction` and `MeshPacket`.
- **Hybrid encryption** using RSA-2048/RSA-OAEP to protect a per-packet AES-256 key and AES-256-GCM for authenticated payload encryption.
- **Tamper detection** through AES-GCM authentication; modified ciphertext is rejected during decryption.
- **Flow through untrusted intermediaries** where intermediary devices carry opaque ciphertext rather than the payment contents.
- **Gossip-based mesh simulation** across virtual devices with a configurable packet TTL.
- **Internet bridge simulation** where a device with internet connectivity uploads held packets to the backend.
- **Concurrent idempotency** using an atomic `ConcurrentHashMap.putIfAbsent` claim on the SHA-256 ciphertext hash.
- **Replay protection** through a 24-hour `signedAt` freshness check and per-payment nonce.
- **Transactional settlement** that debits the sender, credits the receiver, and records a ledger transaction in one database transaction.
- **Optimistic locking** on account records using JPA `@Version`.
- **Interactive dashboard** for payment injection, gossip rounds, bridge flushing, mesh reset, account balances, and transaction history.
- **Concurrency and security tests** covering encryption/decryption, tampering, and simultaneous duplicate delivery.

## Architecture

```text
 Offline Sender
      |
      | PaymentInstruction
      | RSA-OAEP + AES-256-GCM
      v
  MeshPacket
      |
      | simulated Bluetooth gossip
      v
+-------------+    +-------------+    +-------------+
| Virtual     | -> | Virtual     | -> | Bridge Node |
| Device      |    | Device      |    | hasInternet |
+-------------+    +-------------+    +------+------+
                                             |
                                             | HTTPS POST
                                             v
                              +-----------------------------+
                              | Spring Boot Backend         |
                              |                             |
                              | SHA-256 ciphertext hash     |
                              |            |                |
                              |            v                |
                              | Idempotency claim           |
                              |            |                |
                              |            v                |
                              | RSA-OAEP + AES-GCM decrypt  |
                              |            |                |
                              |            v                |
                              | Freshness validation        |
                              |            |                |
                              |            v                |
                              | Transactional settlement    |
                              +-------------+---------------+
                                            |
                              +-------------+-------------+
                              |                           |
                              v                           v
                       Account Balances             Transaction Ledger
```

## Processing Pipeline

The bridge ingestion path is deliberately ordered so duplicate packets can be rejected before decryption and settlement:

```text
Bridge Packet
     |
     v
SHA-256(ciphertext)
     |
     v
Atomic idempotency claim
     |
     +---- duplicate ----> DUPLICATE_DROPPED
     |
     v
RSA-OAEP unwrap AES key
     |
     v
AES-256-GCM decrypt + authentication
     |
     +---- tampered -----> INVALID
     |
     v
Check signedAt freshness
     |
     +---- stale --------> INVALID
     |
     v
@Transactional settlement
     |
     +--> debit sender
     +--> credit receiver
     +--> write ledger
     |
     v
SETTLED
```

## Security and Concurrency Design

### Hybrid Encryption

The encrypted payload follows the hybrid construction documented by the project:

1. Generate a fresh AES-256 key for the packet.
2. Encrypt the payment JSON using AES-256-GCM.
3. Encrypt the AES key using RSA-OAEP and the server's RSA public key.
4. Store the RSA-encrypted key, IV, and authenticated ciphertext together.

The documented layout is:

```text
[256-byte RSA-encrypted AES key]
[12-byte IV]
[AES ciphertext + 16-byte GCM authentication tag]
```

Intermediary devices therefore carry encrypted packet contents. AES-GCM authentication causes tampered ciphertext to fail verification during decryption.

### Concurrent Idempotency

The backend computes `SHA-256(ciphertext)` before decryption and attempts an atomic claim:

```java
Instant previous = seen.putIfAbsent(packetHash, now);
return previous == null;
```

Only the first concurrent claimant proceeds through decryption and settlement. Duplicate deliveries return `DUPLICATE_DROPPED` before the settlement path.

The transaction table also has a unique index on `packetHash` as a database-level defense-in-depth mechanism.

### Replay Protection

The encrypted payment contains:

- `signedAt` — the backend rejects packets older than 24 hours.
- `nonce` — distinguishes separate legitimate payments even when sender, receiver, and amount are the same.

A replay of the same authenticated packet retains the same ciphertext hash and is therefore caught by idempotency.

### Transactional Settlement

`SettlementService` performs debit, credit, and ledger insertion inside a `@Transactional` operation. `Account` uses JPA `@Version` optimistic locking as an additional concurrency safeguard.

## Technologies

| Area | Technology |
|---|---|
| Language | Java 17 |
| Backend | Spring Boot 3.3 |
| Build | Maven Wrapper |
| Persistence | Spring Data JPA + H2 in-memory database |
| Cryptography | RSA-2048 / RSA-OAEP + AES-256-GCM |
| Hashing | SHA-256 |
| Concurrency | `ConcurrentHashMap.putIfAbsent`, Java threads |
| Web API | Spring REST endpoints |
| UI | Spring-served dashboard / HTML template |
| Mesh | Software-simulated gossip network |

## Project Structure

```text
upi-offline-mesh/
├── pom.xml
├── mvnw
├── mvnw.cmd
├── README.md
│
├── src/main/
│   ├── resources/
│   │   ├── application.properties
│   │   └── templates/
│   │       └── dashboard.html
│   │
│   └── java/com/demo/upimesh/
│       ├── UpiMeshApplication.java
│       │
│       ├── model/
│       │   ├── Account.java
│       │   ├── AccountRepository.java
│       │   ├── Transaction.java
│       │   ├── TransactionRepository.java
│       │   ├── MeshPacket.java
│       │   └── PaymentInstruction.java
│       │
│       ├── crypto/
│       │   ├── ServerKeyHolder.java
│       │   └── HybridCryptoService.java
│       │
│       ├── service/
│       │   ├── DemoService.java
│       │   ├── VirtualDevice.java
│       │   ├── MeshSimulatorService.java
│       │   ├── IdempotencyService.java
│       │   ├── SettlementService.java
│       │   └── BridgeIngestionService.java
│       │
│       ├── controller/
│       │   ├── ApiController.java
│       │   └── DashboardController.java
│       │
│       └── config/
│           └── AppConfig.java
│
└── src/test/java/com/demo/upimesh/
    └── IdempotencyConcurrencyTest.java
```

## Build and Run

### Prerequisites

- JDK 17 or newer
- Java available on `PATH`, or `JAVA_HOME` configured
- No separate Maven installation required; the repository includes the Maven wrapper.

### Windows

From the project directory:

```powershell
mvnw.cmd spring-boot:run
```

If using PowerShell and the command is not recognized, use:

```powershell
.\mvnw.cmd spring-boot:run
```

### macOS / Linux

```bash
./mvnw spring-boot:run
```

The application starts on port `8080` by default.

Open:

```text
http://localhost:8080
```

Stop the application with `Ctrl+C`.

## Demo Workflow

### 1. Inject a Payment

Use the dashboard to select sender, receiver, amount, and PIN, then inject the payment into the simulated mesh.

The backend creates a `PaymentInstruction`, encrypts it, wraps it in a `MeshPacket`, and places it on the virtual sender device.

### 2. Run Gossip Rounds

Run the gossip action to propagate packets between virtual devices. Each hop decrements the packet TTL.

The simulator treats every virtual device as being within the simulated Bluetooth range.

### 3. Flush Bridge Nodes

The default `phone-bridge` device is configured as the internet-connected bridge. It uploads held packets to:

```text
POST /api/bridge/ingest
```

The backend then executes the hash → idempotency → decrypt → freshness → settlement pipeline.

### 4. Exercise Concurrent Idempotency

The repository includes a test that delivers the same packet concurrently from three threads and verifies that only one settlement succeeds.

Run it with:

```powershell
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

## API Reference

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/` | Serve dashboard |
| GET | `/api/server-key` | Return server RSA public key |
| GET | `/api/accounts` | Return account balances |
| GET | `/api/transactions` | Return recent transactions |
| GET | `/api/mesh/state` | Return virtual-device state |
| POST | `/api/demo/send` | Simulate sender encryption and packet injection |
| POST | `/api/mesh/gossip` | Run one mesh gossip round |
| POST | `/api/mesh/flush` | Upload packets from internet-connected bridges |
| POST | `/api/mesh/reset` | Reset mesh and idempotency state |
| POST | `/api/bridge/ingest` | Ingest a bridge-delivered payment packet |
| GET | `/h2-console` | Browse the H2 in-memory database |

### Bridge Ingestion Request

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3
```

```json
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-RSA-and-AES-blob"
}
```

Possible outcomes include:

```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

The documented outcomes are `SETTLED`, `DUPLICATE_DROPPED`, and `INVALID`.

## Tests

Run the complete test suite:

```powershell
mvnw.cmd test
```

The documented tests cover:

- `encryptDecryptRoundTrip` — verifies the hybrid encryption/decryption round trip.
- `tamperedCiphertextIsRejected` — modifies ciphertext and verifies the bridge ingestion path returns `INVALID` rather than settling.
- `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` — concurrently submits one packet from three threads and verifies one settlement, two duplicate drops, and a single sender debit.

## Example Concurrency Result

The repository's headline concurrency test models three simultaneous bridge deliveries of the same packet:

```text
3 concurrent deliveries
        |
        v
+-----------------------+
| Idempotency claim     |
| SHA-256 ciphertext    |
+-----------------------+
        |
   +----+----+----+
   |         |    |
   v         v    v
SETTLED   DUPLICATE  DUPLICATE
          DROPPED    DROPPED
```

The expected test assertions are exactly one `SETTLED` result, two `DUPLICATE_DROPPED` results, and one corresponding sender debit.

## Limitations

This project explicitly documents several limitations of the concept and implementation:

- **Simulated mesh:** the Bluetooth-style network is software-simulated; no real BLE hardware is used.
- **Deferred settlement:** the receiver cannot independently verify the sender's available funds while completely offline.
- **Offline double-spend risk:** multiple offline payment instructions can be created before any one reaches the backend; settlement order determines which reaches the ledger first.
- **Demo persistence:** the application uses an H2 in-memory database and seeds accounts on startup.
- **Local idempotency state:** `ConcurrentHashMap` is JVM-local and would need a distributed store such as Redis for multi-instance deployment.
- **Ephemeral server keys:** the RSA keypair is regenerated on startup rather than stored in an HSM/KMS.
- **Bridge authentication:** the demo does not authenticate `/api/bridge/ingest` with mutual TLS or signed bridge certificates.
- **No production rate limiting:** per-bridge and per-sender velocity controls are not implemented.
- **No real banking integration:** settlement is local to the demo ledger rather than connected to NPCI or a bank core.

## Production Evolution

The original project identifies these infrastructure changes for a production deployment:

| Demo Component | Production Direction |
|---|---|
| H2 in-memory DB | PostgreSQL / MySQL with replicas |
| `ConcurrentHashMap` idempotency | Redis `SET NX EX` |
| Startup-generated RSA keypair | HSM / KMS-backed private key |
| Server-side simulated sender | Android/Kotlin client implementation |
| Software mesh simulator | Real BLE GATT or Wi-Fi Direct |
| Local settlement service | Integration with a bank/NPCI core |
| No bridge authentication | Mutual TLS or signed bridge certificates |
| Seeded in-memory accounts | Real authenticated/KYC account system |
| No rate limiting | Per-bridge and per-sender rate controls |
| Console logs | Structured logging and monitoring |

These are proposed production directions, not capabilities of the current demo.

## Engineering Takeaways

This project demonstrates several software-engineering concepts in one end-to-end workflow:

- **Secure data transport:** hybrid public-key/symmetric encryption with authenticated encryption.
- **Concurrency control:** atomic idempotency claims prevent duplicate settlement under simultaneous requests.
- **Stateful processing:** payment packets retain identity, TTL, nonce, timestamp, and encrypted payload state while moving through the simulated mesh.
- **Transactional consistency:** debit, credit, and ledger insertion are grouped into one transaction.
- **Optimistic concurrency:** JPA `@Version` protects account updates from conflicting writes.
- **Layered architecture:** model, crypto, service, controller, and configuration layers separate responsibilities.
- **API-driven backend:** REST endpoints expose the demo pipeline and state to the dashboard.
- **Testable security/concurrency behavior:** the repository includes dedicated tests for tampering, encryption, and concurrent duplicate delivery.

## License

The upstream repository states that the demo code has no license and is intended for learning. If redistributing or modifying the project, preserve appropriate attribution and verify the applicable permissions for the source repository.
