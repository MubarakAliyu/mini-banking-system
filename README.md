# Mini Banking System - Transaction Flow and Distribution Planning

## Overview

This checkpoint consolidates concepts of concurrency control and distributed database design through a mini banking system scenario.

The system supports:
- Transferring money between accounts
- Viewing account balances

---

# Part 1 - Transaction Management

## 1. Concurrency Issue

When two users simultaneously initiate transfers involving the same account, a **lost update** can occur.

A lost update happens when two transactions read the same account balance, calculate different new balances, and then one transaction writes its result over the other transaction's update.

For example:

- Transaction T1 reads the balance as 1000.
- Transaction T2 also reads the balance as 1000.
- T1 subtracts 200 and prepares a balance of 800.
- T2 subtracts 300 and prepares a balance of 700.
- T1 writes 800.
- T2 writes 700.

The update made by T1 is lost because T2 overwrites it.

## 2. Locking Mechanism

Use an **exclusive (X) lock** on the account records being updated.

An exclusive lock allows only one transaction to write to the locked data at a time. The second transaction must wait until the first transaction completes and releases the lock.

For a money transfer, both the source account and destination account should be protected during the transaction.

## 3. Pessimistic or Optimistic Locking?

Use **pessimistic locking**.

The banking transfer scenario involves financial data where conflicting updates can have serious consequences. Pessimistic locking assumes a conflict may occur and locks the required account data before the update proceeds.

This prevents two transfers from modifying the same account balance at the same time.

## 4. Concurrent Transaction Schedule

| Step | Transaction T1 | Transaction T2 | Status |
|---|---|---|---|
| 1 | Requests exclusive lock on Account A | — | T1 gets lock |
| 2 | Reads Account A balance | Requests exclusive lock on Account A | T2 waits |
| 3 | Updates Account A balance | Waiting | Safe |
| 4 | Commits transaction | Waiting | Safe |
| 5 | Releases lock | Gets exclusive lock | Safe |
| 6 | — | Reads latest Account A balance | Safe |
| 7 | — | Updates Account A balance | Safe |
| 8 | — | Commits and releases lock | Safe |

### Is the schedule safe?

**Yes.**

T2 cannot update the account until T1 has finished and released the exclusive lock. Therefore, the transactions do not overwrite each other's updates.

---

# Part 2 - Distributed Database Planning

The bank has three branches:

- Tunis
- Sousse
- Sfax

## 1. Horizontal Fragmentation of Customers

Use **horizontal fragmentation based on branch**.

The Customers table can be divided into three row-based fragments:

```text
Customers_Tunis
    → customers whose branch = 'Tunis'

Customers_Sousse
    → customers whose branch = 'Sousse'

Customers_Sfax
    → customers whose branch = 'Sfax'
```

Each fragment contains the same columns but a different subset of customer records.

Example:

```text
Customers(customer_id, name, email, phone, branch)

Customers_Tunis
Customers_Sousse
Customers_Sfax
```

This improves data locality because each branch can access the customer records most relevant to it.

## 2. Vertical Separation

A column such as **customer email** can be vertically separated into an authentication/contact table.

Example:

```text
Customers(
    customer_id,
    name,
    phone,
    branch
)

CustomerContact(
    customer_id,
    email
)
```

The `customer_id` should remain in both tables so the original information can be reconstructed using a JOIN.

## 3. Replication Strategy

The data most suitable for replication across all branches is **customer information**.

Reason:

- All branches may need to identify and serve customers.
- Local copies can improve availability and access speed.
- Replication provides fault tolerance if one branch becomes unavailable.

**Account balances and transaction history should be handled more carefully.** These records are frequently updated and require strong consistency, so replicating every write to every branch would increase coordination and update costs.

A practical high-level plan is:

```text
Customer information → Replicate across branches

Account balances → Keep strongly synchronized/controlled

Transaction history → Store at the appropriate branch and distribute
                      according to access and processing requirements
```

## 4. Static or Dynamic Allocation for Transaction History?

Use **dynamic allocation** for transaction history.

Transaction access patterns can change over time. Dynamic allocation allows the system to move or reorganize transaction-history fragments based on workload and access patterns.

This can help:

- Reduce access time
- Reduce communication cost
- Adapt to changing workloads

Although dynamic allocation is more complex to implement, it provides flexibility for transaction data whose usage may change over time.

---

# Concepts Demonstrated

## Concurrency Control

- Lost update
- Exclusive locking
- Pessimistic locking
- Safe transaction schedules

## Distributed Database Design

- Horizontal fragmentation
- Vertical fragmentation
- Replication
- Dynamic allocation
- Data locality
- Fault tolerance
- Consistency considerations

---

# Conclusion

The mini banking system uses pessimistic concurrency control with exclusive locks to prevent conflicting balance updates. For distributed design, customer records are horizontally fragmented by branch, sensitive contact information can be vertically separated, frequently accessed customer information can be replicated for availability, and transaction history can use dynamic allocation to adapt to changing access patterns.
