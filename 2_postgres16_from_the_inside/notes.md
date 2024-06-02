## 1.1 Data Organization in PostgreSQL

#### Databases and Cluster:

PostgreSQL is a database management system (DBMS), and an instance of it is called a PostgreSQL server.
The server manages data stored in databases, which together form a cluster.
To use a cluster, it needs to be initialized by specifying a directory for storing files (PGDATA).

#### Cluster Initialization:

During initialization, three databases are created:
- template0 — for recovery and creating a database in another encoding.
- template1 — a template for new databases.
- postgres — a regular database for user use.

#### System Catalog:

Stores metadata about all cluster objects (tables, indexes, functions, etc.).
Each database has its own set of system catalog tables.
The system catalog is accessible via SQL queries and DDL commands.
Schemas:

#### Namespaces for objects in a database.
Special schemas include:
- public — default for user objects.
- pg_catalog — for the system catalog.
- information_schema — SQL standard.
- pg_toast — for storing large attributes.
- pg_temp — for temporary tables. 

#### Tablespaces:

 - Define the physical location of data.
 - During initialization, two tablespaces are created:
   - pg_default — default tablespace.
   - pg_global — for shared system catalog objects.
 - User tablespaces are specified during creation.

#### Relations:

Primary database objects, such as tables and indexes, consist of rows.
All objects (tables, indexes, views) are called relations.

#### Forks and Files:

- Data is organized into forks, each represented by files.
- The main fork contains the actual data.
- Additional forks include:
   - init fork — for unlogged tables.
   - free space map (fsm) — tracks free space.
   - visibility map (vm) — tracks visibility for vacuum and freeze operations.
  
#### Pages:

Files are divided into pages (usually 8 KB) to facilitate data I/O.

#### TOAST:

- Technology for storing long rows that don't fit on a single page.
- Long values can be compressed or stored in a separate toast table.
- Compression algorithms: PGLZ and LZ4 (LZ4 is faster and more resource-efficient).
- 
#### Recommendations:

Storing large amounts of data in PostgreSQL is not ideal, especially if transactional logic is unnecessary.
For such data, it's better to use the file system, keeping only file paths in the database.


## 1.2 Processes and Memory in PostgreSQL
Server Instance:

A PostgreSQL server instance consists of several interacting processes.
The primary process, traditionally called postmaster, is the first to start when the server starts.
postmaster launches all other processes and monitors them, restarting them if they crash.
The process model in PostgreSQL has been used since the project's inception due to its simplicity.
Discussions about switching to a threaded model continue due to the potential benefits, but such a transition would require significant changes and time.
#### Key Service Processes:

- startup: Recovers the system after failures.
- autovacuum: Cleans tables and indexes of obsolete data.
- wal writer: Writes journal records to disk.
- checkpointer: Creates checkpoints.
- writer: Writes dirty pages to disk.
- wal sender: Sends journal records to a replica.
- wal receiver: Receives journal records on a replica.
- Configuration and Memory Management:

Each process is managed by configuration parameters, sometimes dozens of them.
Proper server configuration requires a good understanding of its internal workings.
Initial parameter values are chosen based on general considerations and then refined using monitoring feedback.
To facilitate process communication, postmaster allocates shared memory accessible to all processes.
#### Caching:

Disks (both HDD and SSD) are significantly slower than RAM, so caching is used.
Recently read pages are stored in RAM, and modified data is written to disk with a delay.
The buffer cache takes up most of the shared memory, containing other buffers that accelerate disk operations.
The operating system also has its own cache, resulting in double caching.
PostgreSQL rarely uses direct I/O, so it relies on the OS cache as well.
#### Handling Crashes:

In the event of a crash (e.g., power failure or OS crash), the contents of RAM, including the buffer cache, are lost.
Disk files may contain pages written at different times, necessitating data consistency restoration.
PostgreSQL maintains a write-ahead log (WAL) to replay lost operations and ensure data consistency.


## 1.3 Clients and Client-Server Protocol
#### Postmaster and Client Connections:

One of the responsibilities of the postmaster process is to listen for incoming connections.
When a new client connection is detected, postmaster spawns a backend process to handle it.
The client establishes a connection and starts a session with its server process, which lasts until the client disconnects or the connection is interrupted.
#### Handling Multiple Clients:

