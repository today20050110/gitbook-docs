# 13.5. 特別注意[^1]

Some DDL commands, currently only[TRUNCATE](https://www.postgresql.org/docs/10/static/sql-truncate.html)and the table-rewriting forms of[ALTER TABLE](https://www.postgresql.org/docs/10/static/sql-altertable.html), are not MVCC-safe. This means that after the truncation or rewrite commits, the table will appear empty to concurrent transactions, if they are using a snapshot taken before the DDL command committed. This will only be an issue for a transaction that did not access the table in question before the DDL command started — any transaction that has done so would hold at least an`ACCESS SHARE`table lock, which would block the DDL command until that transaction completes. So these commands will not cause any apparent inconsistency in the table contents for successive queries on the target table, but they could cause visible inconsistency between the contents of the target table and other tables in the database.

Support for the Serializable transaction isolation level has not yet been added to Hot Standby replication targets \(described in[Section 26.5](https://www.postgresql.org/docs/10/static/hot-standby.html)\). The strictest isolation level currently supported in hot standby mode is Repeatable Read. While performing all permanent database writes within Serializable transactions on the master will ensure that all standbys will eventually reach a consistent state, a Repeatable Read transaction run on the standby can sometimes see a transient state that is inconsistent with any serial execution of the transactions on the master.

---



[^1]:  [PostgreSQL: Documentation: 10: 13.5. Caveats](https://www.postgresql.org/docs/10/static/mvcc-caveats.html)
