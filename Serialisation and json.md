# Serialization & Deserialization — Why JSON Exists

> Notes from: *"JSON Is Dead"* — video by Piyush Garg

---

## 1. The Core Problem — Memory vs Network

When your server stores data, it lives in **two places in RAM**:

```mermaid
graph TD
    A["users variable\n(Stack Memory)\nPointer: 0x3EDF..."] -->|points to| B

    subgraph HEAP["Heap Memory"]
        B["[ Array of Objects ]"]
        B --> C["{ fname: 'Piyush'\n  lname: 'Garg'\n  email: '...'\n  interests: [...] }"]
        B --> D["{ fname: '...'\n  lname: '...'\n  email: '...' }"]
    end

    style A fill:#eeedfe,stroke:#4a3fc0,color:#26215c
    style B fill:#c8eedd,stroke:#0f6e56,color:#085041
    style C fill:#ffffff,stroke:#9fe1cb,color:#085041
    style D fill:#ffffff,stroke:#9fe1cb,color:#085041
```

> **Key insight:** The `users` variable is just a pointer — a memory address like `0x3EDF...`. That address is meaningless to another machine. **You cannot send heap memory over a network.**

---

## 2. Serialization & Deserialization Flow

```mermaid
flowchart LR
    A["Object\nin Heap\n(Server)"]
    B["String / Binary\ntransferable"]
    C["String / Binary\nreceived"]
    D["Object\nin Heap\n(Client)"]

    A -->|"JSON.stringify()\nserialize"| B
    B -->|"Network\nHTTP / TCP"| C
    C -->|"JSON.parse()\ndeserialize"| D

    style A fill:#e1f5ee,stroke:#0f6e56,color:#085041
    style B fill:#eeedfe,stroke:#4a3fc0,color:#26215c
    style C fill:#eeedfe,stroke:#4a3fc0,color:#26215c
    style D fill:#e1f5ee,stroke:#0f6e56,color:#085041
```

### How JSON.parse() works internally

The parser scans the string **character by character**:

```mermaid
flowchart LR
    A["{"] -->|create object| B["&quot;fname&quot;"]
    B -->|property key| C[":"]
    C --> D["&quot;Piyush&quot;"]
    D -->|property value| E[","]
    E --> F["&quot;lname&quot;"]
    F -->|next key| G[":"]
    G --> H["&quot;Garg&quot;"]
    H -->|next value| I["}"]
    I -->|object complete| J["Object reconstructed"]

    style A fill:#eeedfe,stroke:#4a3fc0,color:#26215c
    style I fill:#eeedfe,stroke:#4a3fc0,color:#26215c
    style B fill:#e1f5ee,stroke:#0f6e56,color:#085041
    style D fill:#faeeda,stroke:#854f0b,color:#412402
    style F fill:#e1f5ee,stroke:#0f6e56,color:#085041
    style H fill:#faeeda,stroke:#854f0b,color:#412402
    style J fill:#e1f5ee,stroke:#0f6e56,color:#085041
```

---

## 3. The Size Problem at Scale

JSON is human-readable but **not the most efficient** format. Every key name, every quote, every colon uses bytes on the wire.

### Real demo — same user data, two serializers:

| Serializer | Bytes used |
|---|---|
| `JSON.stringify(users)` | **969 bytes** |
| `msgpack.encode(users)` | **58 bytes** |

MessagePack is ~94% smaller for the same data.

### Why size matters:

```mermaid
flowchart LR
    A["Save 10 bytes\nper API call"]
    B["x 1,000,000\nAPI calls"]
    C["= 10,000,000 bytes\nsaved per second"]

    A --> B --> C

    style A fill:#eeedfe,stroke:#4a3fc0,color:#26215c
    style B fill:#eeedfe,stroke:#4a3fc0,color:#26215c
    style C fill:#e1f5ee,stroke:#0f6e56,color:#085041
```

---

## 4. Serializers Compared

| Serializer | Format | Speed | Size | Best for |
|---|---|---|---|---|
| **JSON** | Human-readable text | ~9500 ns/op | Largest | REST APIs, general use |
| **MessagePack** | Binary | Faster | ~40% smaller | High-throughput APIs |
| **Protobuf** | Binary + schema | Very fast | Very compact | gRPC, microservices |
| **Custom** | Anything | Depends | Depends | Special cases |

> **gRPC never uses JSON.** It uses **Protobuf** as its serializer — binary, schema-defined, much faster and smaller.

---

## 5. Both Sides Must Agree

```mermaid
flowchart LR
    A["Server\nNode.js\nserialize with JSON\nor Protobuf or msgpack"]
    B(["serialized bytes\nover the network"])
    C["Client\nGo / Python / JS\ndeserialize with JSON\nmust match server!"]

    A -->|encode| B -->|decode| C

    style A fill:#e1f5ee,stroke:#0f6e56,color:#085041
    style B fill:#e6f1fb,stroke:#185fa5,color:#0c447c
    style C fill:#eeedfe,stroke:#4a3fc0,color:#26215c
```

> **Mismatch = corrupt or unreadable data.** JSON is popular because every language supports it natively — serialize in Node.js, deserialize in Go, Python, Java, etc.

---

## 6. DSA Connection — Serializing a Binary Tree

A binary tree is a heap data structure. It **cannot** be sent over a network directly. This is the real reason the classic "Serialize and Deserialize a Binary Tree" problem exists.

```mermaid
graph TD
    A["1 (root)"] --> B["2"]
    A --> C["3"]
    B --> D["4"]
    B --> E["5"]
    C --> F["6"]
    C --> G["null"]

    style A fill:#eeedfe,stroke:#4a3fc0,color:#26215c
    style B fill:#e1f5ee,stroke:#0f6e56,color:#085041
    style C fill:#e1f5ee,stroke:#0f6e56,color:#085041
    style D fill:#faeeda,stroke:#854f0b,color:#412402
    style E fill:#faeeda,stroke:#854f0b,color:#412402
    style F fill:#faeeda,stroke:#854f0b,color:#412402
    style G fill:#f6f5f0,stroke:#8a8a85,color:#5a5a56
```

**Serialized string:** `"B:1,2,3,4,5,n,6"` — now transferable over the network ✓

The deserializer reads this string and reconstructs the identical tree on the client.

---

## 7. When to Use What

```mermaid
flowchart TD
    A["Building an API?"] --> B{"Millions of\nrequests per second?"}
    B -->|No| C["JSON is fine\nres.json(data)"]
    B -->|Yes| D{"Every byte\nmatters?"}
    D -->|Yes| E["Protobuf\ngRPC"]
    D -->|Maybe| F["MessagePack\nbinary JSON"]

    style A fill:#f6f5f0,stroke:#8a8a85,color:#1a1a18
    style C fill:#e1f5ee,stroke:#0f6e56,color:#085041
    style E fill:#faece7,stroke:#993c1d,color:#4a1b0c
    style F fill:#faeeda,stroke:#854f0b,color:#412402
```

---

## Key Takeaways

- You **cannot send heap memory** over a network — only transferable (primitive) data
- **Serialization** = convert in-memory object → transferable bytes/string
- **Deserialization** = reconstruct the object on the receiving end
- **JSON** is just one serializer — Protobuf and MessagePack are faster/smaller alternatives
- **CPU cost** of serializing/deserializing also matters — a slow serializer bottlenecks your server
- At millions of requests, even a few saved bytes per call compound enormously
- The DSA "serialize a binary tree" problem exists for exactly this reason

---

*Source: "JSON Is Dead" — video by Piyush Garg*
