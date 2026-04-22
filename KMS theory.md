# 🔐 Key Management System (KMS) — Complete Study Notes

> **KMS** = **K**ey **M**anagement **S**ystem  
> Used in: Cryptography · DevOps · Cloud Security · Authentication

---

## 📌 Table of Contents

1. [Why Do We Need KMS?](#1-why-do-we-need-kms)
2. [The Core Problem — Where to Store Keys?](#2-the-core-problem--where-to-store-keys)
3. [How KMS Works — Basic Flow](#3-how-kms-works--basic-flow)
4. [Envelope Encryption](#4-envelope-encryption)
5. [Blast Radius — Limiting Damage](#5-blast-radius--limiting-damage)
6. [Master Key Rotation](#6-master-key-rotation)
7. [Server Authentication with KMS](#7-server-authentication-with-kms)
8. [AWS KMS — Real World Example](#8-aws-kms--real-world-example)
9. [Key Concepts Cheat Sheet](#9-key-concepts-cheat-sheet)

---

## 1. Why Do We Need KMS?

### The Problem — Storing Sensitive Data

You have user credit card numbers. You **cannot** store them as plain text in your database.

> **If your database leaks → all card numbers are instantly exposed to hackers.**

### The Fix — Encryption

Store card numbers in **encrypted form** only. But encryption requires a **KEY**.

```mermaid
flowchart LR
    A["💳 Card Number\n4111-1111-1111-1111"] -->|Encrypt with KEY| B["🔒 Encrypted\naX9#mK2$pL8@vN5"]
    B -->|Store in DB| C[("🗄️ Database")]
    style A fill:#fff3cd,stroke:#ffc107,color:#000
    style B fill:#d4edda,stroke:#28a745,color:#000
    style C fill:#cce5ff,stroke:#004085,color:#000
```

**New problem:** Where do you store the KEY?

---

## 2. The Core Problem — Where to Store Keys?

### All the Bad Options

```mermaid
flowchart TD
    Q["❓ Where to store the KEY?"]
    
    Q --> A["🗄️ In the Database"]
    Q --> B["💻 Hard-coded in Source Code"]
    Q --> C["📄 In .env file on Server"]
    Q --> D["💾 On Server Disk"]
    
    A --> A1["❌ DB hack = Key stolen\n= All data decrypted"]
    B --> B1["❌ Code repo access\n= Key exposed\n= Hard to rotate"]
    C --> C1["❌ Server compromise\n= Key stolen"]
    D --> D1["❌ Server compromise\n= Key stolen"]

    style Q fill:#e2e3e5,stroke:#6c757d,color:#000
    style A fill:#f8d7da,stroke:#dc3545,color:#000
    style B fill:#f8d7da,stroke:#dc3545,color:#000
    style C fill:#f8d7da,stroke:#dc3545,color:#000
    style D fill:#f8d7da,stroke:#dc3545,color:#000
    style A1 fill:#f8d7da,stroke:#dc3545,color:#000
    style B1 fill:#f8d7da,stroke:#dc3545,color:#000
    style C1 fill:#f8d7da,stroke:#dc3545,color:#000
    style D1 fill:#f8d7da,stroke:#dc3545,color:#000
```

> **Core Insight:** The key is the most sensitive piece of data. If it leaks — ALL encrypted data is compromised instantly.

### The Comparison Table

| Storage Location | Safe? | Risk |
|---|---|---|
| Same Database | ❌ | DB leak = key stolen |
| Hard-coded in code | ❌ | Code leak = key stolen, hard to rotate |
| `.env` / disk | ❌ | Server compromise = key stolen |
| **Dedicated KMS** | ✅ | Best option |

---

## 3. How KMS Works — Basic Flow

Instead of storing the key yourself, all encrypt/decrypt work is **delegated to KMS**.

### Encryption Flow

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant S as 🖥️ Your Server
    participant K as 🔐 KMS
    participant D as 🗄️ Database

    U->>S: Sends card number "4111-1111-1111-1111"
    S->>K: "Please encrypt this message"
    K->>K: Encrypts using Master Key
    K->>S: Returns encrypted data "aX9#mK2$..."
    S->>D: Stores encrypted data
    Note over S: Server never sees or stores the KEY!
```

### Decryption Flow

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant S as 🖥️ Your Server
    participant K as 🔐 KMS
    participant D as 🗄️ Database

    U->>S: "Show me my card number"
    S->>D: Fetch encrypted data "aX9#mK2$..."
    D->>S: Returns encrypted data
    S->>K: "Please decrypt this"
    K->>K: Decrypts using Master Key
    K->>S: Returns "4111-1111-1111-1111"
    S->>U: Sends original card number
    Note over K: Key never leaves KMS!
```

> ✅ **Why is this safe?** Your server never holds the key. Even if DB + Server are hacked, the attacker only gets encrypted data — useless without the KMS key.

---

## 4. Envelope Encryption

### The Scalability Problem

If you have **1 million messages**, you can't send all of them to KMS one by one — **too slow, too much load**.

```mermaid
flowchart LR
    M["📨 1 Million Messages"] -->|Send each to KMS?| K["🔐 KMS"]
    K --> P["😵 KMS Overloaded!\nExtremely Slow!"]
    style M fill:#fff3cd,stroke:#ffc107,color:#000
    style K fill:#f8d7da,stroke:#dc3545,color:#000
    style P fill:#f8d7da,stroke:#dc3545,color:#000
```

### The Solution — Envelope Encryption

KMS gives you a **Data Encryption Key (DEK)**. You encrypt data locally using DEK, then store the **encrypted DEK** (not the plain DEK).

```mermaid
flowchart TD
    S["🖥️ Your Server"] -->|"1. Request a key"| K["🔐 KMS"]
    K -->|"2. Returns K1 (plain DEK)\n+ E(K1) (DEK encrypted by Master Key)"| S
    S -->|"3. Encrypt 1M messages locally using K1\n(Fast! No KMS load)"| D["🗄️ Database"]
    S -->|"4. Store E(K1) alongside encrypted messages"| D
    S -->|"5. 🗑️ DESTROY K1 immediately!"| X["❌ K1 deleted from memory"]

    style K fill:#d4edda,stroke:#28a745,color:#000
    style S fill:#cce5ff,stroke:#004085,color:#000
    style D fill:#e2e3e5,stroke:#6c757d,color:#000
    style X fill:#f8d7da,stroke:#dc3545,color:#000
```

### What Gets Stored in the Database

| Column | Value | Description |
|---|---|---|
| `message_id` | `msg_001` | Identifier |
| `encrypted_message` | `xK9#mP2...` | Data encrypted with K1 |
| `encrypted_key` | `E(K1)` | K1 encrypted by Master Key ✅ Safe to store |

> ⚠️ **Never store plain K1!** Always destroy it after use. Only `E(K1)` goes to the DB.

### Decryption Using Envelope Encryption

```mermaid
sequenceDiagram
    participant S as 🖥️ Server
    participant D as 🗄️ Database
    participant K as 🔐 KMS

    S->>D: Fetch encrypted_message + E(K1)
    D->>S: Returns encrypted_message + E(K1)
    S->>K: Send E(K1) — "Please decrypt this key"
    K->>K: Decrypts E(K1) using Master Key
    K->>S: Returns plain K1
    S->>S: Decrypt message using K1 → original data
    S->>S: 🗑️ Destroy K1 again
    Note over K: Master Key never leaves KMS!
```

---

## 5. Blast Radius — Limiting Damage

### The Problem with One Key for Everything

```mermaid
flowchart LR
    K1["🔑 K1 (one key)"] --> M["📨 ALL 1 Million Messages"]
    H["🏴‍☠️ Hacker steals K1"] --> D["💥 ALL 1M messages\nexposed instantly!"]
    style K1 fill:#fff3cd,stroke:#ffc107,color:#000
    style H fill:#f8d7da,stroke:#dc3545,color:#000
    style D fill:#f8d7da,stroke:#dc3545,color:#000
    style M fill:#e2e3e5,stroke:#6c757d,color:#000
```

> This is called a **huge blast radius** — one compromise = everything lost.

### The Fix — Use Different Keys Per Batch

```mermaid
flowchart TD
    K1["🔑 K1"] --> B1["📦 Batch 1\n(msgs 1–10,000)"]
    K2["🔑 K2"] --> B2["📦 Batch 2\n(msgs 10,001–20,000)"]
    K3["🔑 K3"] --> B3["📦 Batch 3\n(msgs 20,001–30,000)"]
    K4["🔑 K4"] --> B4["📦 Batch 4\n(msgs 30,001–40,000)"]

    H["🏴‍☠️ Hacker steals K2"] --> X["💥 ONLY Batch 2\nexposed (10,000 msgs)"]
    H -.->|"K1, K3, K4 are safe ✅"| S["🛡️ 990,000 msgs\nstill protected"]

    style K1 fill:#d4edda,stroke:#28a745,color:#000
    style K2 fill:#f8d7da,stroke:#dc3545,color:#000
    style K3 fill:#d4edda,stroke:#28a745,color:#000
    style K4 fill:#d4edda,stroke:#28a745,color:#000
    style H fill:#f8d7da,stroke:#dc3545,color:#000
    style X fill:#f8d7da,stroke:#dc3545,color:#000
    style S fill:#d4edda,stroke:#28a745,color:#000
```

### Tradeoff — Batch Size vs Performance

| Strategy | Security | Performance | Blast Radius |
|---|---|---|---|
| 1 key for ALL messages | ❌ Dangerous | ✅ Fast | 💥 Huge (all data) |
| 1 key per BATCH (10k msgs) | ✅ Good balance | ✅ Fast | ✅ Small (10k msgs) |
| 1 key per MESSAGE | ✅ Best security | ❌ Very slow | ✅ Minimal (1 msg) |

> **Best Practice:** Use one key per batch. Balance between security and performance.

---

## 6. Master Key Rotation

### The Risk of a Static Master Key

```mermaid
flowchart LR
    MK["🔑 Master Key\n(Static, never changes)"] -->|Encrypts| E1["E(K1)"]
    MK -->|Encrypts| E2["E(K2)"]
    MK -->|Encrypts| E3["E(K3)"]
    H["🏴‍☠️ Hacker steals Master Key"] --> D["💥 ALL encrypted keys\ncan be decrypted!\nALL data exposed!"]
    style MK fill:#fff3cd,stroke:#ffc107,color:#000
    style H fill:#f8d7da,stroke:#dc3545,color:#000
    style D fill:#f8d7da,stroke:#dc3545,color:#000
```

### Solution — Rotate Master Key Every 90/180 Days

```mermaid
timeline
    title Master Key Rotation Timeline
    Day 0   : 🔑 MasterKey_1 active
            : Keys K1–K100 encrypted with MK1
    Day 90  : 🔑 MasterKey_2 replaces MK1
            : New keys K101–K200 encrypted with MK2
    Day 180 : 🔑 MasterKey_3 replaces MK2
            : New keys K201–K300 encrypted with MK3
    Day 270 : 🔑 MasterKey_4 replaces MK3
            : New keys K301+ encrypted with MK4
```

### Benefits of Key Rotation

| Without Rotation | With Rotation (every 90 days) |
|---|---|
| One stolen master key = everything lost forever | Only ~90 days of data at risk |
| Hacker has permanent unlimited access | Key useless after next rotation |
| No time limit on attack | Attack window is limited |
| No way to recover | Can re-encrypt and recover |

### Incident Response If Master Key is Stolen

```mermaid
flowchart TD
    A["🚨 Master Key Compromised!"] --> B["1. Find all DEKs encrypted with this Master Key"]
    B --> C["2. Decrypt all those DEKs (before rotating)"]
    C --> D["3. Re-encrypt all affected messages with new keys"]
    D --> E["4. Re-encrypt all DEKs with new Master Key"]
    E --> F["5. Destroy / Invalidate the compromised Master Key"]
    F --> G["✅ System is secure again"]

    style A fill:#f8d7da,stroke:#dc3545,color:#000
    style G fill:#d4edda,stroke:#28a745,color:#000
```

---

## 7. Server Authentication with KMS

### The Problem

```mermaid
flowchart LR
    S["🖥️ Your Server\n'I need a key'"] -->|Request| K["🔐 KMS"]
    H["🏴‍☠️ Hacker\n'I need a key too'"] -->|Same Request| K
    K --> Q["❓ How does KMS\nknow who is\nlegitimate?"]

    style S fill:#cce5ff,stroke:#004085,color:#000
    style H fill:#f8d7da,stroke:#dc3545,color:#000
    style Q fill:#fff3cd,stroke:#ffc107,color:#000
```

### Why Simple Solutions Fail

```mermaid
flowchart TD
    Q["❓ How to prove identity to KMS?"]
    Q --> A["Username/Password in DB"]
    Q --> B["Token in code"]
    Q --> C["Token in .env"]
    A --> X1["❌ DB hacked = credentials stolen"]
    B --> X2["❌ Code leaked = token exposed"]
    C --> X3["❌ Server hacked = token stolen"]
    Q --> D["🤔 We keep going in circles..."]

    style Q fill:#e2e3e5,stroke:#6c757d,color:#000
    style A fill:#f8d7da,stroke:#dc3545,color:#000
    style B fill:#f8d7da,stroke:#dc3545,color:#000
    style C fill:#f8d7da,stroke:#dc3545,color:#000
    style D fill:#fff3cd,stroke:#ffc107,color:#000
```

### The AWS Solution — Same Account = Automatic Trust

```mermaid
flowchart TD
    AWS["☁️ AWS Account (Rohit's Account)"]
    AWS --> EC2["🖥️ EC2 Server\nID: 123"]
    AWS --> KMS["🔐 KMS Service\nID: 3281"]
    
    EC2 <-->|"AWS knows both belong\nto same account ✅\nNo password needed!"| KMS

    style AWS fill:#FF9900,stroke:#FF9900,color:#fff
    style EC2 fill:#cce5ff,stroke:#004085,color:#000
    style KMS fill:#d4edda,stroke:#28a745,color:#000
```

### How the Token Works (IAM Role)

```mermaid
flowchart TD
    A["🚀 Server Starts"] --> B["☁️ AWS assigns short-lived\ntoken to server's RAM"]
    B --> C{"Token Properties"}
    C --> D["⏰ Auto-rotates every 1 hour"]
    C --> E["🧠 Stored in RAM only\n(not disk, not DB)"]
    C --> F["🔒 Only accessible inside\nprivate AWS network"]
    D & E & F --> G["✅ Server uses token\nto talk to KMS"]
    G --> H["🔐 KMS verifies token\nand serves the request"]

    style A fill:#e2e3e5,stroke:#6c757d,color:#000
    style B fill:#fff3cd,stroke:#ffc107,color:#000
    style G fill:#d4edda,stroke:#28a745,color:#000
    style H fill:#d4edda,stroke:#28a745,color:#000
```

### Security Layers of KMS

```mermaid
flowchart LR
    subgraph L["🛡️ KMS Security Layers"]
        direction TB
        L1["Layer 1: Private IP\nNot publicly accessible"]
        L2["Layer 2: AWS Account Trust\nOnly same-account servers allowed"]
        L3["Layer 3: Rotating Tokens\n1-hour auto-rotation"]
        L4["Layer 4: Minimal Code\nFewer bugs = harder to hack"]
        L5["Layer 5: HSM Hardware\nKeys in dedicated secure chips"]
        L1 --> L2 --> L3 --> L4 --> L5
    end

    style L fill:#f8f9fa,stroke:#6c757d
    style L1 fill:#d4edda,stroke:#28a745,color:#000
    style L2 fill:#d4edda,stroke:#28a745,color:#000
    style L3 fill:#d4edda,stroke:#28a745,color:#000
    style L4 fill:#d4edda,stroke:#28a745,color:#000
    style L5 fill:#d4edda,stroke:#28a745,color:#000
```

---

## 8. AWS KMS — Real World Example

### Storing Your OpenAI API Key Securely

```mermaid
sequenceDiagram
    participant App as 🖥️ Your Server
    participant KMS as 🔐 AWS KMS
    participant DB as 🗄️ Database

    Note over App,DB: 📝 WRITE — Storing the API key
    App->>KMS: "Give me a DEK to encrypt my API key"
    KMS->>App: Returns K1 (plain) + E(K1) (encrypted)
    App->>App: encrypted_api = encrypt(openai_key, K1)
    App->>DB: Store encrypted_api + E(K1)
    App->>App: 🗑️ Destroy K1 from memory

    Note over App,DB: 📖 READ — Using the API key later
    App->>DB: Fetch encrypted_api + E(K1)
    DB->>App: Returns encrypted_api + E(K1)
    App->>KMS: Send E(K1) → "Decrypt this"
    KMS->>App: Returns plain K1
    App->>App: openai_key = decrypt(encrypted_api, K1)
    App->>App: Make API request with openai_key
    App->>App: 🗑️ Destroy K1 again
```

### AWS Services Involved

```mermaid
flowchart TD
    Dev["👨‍💻 Developer\n(Rohit's AWS Account)"] --> EC2["🖥️ EC2 / ECS\n(Your App Server)"]
    Dev --> AWSKMS["🔐 AWS KMS\n(Key Management)"]
    Dev --> RDS["🗄️ RDS / DynamoDB\n(Database)"]
    Dev --> IAM["🪪 IAM Roles\n(Authentication)"]

    IAM -->|Auto-links| EC2
    IAM -->|Auto-links| AWSKMS
    EC2 <-->|"Encrypted requests\n(via private network)"| AWSKMS
    EC2 <-->|"Store/fetch\nencrypted data"| RDS

    style Dev fill:#FF9900,stroke:#FF9900,color:#fff
    style EC2 fill:#cce5ff,stroke:#004085,color:#000
    style AWSKMS fill:#d4edda,stroke:#28a745,color:#000
    style RDS fill:#e2e3e5,stroke:#6c757d,color:#000
    style IAM fill:#fff3cd,stroke:#ffc107,color:#000
```

---

## 9. Key Concepts Cheat Sheet

| Term | Full Form | Meaning |
|---|---|---|
| **KMS** | Key Management System | A dedicated secure service for storing & using cryptographic keys |
| **DEK** | Data Encryption Key | Key generated by KMS; you use it to encrypt your actual data locally |
| **Master Key** | — | Top-level key inside KMS; used to encrypt/decrypt DEKs |
| **Envelope Encryption** | — | Encrypting a DEK with the Master Key so it's safe to store |
| **E(K1)** | Encrypted K1 | Encrypted form of DEK; safe to store in DB; useless without Master Key |
| **Blast Radius** | — | How much data is exposed if a key is stolen. Smaller = better |
| **Key Rotation** | — | Periodically replacing keys to limit the exposure window |
| **HSM** | Hardware Security Module | Physical chip that stores keys with extreme physical security |
| **IAM Role** | Identity & Access Management | AWS system that lets services authenticate each other without passwords |
| **Short-lived Token** | — | Temporary 1-hour credential stored only in RAM; auto-rotated by AWS |

---

### 🧠 The Mental Model

```mermaid
flowchart TD
    subgraph Bank["🏦 Think of KMS Like a Bank Vault"]
        direction LR
        YD["Your Data\n= Valuables in\nsafe deposit boxes"]
        DEK["DEK K1\n= Key to\nyour specific box"]
        MK["Master Key\n= Bank's master key\n(opens any box)"]
        KMS["KMS\n= The bank vault\nitself"]
        EE["Envelope Encryption\n= Locking your box key\ninside another box"]
    end

    Note["💡 You never carry the bank's master key.\nYou just ask the bank to open your box.\nThe bank verifies WHO you are first."]

    style Bank fill:#f8f9fa,stroke:#6c757d
    style Note fill:#fff3cd,stroke:#ffc107,color:#000
    style YD fill:#cce5ff,stroke:#004085,color:#000
    style DEK fill:#d4edda,stroke:#28a745,color:#000
    style MK fill:#f8d7da,stroke:#dc3545,color:#000
    style KMS fill:#e2e3e5,stroke:#6c757d,color:#000
    style EE fill:#fff3cd,stroke:#ffc107,color:#000
```

---

### 🌐 Real-World KMS Services

| Provider | Service |
|---|---|
| ☁️ AWS | AWS KMS (Key Management Service) |
| 🟦 Google Cloud | Cloud KMS |
| 🟪 Azure | Azure Key Vault |
| 🏗️ HashiCorp | Vault |

---

*Notes based on KMS concepts in System Design & Security*  
*Topics: Envelope Encryption · Blast Radius · Key Rotation · AWS IAM · Master Keys · HSM*
