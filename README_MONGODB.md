# Helical Insight - MongoDB Driver Integration (Round 2 Assessment)

## 1. Overview
This document outlines the implementation of **MongoDB database driver and connectivity support** for the open-source Helical Insight application. 

The integration follows the native plugin-and-repository architecture of Helical Insight, mirroring the existing database driver implementations (such as PostgreSQL, MySQL, ClickHouse, and SQLite) to ensure full compatibility without breaking any existing system functionality.

---

## 2. Driver Details

* **Driver Name**: Official MongoDB JDBC Driver (All-in-One)
* **Version**: `2.1.2`
* **Driver Class Name**: `com.mongodb.jdbc.MongoDriver`
* **File Location**: [`server/hi-repository/System/Drivers/mongodb-jdbc-2.1.2-all.jar`](file:///c:/1Only/helicalinsight/server/hi-repository/System/Drivers/mongodb-jdbc-2.1.2-all.jar)
* **Supported Protocols**:
  * Standard standalone / replica set: `jdbc:mongodb://<host>:<port>/<database>`
  * MongoDB Atlas (SRV): `jdbc:mongodb+srv://<host>/<database>`
* **Default Port**: `27017`

---

## 3. Summary of Changes Made

All modifications are modular and additive, preserving the integrity of all existing drivers and core application logic.

### 1. Driver Placement
* **File Added**: [`server/hi-repository/System/Drivers/mongodb-jdbc-2.1.2-all.jar`](file:///c:/1Only/helicalinsight/server/hi-repository/System/Drivers/mongodb-jdbc-2.1.2-all.jar)
* **Purpose**: Helical Insight dynamically scans this folder at startup for JDBC driver `.jar` files matching the regex pattern `.*[Dd]river` to load into the JVM classpath.

### 2. JDBC URL & Port Mapping
* **File Modified**: [`server/hi-repository/System/Admin/databaseDrivers.properties`](file:///c:/1Only/helicalinsight/server/hi-repository/System/Admin/databaseDrivers.properties)
* **Lines Added**:
  ```properties
  # MongoDB
  com.mongodb.jdbc.MongoDriver=jdbc:mongodb://{{hostName}}:{{port}}/{{database}},27017
  type.srv.com.mongodb.jdbc.MongoDriver=jdbc:mongodb+srv://{{hostName}}/{{database}},27017
  ```
* **Purpose**: Defines the default port (`27017`) and the connection URL template used by the frontend datasource modal when auto-generating connection strings for standard and Atlas SRV endpoints.

### 3. Connection Validation / Ping Query
* **File Modified**: [`server/hi-repository/System/Admin/driverDefaultQuery.properties`](file:///c:/1Only/helicalinsight/server/hi-repository/System/Admin/driverDefaultQuery.properties)
* **Line Added**:
  ```properties
  com.mongodb.jdbc.MongoDriver=SELECT 1
  ```
* **Purpose**: Specifies the lightweight validation query executed by Helical Insight when the user clicks **"Test Connection"** in the UI.

### 4. Hibernate SQL Dialect Mapping
* **File Modified**: [`server/hi-repository/System/Admin/sqlDialects.properties`](file:///c:/1Only/helicalinsight/server/hi-repository/System/Admin/sqlDialects.properties)
* **Line Added**:
  ```properties
  com.mongodb.jdbc.MongoDriver=org.hibernate.dialect.MySQLDialect
  ```
* **Purpose**: Associates the driver with a compatible Hibernate dialect for query generation and schema handling.

### 5. SQL Function XML Mapping
* **File Modified**: [`server/hi-repository/System/Admin/sqlFunctionsXmlMapping.properties`](file:///c:/1Only/helicalinsight/server/hi-repository/System/Admin/sqlFunctionsXmlMapping.properties)
* **Line Added**:
  ```properties
  com.mongodb.jdbc.MongoDriver=himongo
  ```
* **Purpose**: Maps MongoDB SQL functions to Helical Insight's built-in `himongo` function definition catalog.

### 6. Dynamic Datasource Listing & UI Status
* **File Modified**: [`server/hi-repository/System/Admin/Static/DataSourcesList.groovy`](file:///c:/1Only/helicalinsight/server/hi-repository/System/Admin/Static/DataSourcesList.groovy)
* **Changes**:
  * Added `"Mongodb"` to the `supportedArray`.
  * Added driver mapping check:
    ```groovy
    if(it.driver=="com.mongodb.jdbc.MongoDriver" || it.driver=="com.helical.mongodb.MongoJdbcDriver") {
        modelJson.name=findDbName="Mongodb"
    }
    ```
  * Registered metadata category and connection provider:
    ```groovy
    } else if (findDbName.equals("Mongodb")) {
        modelJson.categoryName = "No SQL & Big Data"
        modelJson.categoryType = "nosql_bigdata"
        modelJson.type = "global.jdbc"
        modelJson.dataSourceProvider = "tomcat"
    }
    ```
* **Purpose**: Instructs the frontend UI to display the **MongoDB** connection card with an active green checkmark indicator under the **No SQL & Big Data** and **Supported** categories.

### 7. Database Metadata Mapping Template (EFWD)
* **File Created**: [`server/hi-repository/System/Admin/DbConfig/mongodb.efwd`](file:///c:/1Only/helicalinsight/server/hi-repository/System/Admin/DbConfig/mongodb.efwd)
* **Purpose**: Provides data maps (`getCatalog` and `getColumn`) so Helical Insight can read collections and fields from MongoDB when users navigate catalogs and create reports.

---

## 4. Steps to Configure and Use the MongoDB Connection

Follow these steps to configure and verify MongoDB connectivity in the Helical Insight application:

### Step 1: Ensure MongoDB Instance is Running
Make sure your MongoDB server or cluster is accessible:
* **Local instance**: Default port `27017` (e.g., `localhost:27017`).
* **Cloud instance**: [MongoDB Atlas](https://www.mongodb.com/atlas) connection URI.

### Step 2: Start Helical Insight
Start the backend and frontend application servers:
* **Backend**: Run Tomcat / Maven setup (`./scripts/setup-dev.ps1` or deploy `hi-ee.war`).
* **Frontend**: Start the client UI (`cd client && npm start`).

### Step 3: Open Data Sources Tab
1. Open your browser and navigate to the Helical Insight web console (`http://localhost:8080/hi-ee` or your local development port).
2. Click on the **Data Sources** tab in the main navigation.
3. Under the **Supported** or **No SQL & Big Data** section, locate the **MongoDB** card.
4. Verify the **green checkmark** is present on the card, indicating the driver JAR is detected and loaded.

### Step 4: Configure the Connection
1. Click the **MongoDB** card to open the datasource configuration modal.
2. Enter the connection parameters:
   * **Host Name**: `localhost` (or your remote IP / cluster hostname)
   * **Port**: `27017` (pre-filled by default)
   * **Database Name**: Enter your target database name (e.g., `test` or your custom DB)
   * **Data Source Name**: Enter a friendly identifier (e.g., `MongoDB_DS`)
   * **User Name & Password**: Enter credentials if authentication is enabled, or leave empty if using local unauthenticated MongoDB.
3. Click the **"Test Connection"** button. A green success confirmation message will appear.
4. Click **"Save"** to persist the datasource in `globalConnections.xml`.

### Step 5: Create Metadata & Reports
1. Navigate to **Metadata** > **Create**.
2. Select your newly created **MongoDB** datasource.
3. Drag collections (tables) and fields (columns) to create metadata views and generate ad-hoc reports and dashboards.

---

## 5. Verification & Testing Completed

1. **JVM Class Loading**: Verified `com.mongodb.jdbc.MongoDriver` loads successfully under Java 17 via `Class.forName()`.
2. **URL Pattern Acceptance**: Confirmed driver accepts both `jdbc:mongodb://` and `jdbc:mongodb+srv://`.
3. **Repository Validation**: Verified all 6 configuration files follow Helical Insight's exact property patterns.
4. **Non-Breaking Verification**: Confirmed no existing drivers, classes, or database connectors were modified or disabled.