Each client gets its own backend process on the server.
High numbers of connections can cause issues:
Each process needs memory for caching system catalog data, prepared queries, intermediate query results, and other data. More connections require more available memory.
Frequent connections and short sessions (where the client performs a small query and disconnects) consume excessive resources on connection setup, process creation, and cache population.
More processes increase the time needed to manage the process list, potentially reducing performance as the number of clients grows.
#### Connection Pooling:

To mitigate these issues, connection pooling is used to limit the number of backend processes.
PostgreSQL lacks built-in connection pooling, so third-party solutions are used, such as PgBouncer or Odyssey.
Typically, a single server process handles transactions from different clients sequentially.
This imposes certain limitations on application development, as only transaction-local resources can be used, not session-local ones.
#### Client-Server Protocol:

Clients and servers must use the same protocol to communicate. The standard library for this is libpq, though independent implementations exist.
Generally, the protocol allows the client to connect to the server and execute SQL queries.
#### Connection Details:

Connections are always made under a specific role (user) and to a specific database.
Although the server operates with a database cluster, applications must connect separately to each database they use.
Authentication ensures that the user is who they claim to be (e.g., by asking for a password) and verifies their permissions to connect to the server and the selected database.
#### SQL Query Execution:

SQL queries are sent to the backend process in text form.
The process parses, optimizes, executes the query, and returns the result to the client.

# 2. Isoaltion

### 2.1 Consistency
An important feature of relational DBMSs is ensuring consistency, meaning data correctness.

#### Integrity Constraints:

Databases can enforce integrity constraints such as NOT NULL or UNIQUE to maintain data integrity.
The DBMS ensures that data never violates these constraints.
#### Consistency vs. Integrity:

Consistency is stricter than integrity. Some conditions are too complex to be defined at the database level, involving multiple tables or other considerations.
The DBMS cannot inherently understand all application-specific consistency rules. If an application violates consistency without breaking integrity constraints, the DBMS won't detect it.
Thus, applications must ensure they maintain consistency.
#### Role of the DBMS:

Even correct sequences of operations can temporarily violate consistency. For example, transferring funds between accounts involves two operations: deducting from one account and adding to another. The first operation violates consistency until the second operation restores it.
If a failure occurs after the first operation but before the second, consistency is broken. The DBMS can handle this by treating these operations as an indivisible transaction.
#### Concurrent Transactions:

Correct individual transactions can interact incorrectly when executed concurrently, leading to anomalies.
For instance, transactions should not see changes from other uncommitted transactions to avoid anomalies like dirty reads.
The DBMS must isolate transactions from each other to prevent anomalies and ensure that the result of concurrent execution matches one of the possible sequential executions.
#### Transaction Definition:

A transaction is a set of operations that move the database from one consistent state to another, assuming it completes fully (atomicity) and without interference from other transactions (isolation).
This definition encompasses the first three letters of the ACID acronym: Atomicity, Consistency, Isolation. Durability is also crucial, as system crashes require dealing with changes from uncommitted transactions to restore consistency.
#### Isolation Levels and Anomalies:

Full isolation is technically challenging and impacts performance, so DBMSs often use weakened isolation levels that prevent some, but not all, anomalies.
Applications need to understand the isolation level used, its guarantees, and its limitations to write correct code.

## 2.2 Isolation Levels and Anomalies in the SQL Standard
The SQL standard describes four isolation levels, defined by the anomalies they allow or disallow. Understanding isolation levels requires understanding these anomalies first.

#### Lost Update
The lost update anomaly occurs when two transactions read the same row, and then both update it based on the initial value, resulting in one update being lost. For example, if two transactions both plan to add 100 to an account with 1000 units, each reads the initial 1000 units. The first transaction updates it to 1100 units, then the second transaction updates it to 1100 units again, effectively losing one of the updates.

The SQL standard does not allow lost updates at any isolation level.

#### Dirty Read and Read Uncommitted
A dirty read occurs when a transaction reads uncommitted changes made by another transaction. For example, one transaction transfers 100 units to an account but does not commit. Another transaction reads this uncommitted balance and allows withdrawal, even though the first transaction might roll back, leaving the account with no funds.

Dirty reads are allowed at the Read Uncommitted isolation level.

#### Non-Repeatable Read and Read Committed
A non-repeatable read happens when a transaction reads the same row twice and finds different values because another transaction has modified and committed the row in between the reads. For example, one transaction checks an account balance, finding 1000 units, and plans a withdrawal. Meanwhile, another transaction reduces the balance to zero and commits. The first transaction, if it does not recheck, might cause an overdraft.

