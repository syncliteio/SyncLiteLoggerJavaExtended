# SyncLite - Build Anything Sync Anywhere

SyncLite is a no-code, real-time, relational data consolidation platform empowering developers to rapidly build data-intensive applications for desktops, edge devices, smartphones, with the capability to enable in-app data management, in-app analytics and perform real-time data consolidation from numerous application instances into one or more databases, data warehouses, or data lakes of your choice.

```
{Edge/Desktop/Phone Apps} + {SyncLite Logger + Embedded databases} ------> {Staging Storage} ------> { SyncLite Consolidator} -----> {Destination DB/DW/DataLakes}
```

For more details, refer : https://github.com/syncliteio/SyncLiteLoggerJava

Extended SyncLite Logger enables end-to-end real-time (CDC) data replication/consonolidation from various embedded databases including SQLite, DuckDB, Apache Derby and H2 to a wide range of database, data warehouse and data lakes.

For more details, check out 
Website : https://www.synclite.io
YouTube : https://www.youtube.com/@syncliteplatform


# SyncLite Components

```SyncLite Logger(and Syncite Extended Logger)``` : SyncLite Logger is a lighweight JDBC wrapper built on top of SQLite (extended to other embedded databases such as DuckDB, Apache Derby and H2), providing a SQL interface over JDBC for user applications, enabling them for in-app data management/analytics using one or more supported embedded databases, while logging all the SQL transactional activity into log files and shipping them to one of more configured staging storages like SFTP/S3/MinIO/Kafka/GoogleDrive/MSOneDrive/NFS etc. 

```SyncLite Consolidator``` : SyncLite Consolidator is a Java application deployed on an on-premise host or a cloud VM is configured to scan thousands of SyncLite devices/databases and their logs continously from the configured staging storage which are uploaded by numerous edge/desktop applications, performs real-time transactional data replication/consolidation into one or more configured databases, data warehouses or data lakes of user's choice. Refer : https://hub.docker.com/r/syncliteio/synclite-consolidator

