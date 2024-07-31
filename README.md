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
    <version>2024.07.31</version>
</dependency>
```

## Configuration File

Refer src/main/resources/synclite_logger.conf file for all available configuration options for SyncLite Logger. Refer "SyncLite Logger Configuration" section in the documentation at https://www.synclite.io/resources/documentation for more details about all configuration options. 

## Application Code Samples (SQL API)

### 1. SQLite : 

SQLite device (aka transcational device) supports all database operations as supported by SQLite and performs transactional logging of all the DDL and DML operations performed by the application. It empowers developers to build use cases such as native SQL (hot) hot data stores, SQL application caches, edge enablement of cloud databases, building OLTP + OLAP solutions etc.

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


public class TestSQLiteDevice {
	
	public static Path syncLiteDBPath;
	public static void appStartup() throws SQLException, ClassNotFoundException {
		syncLiteDBPath = Path.of(System.getProperty("user.home"), "synclite", "db");
		Class.forName("io.synclite.logger.SQLite");
		Path dbPath = syncLiteDBPath.resolve("t.db");
		SQLite.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
	}	
	
	public void myAppBusinessLogic() throws SQLException {
		//
		//Some application business logic
		//
		//Perform some database operations		
		try (Connection conn = DriverManager.getConnection("jdbc:synclite_sqlite:" + syncLiteDBPath.resolve("t.db"))) {
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
		SQLite.closeDevice(Path.of("t.db"));
		//You can also close all open databases in a single SQL : CLOSE ALL DATABASES
	}	
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		appStartup();
		TestSQLiteDevice testApp = new TestSQLiteDevice();
		testApp.myAppBusinessLogic();
	}

}

```
#### Python   

```
import jaydebeapi

props = {
  "config": "synclite_logger.conf",
  "device-name" : "sqlite1"
}
conn = jaydebeapi.connect("io.synclite.logger.SQLite",
                           "jdbc:synclite_sqlite:c:\\synclite\\python\\data\\t.db",
                           props,
                           "synclite-logger-<version>.jar",)

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
curs.execute("close database c:\\synclite\\python\\data\\t.db");

#You can also close all open databases in a single SQL : CLOSE ALL DATABASES
```

### 2. DuckDB : 

DuckDB device supports all SQL operations, performs transactional logging of all the DDL and DML operations performed by the application. It empowersdevelopers to build use cases such as native SQL (hot) hot data stores, SQL application caches, edge enablement of cloud databases, building OLTP + OLAP solutions etc.

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


public class TestDuckDBDevice {
	
	public static Path syncLiteDBPath;
	public static void appStartup() throws SQLException, ClassNotFoundException {
		syncLiteDBPath = Path.of(System.getProperty("user.home"), "synclite", "db");
		Class.forName("io.synclite.logger.DuckDB");
		Path dbPath = syncLiteDBPath.resolve("t.db");
		DuckDB.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
	}	
	
	public void myAppBusinessLogic() throws SQLException {
		//
		//Some application business logic
		//
		//Perform some database operations		
		try (Connection conn = DriverManager.getConnection("jdbc:synclite_duckdb:" + syncLiteDBPath.resolve("t.db"))) {
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
		DuckDB.closeDevice(Path.of("t.db"));
		//You can also close all open databases in a single SQL : CLOSE ALL DATABASES
	}	
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		appStartup();
		TestDuckDBDevice testApp = new TestDuckDBDevice();
		testApp.myAppBusinessLogic();
	}

}

```
#### Python   

```
import jaydebeapi

props = {
  "config": "synclite_logger.conf",
  "device-name" : "sqlite1"
}
conn = jaydebeapi.connect("io.synclite.logger.DuckDB",
                           "jdbc:synclite_duckdb:c:\\synclite\\python\\data\\t.db",
                           props,
                           "synclite-logger-<version>.jar",)

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
curs.execute("close database c:\\synclite\\python\\data\\t.db");

#You can also close all open databases in a single SQL : CLOSE ALL DATABASES
```

### 3. Apache Derby : 

Apache Derby device supports all SQL operations, performs transactional logging of all the DDL and DML operations performed by the application. It empowers developers to build use cases such as native SQL (hot) hot data stores, SQL application caches, edge enablement of cloud databases, building OLTP + OLAP solutions etc.

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


public class TestDerbyDevice {
	
	public static Path syncLiteDBPath;
	public static void appStartup() throws SQLException, ClassNotFoundException {
		syncLiteDBPath = Path.of(System.getProperty("user.home"), "synclite", "db");
		Class.forName("io.synclite.logger.Derby");
		Path dbPath = syncLiteDBPath.resolve("t.db");
		Derby.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
	}	
	
	public void myAppBusinessLogic() throws SQLException {
		//
		//Some application business logic
		//
		//Perform some database operations		
		try (Connection conn = DriverManager.getConnection("jdbc:synclite_derby:" + syncLiteDBPath.resolve("t.db"))) {
			try (Statement stmt = conn.createStatement()) { 
				//Example of executing a DDL : CREATE TABLE. 
				//You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
				stmt.execute("CREATE TABLE feedback(rating INT, comment TEXT)");
				
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
		Derby.closeDevice(Path.of("t.db"));
		//You can also close all open databases in a single SQL : CLOSE ALL DATABASES
	}	
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		appStartup();
		TestDerbyDevice testApp = new TestDerbyDevice();
		testApp.myAppBusinessLogic();
	}

}

```
#### Python   