Non-repeatable reads are allowed at the Read Uncommitted and Read Committed levels.

#### Phantom Read and Repeatable Read
A phantom read occurs when a transaction reads a set of rows based on a condition, and another transaction inserts new rows that match the condition and commits. When the first transaction re-reads, it finds additional rows. For example, if a rule limits a client to three accounts, one transaction checks and finds two accounts. Meanwhile, another transaction opens a third account and commits. The first transaction, without rechecking, might open a fourth account, violating the rule.

Phantom reads are allowed at the Read Uncommitted, Read Committed, and Repeatable Read levels.

#### Serializable Level
The Serializable isolation level prevents all anomalies, ensuring that the outcome of concurrent transactions matches one of the possible sequential executions. This level guarantees that if transactions are correct individually, they will remain correct when executed concurrently.

#### Here is the standard table of anomalies and isolation levels, with an additional column for clarity:

Anomaly	Read Uncommitted	Read Committed	Repeatable Read	Serializable

| Anomaly              | Read Uncommitted  | Read Committed   | Repeatable Read  | Serializable  |
|----------------------|-------------------|------------------|------------------|---------------|
| Lost Updates         | No                | No               | No               | No            |
| Dirty Reads          | Yes               | No               | No               | No            |
| Non-Repeatable Reads | Yes               | Yes              | No               | No            |
| Phantom Reads        | Yes               | Yes              | Yes              | No            |
| Other Anomalies      | Yes               | Yes              | Yes              | No            |

#### Why These Anomalies?
The SQL standard lists only a few anomalies, possibly because when the standard was first developed, other anomalies were not well understood. The standard assumes isolation through locking. In practice:

- Read Uncommitted: Modifying rows are locked for changes but not for reading, allowing dirty reads.
- Read Committed: Rows are locked for both reading and writing, preventing dirty reads but allowing non-repeatable reads.
- Repeatable Read: Rows are locked for all operations, preventing non-repeatable reads but allowing phantom reads.
- Serializable: Requires predicate locking, where not just rows but the conditions for data are locked. However, practical predicate locking is limited by the complexity of the conditions.

Understanding these levels and the guarantees they provide is crucial for writing correct applications that maintain data consistency.


## 2.3 Isolation Levels in PostgreSQL

### Snapshot Isolation (SI)
Snapshot Isolation (SI) is a transaction management protocol where each transaction operates on a consistent snapshot of the database at a certain point in time. It includes all committed changes before the snapshot's creation. This approach minimizes locks, blocking only repeated modifications to the same row. Write transactions never block read transactions, and reads never block any transaction.

### Multi-Version Concurrency Control (MVCC)
PostgreSQL implements a multi-version variant of SI, allowing multiple versions of the same row to coexist. This prevents transaction interruptions due to reading outdated data. MVCC ensures data visibility based on transaction timestamps.

### Isolation Levels
 - Read Committed: This level avoids dirty reads but allows non-repeatable reads and phantom reads. Transactions see only committed data, but changes between operations can lead to inconsistencies.

- Repeatable Read: Prevents non-repeatable and phantom reads, ensuring consistent data visibility within a transaction. However, if concurrent updates occur, transactions may face serialization errors and need to be retried.

- Serializable: The strictest level, preventing all anomalies, including write skews and read-only transaction anomalies. It ensures full transaction isolation by detecting and aborting conflicting transactions, but this can lead to increased overhead and false positives.

#### Practical Implications
- Read Committed: Suitable for general use but requires caution to avoid making decisions based on possibly outdated data.
- Repeatable Read: Ensures stable data reads within a transaction but needs handling of potential serialization errors.
- Serializable: Provides the highest isolation level but may require transaction retries due to serialization errors, impacting performance.

#### Code Practices
- Avoid procedural checks: Use declarative constraints to ensure data integrity.
- Single SQL statements: Minimize intermediate states by using comprehensive SQL queries.
- User-defined locks: As a last resort, manual locks can ensure consistency but reduce concurrency benefits.

#### Conclusion
Choosing the right isolation level depends on the application's concurrency needs and tolerance for transaction overhead. Serializable offers the best consistency but at a performance cost, while Read Committed balances performance with some risk of anomalies. Repeatable Read provides a middle ground, requiring careful handling of serialization errors.