(Note: Refer https://github.com/syncliteio/SyncLiteLoggerJava for other components).
# Using Extended SyncLite Logger

This repository has been created to distribute the Extended SyncLite logger jar file as updated in src/main/resources. You can use the following maven dependency in your edge/desktop applications for creating and operating edge databases/devices.

```
<dependency>
    <groupId>io.synclite</groupId>
    <artifactId>synclite-logger-extended</artifactId>
    <version>2024.08.08</version>
</dependency>
```

## Configuration File

Refer src/main/resources/synclite_logger.conf file for all available configuration options for SyncLite Logger. Refer "SyncLite Logger Configuration" section in the documentation at https://www.synclite.io/resources/documentation for more details about all configuration options. 

## Application Code Samples (SQL API)

### Transactional Devices : 

Transactional devices (SQLite, DuckDB, Apache Derby, H2, HyperSQL) support all database operations and performs transactional logging of all the DDL and DML operations performed by the application. These empower developers to build use cases such as building data-intensive sync-ready applications, Edge + Cloud GenAI search and RAG applications, native SQL (hot) hot data stores, SQL application caches, edge enablement of cloud databases etc.

#### Java
```
package testApp;

import java.nio.file.Path;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.sql.Statement;
import io.synclite.logger.*;


public class TestTransactionalDevice {
	
	public static Path syncLiteDBPath;
	public static void appStartup() throws SQLException, ClassNotFoundException {
		syncLiteDBPath = Path.of(System.getProperty("user.home"), "synclite", "db");
		Class.forName("io.synclite.logger.SQLite");
		//
		//////////////////////////////////////////////////////
		//For other types of transactional devices : 
		//DuckDB : Class.forName("io.synclite.logger.DuckDB");
		//Apache Derby : Class.forName("io.synclite.logger.Derby");
		//H2 : Class.forName("io.synclite.logger.H2");
		//HyperSQL : Class.forName("io.synclite.logger.HyperSQL");
		//////////////////////////////////////////////////////
		//

		Path dbPath = syncLiteDBPath.resolve("test_tran.db");
		SQLite.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
		//
		//////////////////////////////////////////////////////
		//For other types of transactional devices : 
		//DuckDB : DuckDB.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
		//Apache Derby : Derby.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
		//H2 : H2.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
		//HyperSQL : HyperSQL.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
		//////////////////////////////////////////////////////
		//
	}	
	
	public void myAppBusinessLogic() throws SQLException {
		//
		//Some application business logic
		//
		//Perform some database operations		
		try (Connection conn = DriverManager.getConnection("jdbc:synclite_sqlite:" + syncLiteDBPath.resolve("test_sqlite.db"))) {
			//
		        //////////////////////////////////////////////////////////////////
			//For other types of transactional devices use following connection strings :
			//For DuckDB : jdbc:synclite_duckdb:<db_path>
			//For Apache Derby : jdbc:synclite_derby:<db_path>
			//For H2 : jdbc:synclite_h2:<db_path>
			//For HyperSQL : jdbc:synclite_hsqldb:<db_path>
			///////////////////////////////////////////////////////////////////
			//
			try (Statement stmt = conn.createStatement()) { 
				//Example of executing a DDL : CREATE TABLE. 
				//You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
				stmt.execute("CREATE TABLE IF NOT EXISTS feedback(rating INT, comment TEXT)");
				
				//Example of performing INSERT
				stmt.execute("INSERT INTO feedback VALUES(3, 'Good product')");				
			}
			
			//Example of setting Auto commit OFF to implement transactional semantics
			conn.setAutoCommit(false);
			try (Statement stmt = conn.createStatement()) { 
				//Example of performing basic DML operations INSERT/UPDATE/DELETE
				stmt.execute("UPDATE feedback SET comment = 'Better product' WHERE rating = 3");
				stmt.execute("INSERT INTO feedback VALUES (1, 'Poor product')");
				stmt.execute("DELETE FROM feedback WHERE rating = 1");
			}
			conn.commit();
			conn.setAutoCommit(true);
			
			//Example of Prepared Statement functionality for bulk insert.			
			try(PreparedStatement pstmt = conn.prepareStatement("INSERT INTO feedback VALUES(?, ?)")) {
				pstmt.setInt(1, 4);
				pstmt.setString(2, "Excellent Product");
				pstmt.addBatch();
				
				pstmt.setInt(1, 5);
				pstmt.setString(2, "Outstanding Product");
				pstmt.addBatch();
				
				pstmt.executeBatch();			
			}
		}
		//Close SyncLite database/device cleanly.
		SQLite.closeDevice(Path.of("test_sqlite.db"));
		//
		///////////////////////////////////////////////////////
		//For other types of transactional devices :
		//DuckDB : DuckDB.closeDevice
		//Apache Derby : Derby.closeDevice
		//H2 : H2.closeDevice
		//HyperSQL : HyperSQL.closeDevice
		//////////////////////////////////////////////////////
		//
		//You can also close all open databases in a single SQL : CLOSE ALL DATABASES
	}	
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		appStartup();
		TestTransactionalDevice testApp = new TestTransactionalDevice();
		testApp.myAppBusinessLogic();
	}
}

```
#### Python   

```
import jaydebeapi

props = {
  "config": "synclite_logger.conf",
  "device-name" : "tran1"
}
conn = jaydebeapi.connect("io.synclite.logger.SQLite",
                           "jdbc:synclite_duckdb:c:\\synclite\\python\\data\\test_sqlite.db",
                           props,
                           "synclite-logger-<version>.jar",)
#//
#////////////////////////////////////////////////////////////////
#For other types of transactional devices use following are the class names and connection strings :
#For DuckDB - Class : io.synclite.logger.DuckDB, Connection String : jdbc:synclite_duckdb:<db_path>
#For Apache Derby - Class : io.synclite.logger.Derby, Connection String : jdbc:synclite_derby:<db_path>
#For H2 - Class : io.synclite.logger.H2, Connection String : jdbc:synclite_h2:<db_path>
#For HyperSQL - Class : io.synclite.logger.HyperSQL, Connection String : jdbc:synclite_hsqldb:<db_path>
#/////////////////////////////////////////////////////////////////
#//

curs = conn.cursor()

#Example of executing a DDL : CEATE TABLE.
#You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
curs.execute('CREATE TABLE IF NOT EXISTS feedback(rating INT, comment TEXT)')

#Example of performing basic DML operations INSERT/UPDATE/DELETE
curs.execute("insert into feedback values (3, 'Good product')")

#Example of setting Auto commit OFF to implement transactional semantics
conn.jconn.setAutoCommit(False)
curs.execute("update feedback set comment = 'Better product' where rating = 3")
curs.execute("insert into feedback values (1, 'Poor product')")
curs.execute("delete from feedback where rating = 1")
conn.commit()
conn.jconn.setAutoCommit(True)


#Example of Prepared Statement functionality for bulk insert.
args = [[4, 'Excellent product'],[5, 'Outstanding product']]
curs.executemany("insert into feedback values (?, ?)", args)

#Close SyncLite database/device cleanly.
curs.execute("close database c:\\synclite\\python\\data\\test_sqlite.db");

#You can also close all open databases in a single SQL : CLOSE ALL DATABASES
```

### Appender Devices :

Appender devices (SQLiteAppender, DuckDBAppender, DerbyAppender, H2Appender, HyperSQLAppender) supports all DDL operations and Prepared Statement based INSERT operations and are highly optimized for high speed concurrent batched data ingestion, performing logging of the ingested data. Unlike transactional devices, appender devices only allow INSERT DML operations (UPDATE and DELETE are not supported). Appender devices empower developers to build high volume streaming applications enabled with last mile data integration from thousands of edge points into centralized database destinations as well as in-app analytics by enabling fast read access to ingested data from the underlying local embedded databases storing the ingested streamed data.

#### Java

```
package testApp;

import java.nio.file.Path;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.sql.Statement;
import io.synclite.logger.*;

public class TestAppenderDevice {
	public static Path syncLiteDBPath;

	public static void appStartup() throws SQLException, ClassNotFoundException {
		syncLiteDBPath = Path.of(System.getProperty("user.home"), "synclite", "db");
		Class.forName("io.synclite.logger.SQLiteAppender");
		//
		//////////////////////////////////////////////////////
		//For other types of appender devices : 
		//DuckDB : Class.forName("io.synclite.logger.DuckDBAppender");
		//Apache Derby : Class.forName("io.synclite.logger.DerbyAppender");
		//H2 : Class.forName("io.synclite.logger.H2Appender");
		//HyperSQL : Class.forName("io.synclite.logger.HyperSQLAppender");
		//////////////////////////////////////////////////////
		//
		Path dbPath = syncLiteDBPath.resolve("test_appender.db");
		SQLiteAppender.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
	}

	public void myAppBusinessLogic() throws SQLException {
		//
		// Some application business logic
		//
		// Perform some database operations
		try (Connection conn = DriverManager.getConnection("jdbc:synclite_sqlite_appender:" + syncLiteDBPath.resolve("test_appender.db"))) {
			//
		        //////////////////////////////////////////////////////////////////
			//For other types of appender devices use following connection strings :
			//For DuckDBAppender : jdbc:synclite_duckdb_appender:<db_path>
			//For DerbyAppender : jdbc:synclite_derby_appender:<db_path>
			//For H2Appender : jdbc:synclite_h2_appender:<db_path>
			//For HyperSQLAppender : jdbc:synclite_hsqldb_appender:<db_path>
			///////////////////////////////////////////////////////////////////
			//
			try (Statement stmt = conn.createStatement()) {
				// Example of executing a DDL : CREATE TABLE.
				// You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
				stmt.execute("CREATE TABLE IF NOT EXISTS feedback(rating INT, comment TEXT)");
			}

			//
			// Example of Prepared Statement functionality for bulk insert.
			// Note that Appender Devices allows all DDL operations, INSERT INTO DML operations (UPDATES and DELETES are not allowed) and SELECT queries.
			//
			try (PreparedStatement pstmt = conn.prepareStatement("INSERT INTO feedback VALUES(?, ?)")) {
				pstmt.setInt(1, 4);
				pstmt.setString(2, "Excellent Product");
				pstmt.addBatch();

				pstmt.setInt(1, 5);
				pstmt.setString(2, "Outstanding Product");
				pstmt.addBatch();

				pstmt.executeBatch();
			}
		}
		// Close SyncLite database/device cleanly.
		SQLiteAppender.closeDevice(Path.of("test_appender.db"));
		//
		///////////////////////////////////////////////////////
		//For other types of appender devices :
		//DuckDBAppender : DuckDBAppender.closeDevice
		//DerbyAppender : DerbyAppender.closeDevice
		//H2Appender : H2Appender.closeDevice
		//HyperSQLAppender : HyperSQLAppender.closeDevice
		//////////////////////////////////////////////////////
		//
		// You can also close all open databases/devices in a single SQL : CLOSE ALL
		// DATABASES
	}

	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		appStartup();
		TestAppenderDevice testApp = new TestAppenderDevice();
		testApp.myAppBusinessLogic();
	}

}
```

#### Python

```
import jaydebeapi
props = {
  "config": "synclite_logger.conf",
  "device-name" : "appender1"
}
conn = jaydebeapi.connect("io.synclite.logger.SQLiteAppender",
                           "jdbc:synclite_sqlite_appender:c:\\synclite\\python\\data\\test_appender.db",
                           props,
                           "synclite-logger-<version>.jar",)
#//
#////////////////////////////////////////////////////////////////
#For other types of appender devices use following are the class names and connection strings :
#For DuckDBAppender - Class : io.synclite.logger.DuckDBAppender, Connection String : jdbc:synclite_duckdb_appender:<db_path>
#For DerbyAppender - Class : io.synclite.logger.DerbyAppender, Connection String : jdbc:synclite_derby_appender:<db_path>
#For H2Appender - Class : io.synclite.logger.H2Appender, Connection String : jdbc:synclite_h2_appender:<db_path>
#For HyperSQLAppender - Class : io.synclite.logger.HyperSQLAppender, Connection String : jdbc:synclite_hsqldb_appender:<db_path>
#/////////////////////////////////////////////////////////////////
#//

curs = conn.cursor()

#Example of executing a DDL : CREATE TABLE.
#You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
curs.execute('CREATE TABLE IF NOT EXISTS feedback(rating INT, comment TEXT)')

#Example of Prepared Statement functionality for bulk insert.
args = [[4, 'Excellent product'],[5, 'Outstanding product']]
curs.executemany("insert into feedback values (?, ?)", args)

#Close SyncLite database/device cleanly.
curs.execute("close database c:\\synclite\\python\\data\\test_appender.db");

#You can also close all open databases in a single SQL : CLOSE ALL DATABASES
```


# Deploying SyncLite Consolidator

Refer https://hub.docker.com/r/syncliteio/synclite-consolidator for setting up SyncLite Consolidator.


# Support

Contact support@synclite.io for support and feedback.