```
import jaydebeapi

props = {
  "config": "synclite_logger.conf",
  "device-name" : "derby1"
}
conn = jaydebeapi.connect("io.synclite.logger.Derby",
                           "jdbc:synclite_derby:c:\\synclite\\python\\data\\t.db",
                           props,
                           "synclite-logger-<version>.jar",)

curs = conn.cursor()

#Example of executing a DDL : CEATE TABLE.
#You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
curs.execute('CREATE TABLE feedback(rating INT, comment TEXT)')

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
curs.execute("close database c:\\synclite\\python\\data\\t.db");

#You can also close all open databases in a single SQL : CLOSE ALL DATABASES
```

### 4. H2 : 

H2 device supports all SQL operations, performs transactional logging of all the DDL and DML operations performed by the application. It empowers developers to build use cases such as native SQL (hot) hot data stores, SQL application caches, edge enablement of cloud databases, building OLTP + OLAP solutions etc.

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


public class TestH2Device {
	
	public static Path syncLiteDBPath;
	public static void appStartup() throws SQLException, ClassNotFoundException {
		syncLiteDBPath = Path.of(System.getProperty("user.home"), "synclite", "db");
		Class.forName("io.synclite.logger.H2");
		Path dbPath = syncLiteDBPath.resolve("t.db");
		H2.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
	}	
	
	public void myAppBusinessLogic() throws SQLException {
		//
		//Some application business logic
		//
		//Perform some database operations		
		try (Connection conn = DriverManager.getConnection("jdbc:synclite_h2:" + syncLiteDBPath.resolve("t.db"))) {
			try (Statement stmt = conn.createStatement()) { 
				//Example of executing a DDL : CREATE TABLE. 
				//You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
				stmt.execute("CREATE TABLE feedback(rating INT, comment TEXT)");
				
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
		H2.closeDevice(Path.of("t.db"));
		//You can also close all open databases in a single SQL : CLOSE ALL DATABASES
	}	
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		appStartup();
		TestH2Device testApp = new TestH2Device();
		testApp.myAppBusinessLogic();
	}

}

```
#### Python   

```
import jaydebeapi

props = {
  "config": "synclite_logger.conf",
  "device-name" : "h2device1"
}
conn = jaydebeapi.connect("io.synclite.logger.H2",
                           "jdbc:synclite_h2:c:\\synclite\\python\\data\\t.db",
                           props,
                           "synclite-logger-<version>.jar",)

curs = conn.cursor()

#Example of executing a DDL : CEATE TABLE.
#You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
curs.execute('CREATE TABLE feedback(rating INT, comment TEXT)')

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
curs.execute("close database c:\\synclite\\python\\data\\t.db");

#You can also close all open databases in a single SQL : CLOSE ALL DATABASES
```

### 5. HyperSQL : 

HyperSQL device supports all SQL operations, performs transactional logging of all the DDL and DML operations performed by the application. It empowers developers to build use cases such as native SQL (hot) hot data stores, SQL application caches, edge enablement of cloud databases, building OLTP + OLAP solutions etc.

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


public class TestHyperSQLDevice {
	
	public static Path syncLiteDBPath;
	public static void appStartup() throws SQLException, ClassNotFoundException {
		syncLiteDBPath = Path.of(System.getProperty("user.home"), "synclite", "db");
		Class.forName("io.synclite.logger.HyperSQL");
		Path dbPath = syncLiteDBPath.resolve("t.db");
		HyperSQL.initialize(dbPath, syncLiteDBPath.resolve("synclite_logger.conf"));
	}	
	
	public void myAppBusinessLogic() throws SQLException {
		//
		//Some application business logic
		//
		//Perform some database operations		
		try (Connection conn = DriverManager.getConnection("jdbc:synclite_hsqldb:" + syncLiteDBPath.resolve("t.db"))) {
			try (Statement stmt = conn.createStatement()) { 
				//Example of executing a DDL : CREATE TABLE. 
				//You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
				stmt.execute("CREATE TABLE feedback(rating INT, comment TEXT)");
				
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
		HyperSQL.closeDevice(Path.of("t.db"));
		//You can also close all open databases in a single SQL : CLOSE ALL DATABASES
	}	
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		appStartup();
		TestH2Device testApp = new TestH2Device();
		testApp.myAppBusinessLogic();
	}

}

```
#### Python   

```
import jaydebeapi

props = {
  "config": "synclite_logger.conf",
  "device-name" : "hsqldb1"
}
conn = jaydebeapi.connect("io.synclite.logger.HyperSQL",
                           "jdbc:synclite_hsqldb:c:\\synclite\\python\\data\\t.db",
                           props,
                           "synclite-logger-<version>.jar",)

curs = conn.cursor()

#Example of executing a DDL : CEATE TABLE.
#You can execute other DDL operations : DROP TABLE, ALTER TABLE, RENAME TABLE.
curs.execute('CREATE TABLE feedback(rating INT, comment TEXT)')

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
curs.execute("close database c:\\synclite\\python\\data\\t.db");

#You can also close all open databases in a single SQL : CLOSE ALL DATABASES
```

# Deploying SyncLite Consolidator

Refer https://hub.docker.com/r/syncliteio/synclite-consolidator for setting up SyncLite Consolidator.


# Support

Contact support@synclite.io for support and feedback.