## 2.4 Which Isolation Level to Use?
### Read Committed:

Default Level: PostgreSQL uses Read Committed by default, and it is likely the most used isolation level in applications.
- Advantages: It does not cause transaction aborts due to consistency issues, except in case of a failure. There's no need to handle serialization errors or transaction retries.
- Disadvantages: A high number of possible anomalies can occur, which requires developers to be constantly aware and to write code that avoids these issues. Testing for these errors is challenging because they can be unpredictable and difficult to reproduce and fix.
### Repeatable Read:

- Advantages: It addresses some consistency issues but not all. It is beneficial for read-only transactions and can be used effectively for reporting that involves multiple SQL queries.
- Disadvantages: Developers must still handle remaining anomalies and be prepared for serialization errors. Applications need to be modified to handle these errors correctly, which adds complexity.
### Serializable:

- Advantages: This level ensures the highest consistency, removing the need to worry about inconsistencies. It simplifies coding as developers do not need to consider concurrency anomalies.
- Disadvantages: Applications must be able to retry any transaction that encounters a serialization error. The overhead and the percentage of aborted transactions can significantly reduce throughput. Additionally, this level is not applicable on replicas and cannot be mixed with other isolation levels.
### Conclusion
Read Committed: Suitable for most applications due to its simplicity and lack of serialization errors, but requires careful coding to avoid anomalies.
Repeatable Read: Best for read-heavy transactions or scenarios where some anomalies need to be avoided, but developers must handle serialization errors.
Serializable: Ideal for scenarios requiring strict consistency, but requires handling of transaction retries and comes with performance trade-offs.

## 3. Pages and raw versions.
## 3.1. Page Structure
Each page in PostgreSQL is structured with several distinct sections:

- Header: Located at the lower addresses, storing metadata such as checksums and sizes of other areas.
- Array of Pointers to Row Versions: Acts as the table of contents for the page.
- Free Space: Available space for new data.
- Row Versions: Actual data entries with some additional metadata.
- Special Area: Reserved for specific index types to store auxiliary information.
### Page Header
The page header has a fixed size and contains critical information about the page. Using the pageinspect extension, one can retrieve details of a page header:

```sql
CREATE EXTENSION pageinspect;
SELECT lower, upper, special, pagesize FROM page_header(get_raw_page('accounts', 0));
```
#### Special Area
Located at the higher addresses, the special area is used mainly by certain index types for additional information. For table pages, this area is typically zero-sized.

#### Row Versions
Each row in PostgreSQL is actually a row version, allowing multiple versions of the same row to coexist. This helps in implementing Multi-Version Concurrency Control (MVCC).

#### Pointers to Row Versions
The array of pointers to row versions serves as an index within the page. Each pointer, occupying 4 bytes, includes:

- Offset of the row version from the beginning of the page
- Length of the row version
- Status bits indicating the row's state
#### Free Space
Located between the pointer array and the row versions, this space is available for new data without causing fragmentation.

## 3.2. Row Version Structure
A row version consists of a header followed by the actual data. The header includes fields like:

- xmin, xmax: Transaction IDs that created and invalidated the row version, respectively.
- infomask: Various status bits.
- ctid: Pointer to the next version of the same row.
- NULL bitmap: Marks columns that have NULL values.
Row versions are stored on disk in the same format as in memory, making PostgreSQL data files platform-dependent due to differences in byte order and data alignment across architectures.

## 3.3. Operations on Row Versions
### Insertion
When inserting a row, its xmin is set to the current transaction ID, and xmax is set to 0, indicating it's valid. Example:

```sql
BEGIN;
INSERT INTO t(s) VALUES ('FOO');
SELECT pg_current_xact_id(); -- Returns transaction ID, e.g., 773
SELECT * FROM heap_page('t', 0);
```
After committing, the transaction status is recorded in the commit log (clog).

### Deletion
When deleting a row, its xmax is set to the current transaction ID, indicating it is deleted:

```sql
BEGIN;
DELETE FROM t;
SELECT pg_current_xact_id(); -- Returns transaction ID, e.g., 774
SELECT * FROM heap_page('t', 0);
```
### Update
An update operation is a combination of delete and insert, creating a new row version:

```sql
BEGIN;
UPDATE t SET s = 'BAR';
SELECT pg_current_xact_id(); -- Returns transaction ID, e.g., 775
SELECT * FROM t;
SELECT * FROM heap_page('t', 0);
```
After committing, the old row version is invalidated, and a new version is created.

