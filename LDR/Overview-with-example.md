### 1. What is LDR?

**LDR = Logical Data Replication.**

> One CockroachDB cluster sends its data changes to another CockroachDB cluster.

In our example:

**Mumbai Cluster ↔ Singapore Cluster**

Both sides can be active.

```text
Mumbai Cluster                    Singapore Cluster
   2 Nodes                              2 Nodes
      │                                    │
      └──────────── LDR ───────────────────┘
                 ↕
          Data Changes
```

So, when data changes in Mumbai, the change can be replicated to Singapore.

And when data changes in Singapore, the change can be replicated back to Mumbai.

That is why we call it **Bidirectional LDR**.

---

### 2. Step 1 — Prepare both clusters

First, we prepare **Mumbai and Singapore**.

We do things like:

```text
Mumbai
  │
  ├── Enable rangefeed
  ├── Create replication user
  └── Grant required replication privilege
```

Same thing on Singapore:

```text
Singapore
  │
  ├── Enable rangefeed
  ├── Create replication user
  └── Grant required replication privilege
```

#### Why rangefeed?

Very simple:

> Rangefeed helps CockroachDB continuously watch for changes happening to the data.

For example:

```text
UPDATE customer
SET city = 'Hyderabad'
WHERE id = 101;
```

CockroachDB needs to know:

> "Something changed."

LDR uses this change information for replication.

---

### 3. Step 2 — External Connections

Now Mumbai needs to know **how to connect to Singapore**.

And Singapore needs to know **how to connect to Mumbai**.

Think about it like saving another server's connection details.

```text
Mumbai
   │
   │  "How can I reach Singapore?"
   ▼
Singapore URI


Singapore
   │
   │  "How can I reach Mumbai?"
   ▼
Mumbai URI
```

So each cluster stores the **other cluster's connection information**.

This is why we create **External Connections**.

---

### 4. Step 3 — Start LDR

Now we actually start replication.

Important point:

> The LDR stream is started from the **destination cluster**.

For example:

```text
Mumbai ─────────────────────► Singapore
       Source                  Destination
```

Here:

* Mumbai = Source
* Singapore = Destination

The destination Singapore starts/accepts the LDR setup.

---

### 5. First LDR stream

We create the first replication direction:

```text
Mumbai
  │
  │  Data Changes
  │  ───────────────►
  │
Singapore
```

Example:

```text
Mumbai:

INSERT INTO customer VALUES (101, 'Ravi');
```

LDR captures the logical change and sends it toward Singapore.

Conceptually:

```text
Mumbai
  │
  │ INSERT customer 101
  ▼
LDR Stream
  │
  ▼
Singapore
  │
  ▼
customer 101
```

---

### 6. Second LDR stream

For bidirectional replication, we need another stream in the opposite direction.

```text
Mumbai ◄──────────────────── Singapore
       Reverse LDR Stream
```

Now Singapore can send its changes to Mumbai.

So finally:

```text
              LDR
       ┌─────────────────┐
       │                 │
       ▼                 │
   Mumbai ◄──────────► Singapore
       │                 │
       └─────────────────┘
```

This is **Bidirectional LDR**.

---

### 7. Very important: LDR is not physical replication

This is a very important interview point.

#### Physical replication

Usually:

```text
Primary
   │
   │ Physical/WAL-level changes
   ▼
Replica
```

The replica is maintaining a physical copy of the database state.

#### LDR

LDR works at the **logical data-change level**.

Conceptually:

```text
Mumbai
   │
   │ Logical changes
   │ INSERT
   │ UPDATE
   │ DELETE
   ▼
LDR
   │
   ▼
Singapore
```

So don't think:

> "Mumbai disk blocks are copied to Singapore."

Instead think:

> "Changes to the data are logically replicated to the other cluster."

---

### 8. Step 4 — Monitor

After starting LDR, we should not simply assume:

> "Replication is working."

We need to monitor it.

We check from the **DB Console** and relevant SQL/replication status.

We want to confirm:

```text
Mumbai → Singapore
        HEALTHY

Singapore → Mumbai
        HEALTHY
```

We should look for things such as:

* Replication status
* Stream health
* Replication lag
* Errors
* Connection problems
* Data replication progress

---

### 9. Final architecture

The complete flow is:

```text
                LDR SETUP
                    │
          ┌─────────┴─────────┐
          │                   │
       MUMBAI             SINGAPORE
       2 Nodes              2 Nodes
          │                   │
          │  Step 1           │
          │  Prepare          │
          │                   │
          └─────────┬─────────┘
                    │
             Step 2: External
                Connections
                    │
                    ▼
             Step 3: Start LDR
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
       Mumbai ───────────► Singapore
          ▲                   │
          │                   │
          └───────────────────┘
              Reverse LDR

                    │
                    ▼
             Step 4: Monitor

                    │
                    ▼

       MUMBAI ◄────────────► SINGAPORE
                ACTIVE
```

### 10. Important Points

Just remember these **4 steps**:

| Step             | What we do                       | Easy meaning                             |
| ---------------- | -------------------------------- | ---------------------------------------- |
| **1. Prepare**   | Rangefeed + user + privilege     | Make both clusters ready                 |
| **2. Connect**   | External Connections             | Tell each cluster how to reach the other |
| **3. Start LDR** | Create/start replication streams | Start sending changes                    |
| **4. Monitor**   | DB Console/status                | Check whether replication is healthy     |

### One-line explanation for your team

