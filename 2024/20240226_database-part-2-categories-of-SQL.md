# Database - Part 2: Categories of SQL

1. DDL – Data Definition Language: actually consists of the SQL commands that can be used to define the database schema. It simply deals with descriptions of the database schema and is used to create and modify the structure of database objects in the database. DDL is a set of SQL commands used to create, modify, and delete database structures but not data. These commands are normally not used by a general user, who should be accessing the database via an application.
   1. `CREATE`: This command is used to create the database or its objects (like table, index, function, views, store procedure, and triggers).
   2. `DROP`: This command is used to delete objects from the database.
   3. `ALTER`: This is used to alter the structure of the database.
   4. `TRUNCATE`: This is used to remove all records from a table, including all spaces allocated for the records are removed.
   5. `COMMENT`: This is used to add comments to the data dictionary.
   6. `RENAME`: This is used to rename an object existing in the database.
2. DQL – Data Query Language statements are used for performing queries on the data within schema objects. The purpose of the DQL Command is to get some schema relation based on the query passed to it. We can define DQL as follows it is a component of SQL statement that allows getting data from the database and imposing order upon it. It includes the SELECT statement. This command allows getting the data out of the database to perform operations with it.
   1. `SELECT`: It is used to retrieve data from the database.
3. DML – Data Manipulation Language: The SQL commands that deal with the manipulation of data present in the database belong to DML or Data Manipulation Language and this includes most of the SQL statements. It is the component of the SQL statement that controls access to data and to the database. Basically, DCL statements are grouped with DML statements.
   1. `INSERT`: It is used to insert data into a table.
   2. `UPDATE`: It is used to update existing data within a table.
   3. `DELETE`: It is used to delete records from a database table.
   4. `LOCK`: Table control concurrency.
   5. `CALL`: Call a PL/SQL or JAVA subprogram.
   6. `EXPLAIN PLAN`: It describes the access path to data.
4. DCL – Data Control Language: DCL includes commands such as GRANT and REVOKE which mainly deal with the rights, permissions, and other controls of the database system. 
   1. `GRANT`: This command gives users access privileges to the database.
   2. `REVOKE`: This command withdraws the user’s access privileges given by using the GRANT command.
5. TCL – Transaction Control Language: Transactions group a set of tasks into a single execution unit. Each transaction begins with a specific task and ends when all the tasks in the group are successfully completed. If any of the tasks fail, the transaction fails. Therefore, a transaction has only two results: success or failure
   1. `BEGIN`: Opens a Transaction.
   2. `COMMIT`: Commits a Transaction.
   3. `ROLLBACK`: Rollbacks a transaction in case of any error occurs.
   4. `SAVEPOINT`: Sets a save point within a transaction.