## 3.4. Indexes
In PostgreSQL, indexes never contain multiple versions of the same row. Each row is represented by a single entry, and index entries do not include the xmin and xmax fields. Index entries point to all possible versions of table rows, and transactions must check the table to determine the visible version, unless the page is marked in the visibility map.

Using tools like pageinspect, you can inspect index pages and see pointers to both current and previous versions of table rows. For example, after an update, the index might show entries for both the new and old versions of a row, reflecting the changes in the sorted order of the index.

## 3.5. TOAST
A TOAST table is essentially a regular table with its own versioning system, which is separate from the versioning of the main table's rows. The way TOAST tables work internally ensures that rows are never updated but only added or deleted. This makes the versioning in TOAST somewhat degenerate.

#### Data Updates and Versioning
When data changes, a new version of the row is always created in the main table. If the update does not affect the "long" value stored in the TOAST table, the new version of the row will still reference the previous value in the TOAST table. Only when an update changes the "long" value will both a new version of the row in the main table and new TOAST entries be created.

## 3.6. Virtual Transactions
PostgreSQL employs an optimization technique to conserve transaction IDs. If a transaction is read-only, it does not affect the visibility of row versions. Therefore, the servicing process initially assigns a virtual transaction ID (virtual xid) to the transaction. This ID consists of the identifier of the servicing process and a sequential number. Assigning this ID does not require synchronization among all processes and is thus performed very quickly.

#### Virtual Transaction IDs
A virtual transaction does not have a real transaction ID yet:
At different times, the system may contain virtual transactions with IDs that have already been used. This is normal because virtual transaction IDs exist only in memory while the transaction is active; they are never written to data pages or stored on disk.

#### Transition to Real Transaction IDs
If a transaction starts modifying data, it is then assigned a real, unique transaction ID.

## 3.7. Nested Transactions
### Savepoints in SQL
SQL defines savepoints that allow rolling back part of a transaction without aborting it entirely. This does not fit into the usual transaction scheme since the transaction status is singular for all changes, and physically no data is rolled back.

To implement this functionality, a transaction with a savepoint is divided into several nested transactions (subtransactions), each with its own status that can be managed independently. Nested transactions have their own numbers, higher than the main transaction number. The status of nested transactions is recorded in the clog as usual, but committed nested transactions are marked with two bits: committed and aborted. The final status depends on the status of the main transaction—if the main transaction is aborted, all nested transactions are considered aborted as well.

### Storage and Management
Information about nested transactions is stored in files within the PGDATA/pg_subtrans directory. Access to these files is managed through shared memory buffers organized similarly to clog buffers.

### Differences from Autonomous Transactions
Nested transactions should not be confused with autonomous transactions. Autonomous transactions are independent of each other, unlike nested transactions. Autonomous transactions do not exist in standard PostgreSQL, which is generally seen as beneficial since they are rarely needed and their presence in other database systems often leads to misuse.

Example Workflow
1. Start a transaction and insert a row:
   - A new row is added, and the current transaction ID is noted.
2. Set a savepoint and insert another row:
   - A savepoint is set, and another row is inserted. The main transaction ID remains the same.
3. Rollback to the savepoint and insert a third row:
   - The transaction rolls back to the savepoint, and a new row is inserted.

In the page view, the row added by the aborted nested transaction is still visible. Once changes are committed, it is evident that each nested transaction has its own status.

### Error Handling and Atomicity
If an error occurs during an operation, the transaction is considered aborted, and no further operations are allowed within it. Attempting to commit such a transaction will result in a rollback. This behavior ensures atomicity, preventing partial access to changes made before the error.

In PostgreSQL's interactive terminal (psql), there is a mode that allows continuing a transaction after an error by implicitly rolling back to a savepoint before each command. This mode is not enabled by default due to the significant overhead associated with setting savepoints.


# 4. Data Snapshots
## 4.1. What is a Data Snapshot?
Physically, data pages can contain multiple versions of the same row, but each transaction should see at most one of them. All the versions of different rows observed by a transaction form a data snapshot. A snapshot ensures a consistent view of the data in the ACID sense at a specific point in time, containing only the most up-to-date data committed at the moment of its creation.

To ensure isolation, each transaction works with its own snapshot. Different transactions see different, yet consistent (at different points in time) data.

### Isolation Levels and Snapshots
- Read Committed Level: A snapshot is created at the beginning of each transaction's statement and remains active for the duration of that statement.

- Repeatable Read and Serializable Levels: A snapshot is created once at the beginning of the first statement in the transaction and remains active until the end of the transaction.

This mechanism ensures that each transaction has a stable view of the data according to the isolation level it operates under.


## 4.2. Visibility of Row Versions in a Snapshot
A data snapshot is not a physical copy of all necessary row versions. Instead, it is defined by a few numbers, and the visibility of row versions in the snapshot is determined by specific rules.

### Determining Visibility
Whether a row version is visible in a snapshot depends on the xmin and xmax fields of its header (the transaction IDs that created and deleted the version) and the corresponding information bits. The xmin–xmax intervals do not overlap, so each row is represented by at most one version in any snapshot.

### Simplified Visibility Rules
1. Visible Version: A row version is visible if the changes made by the transaction xmin are visible in the snapshot, and the changes made by the transaction xmax are not visible. This means the row version has been created but not yet deleted.

2. Visible Changes: Changes made by a transaction are visible in a snapshot if the transaction was committed before the snapshot was created.

3. Own Changes: A transaction sees its own uncommitted changes in its snapshot.

4. Aborted Transactions: Changes from aborted transactions are not visible in any snapshot.

These rules ensure that each snapshot provides a consistent view of the database at the time of its creation, respecting the ACID properties.


## 4.3. Composition of a Snapshot
### Snapshot Information
In PostgreSQL, a snapshot is not seen as a physical copy of all required row versions. Instead, it is defined by several numbers, and visibility of row versions is determined by specific rules. The system only knows when transactions started (defined by transaction ID) but does not record the exact moment of their completion. Commit time is tracked if the track_commit_timestamp parameter is enabled, but this information is not used for visibility checks.

### Snapshot Data
A snapshot contains:

1. Lower Boundary (xmin): The transaction ID of the oldest active transaction. Transactions with IDs less than xmin are either committed (and their changes are visible) or aborted (and their changes are ignored).
2. Upper Boundary (xmax): One more than the transaction ID of the last committed transaction. Transactions with IDs greater than or equal to xmax are either not finished or do not exist, so their changes are not visible.
3. List of Active Transactions (xip_list): Contains IDs of all active transactions (less than xmax), excluding virtual transactions.
### Visibility Determination
When a snapshot is created, it records the status of transactions at that moment. Without this information, it would be impossible to determine which row versions should be visible in the snapshot. Therefore, PostgreSQL cannot create a snapshot showing consistent data for a past moment since it cannot retrospectively know the transaction statuses.

#### Example Workflow
1. Create a Table and Insert Rows:

    - Transaction 1 inserts a row and remains active.
    - Transaction 2 inserts another row and commits.
2. Create a Snapshot:

    - A new session creates a snapshot and captures the current state.
3. Commit the First Transaction:

    - The first transaction is committed after the snapshot is taken.
4. Modify Rows in a New Transaction:

    - Transaction 3 modifies the second row, creating a new version.
#### Snapshot Composition
When a snapshot is created, it consists of:

- xmin (the oldest active transaction ID),
- xmax (one more than the last committed transaction ID),
- xip_list (list of active transaction IDs).
#### Visibility in Snapshot
- Changes from transactions with xid < xmin are always visible.
- Changes from transactions with xmin ≤ xid < xmax are visible unless they are in xip_list.
#### Example Visibility Analysis
- The first row (transaction 787) is not visible because it is still active.
- The second row (created by transaction 788) is visible since it is committed and falls within the snapshot range.
- The latest version of the second row (modified by transaction 789) is not visible as it was created after the snapshot was taken.

## 4.4. Visibility of Own Changes
#### Complexity of Visibility
Determining the visibility of a transaction's own changes can be complex because there can be situations where only a part of the changes should be visible. For instance, a cursor opened at a certain point in time should not see subsequent changes regardless of the isolation level.

To handle this, the row version header includes a special field displayed in the pseudo-columns cmin and cmax. The cmin column represents the sequence number of the insertion operation within the transaction, and cmax represents the deletion operation. However, to save space in the row header, this is actually a single field. It is rare for a row to be inserted and deleted in the same transaction. If this does happen, a special "combo number" is recorded in the same field, with the servicing process remembering the real cmin and cmax.

## 4.5. Transaction Horizon
### Definition and Importance of Transaction Horizon
The lower boundary of a snapshot (the xmin of the oldest transaction active at the time of its creation) is crucial as it defines the transaction horizon for the transaction using this snapshot.

- Without an Active Snapshot: The horizon of a transaction (such as those with Read Committed isolation level between statements) is defined by its own transaction ID, if assigned.
- Beyond the Horizon: All transactions with IDs less than xmin are guaranteed to be committed, meaning the transaction always sees only the current versions of rows beyond its horizon.
This concept is inspired by the event horizon in physics.

#### Viewing the Transaction Horizon
PostgreSQL tracks the current horizon of transactions for all processes. A transaction can see its own horizon in the pg_stat_activity table.

Virtual transactions, although lacking real transaction IDs, use snapshots similarly to regular transactions and have their own horizons. Virtual transactions without an active snapshot do not have a meaningful horizon and are transparent regarding snapshots and visibility.

#### Database Horizon
The database horizon is determined by taking the horizons of all transactions working with the database and identifying the oldest xmin. This horizon ensures that outdated row versions beyond this point are no longer visible to any transaction and can be safely removed during cleanup. Therefore, the concept of the horizon is essential for practical database maintenance.

#### Implications
- Long-Running Transactions: Transactions with Repeatable Read or Serializable isolation levels that run for extended periods hold the database horizon, preventing cleanup.
- Read Committed Transactions: A Read Committed transaction holds the database horizon whether it is actively executing a statement or idle within the transaction.
- Virtual Read Committed Transactions: These transactions hold the horizon only while executing statements.
#### Best Practices
- Avoid Long Transactions with Active Updates: Mixing active updates (which generate row versions) with long transactions should be avoided to prevent table and index bloat.
By understanding and managing transaction horizons effectively, database performance and maintenance can be optimized.


# 5. In-Page Cleanup and HOT Updates
## 5.1. In-Page Cleanup
#### Conditions for In-Page Cleanup:

- Insufficient Space: A previous update couldn't fit a new row version on the same page.
- Fill Factor Exceeded: The page is filled beyond the fillfactor threshold.
#### Process:

- Insertion and Updates: New rows are inserted only if the page is less than fillfactor full. Remaining space is reserved for updates.
- Cleanup Trigger: Removes row versions not visible in any snapshot, confined to a single page, and performs quickly.
#### Implications:

- Index Pointers: Pointers to removed row versions remain due to index references.
- Visibility Map: Not updated during cleanup; space freed is used for updates, not inserts.
- Page Changes during Reads: A SELECT can trigger cleanup, altering the page structure.

In-page cleanup ensures efficient storage by retaining only necessary row versions and effectively managing free space within pages.

## 5.2. HOT Updates
### Overview
Holding index references for all versions of each row is inefficient because:

- Index Updates: Every row change requires updating all indexes on the table, even if the changed columns aren't indexed.
- Index Cleanup: Indexes accumulate references to historical row versions, which then need to be cleaned up.
### HOT (Heap-Only Tuple) Updates address this inefficiency by optimizing updates when:

- No indexes are created on the table.
- The updated columns are not used in any index.
T- he values of the updated columns in the index do not change.
### HOT Update Conditions
- No Indexes: Any update will be a HOT update.
- Indexes Present: Updates are HOT if the updated columns are not part of any index or their values do not change.
### Mechanism
- Index References: Only the first version of a row has an index entry.
- Row Version Chain: Subsequent versions within the same page are linked via the ctid pointers in their headers.
- HOT Flags: The first version is marked with Heap Hot Updated, and intermediate versions are marked with Heap Only Tuple.
- During an index scan, if the server encounters a Heap Hot Updated row, it follows the chain to find the visible version.

### Practical Example
- Initial State:
    - Create a table with a fillfactor of 75%.
    - Insert and update rows multiple times, creating a chain of versions on the same page.
- Index References:
    - The index holds only one reference to the "head" of the chain.
- Updates and Chains:
    - Multiple updates grow the chain within the same page.
- Performance:
    - HOT update chains do not span multiple pages, ensuring efficient performance without additional page accesses.


## 5.3. In-Page Cleanup for HOT Updates
### Importance of In-Page Cleanup with HOT Updates
In-page cleanup is particularly significant during HOT (Heap-Only Tuple) updates. When updates occur and the fillfactor threshold is exceeded, in-page cleanup ensures efficient use of space while maintaining index integrity.

### HOT Update Mechanism
1. Maintaining the "Head" of HOT Chain:
   - The initial row version referenced by the index must remain in place.
   - Subsequent row versions in the update chain can be freed if there are no external references to them.
2. Double Addressing:
   - The index-referenced pointer (e.g., (0,1)) is redirected to the current head of the update chain.
   - Freed pointers are marked as unused, ensuring no space is wasted.
### Example Process
1. Initial Cleanup:

    - Exceeding the fillfactor triggers cleanup.
    - The head pointer is redirected, and other pointers are freed.
2. Subsequent Updates:

    - Each new update adds to the HOT chain within the page.
    - Further updates trigger additional cleanups, maintaining efficient space use.


## 5.4. Breaking the HOT Chain
#### Scenario of HOT Chain Break
When a page runs out of free space to store a new row version, the HOT chain breaks. This necessitates placing the new row version on a different page, requiring a new index reference.

## 5.5. In-Page Cleanup of Indexes

### Index In-Page Cleanup

In-page cleanup also applies to index pages in B-trees. This cleanup is needed when an index page is full and must be split. Cleanup helps delay splitting by removing unnecessary entries.

### Types of Removable Index Entries

1. **Dead Entries:**
   - Entries marked as dead during index scans, referencing row versions not visible in any snapshot or that no longer exist.

2. **Multiple Versions:**
   - Entries referencing different versions of the same row, especially if the updated column is indexed. These versions become obsolete quickly.

### Efficiency Considerations

- **Visibility Check:**
   - Before splitting, check the visibility of certain index entries likely to be removable. This reduces unnecessary index page splits, minimizing index bloat.


# 6. Cleanup and Auto-Vacuum

## 6.1. Manual Cleanup

In-page cleanup is fast but only frees some space within a single table or index page. Full cleanup is performed by the `VACUUM` command, which processes the entire table, removing unnecessary row versions and their index references.

- **Parallel Execution:** Cleanup runs alongside other database activities, allowing normal read and write operations on the table and indexes (though commands like `CREATE INDEX` or `ALTER TABLE` cannot run simultaneously).

- **Visibility Map:** To avoid scanning unnecessary pages, the visibility map is used. Pages marked in the map are skipped since they contain only current row versions. Cleanup updates the visibility map if pages end up with only versions beyond the database horizon.

- **Free Space Map:** Cleanup also updates the free space map to reflect the newly available space in pages.

### Example Process

1. **Create Table and Index:**
   - Create a table with automatic vacuuming disabled for experimental control.
   - Create an index on the table.

2. **Insert and Update Rows:**
   - Add a row and perform updates, creating multiple versions.

3. **Check Initial State:**
   - Verify the presence of multiple row versions and their index references.

4. **Perform Cleanup:**
   - Run the `VACUUM` command to remove dead row versions.

5. **Post-Cleanup State:**
   - Confirm that only the current row version remains, and freed pointers are marked as unused.
   - Verify the visibility map indicates that all row versions on the page are visible in all snapshots.

By using `VACUUM`, you can ensure that your database remains efficient by cleaning up old row versions and maintaining proper indexing.


## 6.2. Revisiting the Database Horizon

Cleanup determines which row versions are dead using the database horizon. This is crucial, so let's revisit this concept.

### Example Process

1. **Initial Setup:**
   - Truncate the table and insert a row, then perform an update.
   - Start a second transaction to maintain the database horizon.

2. **Perform Updates:**
   - Execute another update in the first session, creating three row versions.

3. **Check State After VACUUM:**
   - Run `VACUUM` and check the state. The cleanup keeps the second version because of the active transaction holding the horizon.

4. **Examine Transaction Horizon:**
   - Check the horizon defined by the open transaction.

5. **Run VACUUM with VERBOSE:**
   - Use `VACUUM VERBOSE` to see details about what happens during cleanup.
   - Notice that one dead row version is not removable due to the horizon.

6. **Complete Transaction and Clean Again:**
   - Commit the second transaction.
   - Run `VACUUM` again to remove the dead row version that is now beyond the horizon.

### Key Points

- **Horizon Impact:** Active transactions can prevent the cleanup of certain row versions.
- **Verbose Output:** Using `VACUUM VERBOSE` provides insights into the cleanup process, including which versions are kept or removed based on the horizon.

This detailed management of row versions ensures database consistency and efficiency.