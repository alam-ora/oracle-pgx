## Create the Property Graph

A property graph consists of a set of objects or vertices, and a set of arrows or edges connecting the objects. Vertices and edges can have multiple properties, which are represented as key-value pairs.

Each vertex has a unique identifier and can have:

* A set of outgoing edges
* A set of incoming edges
* A collection of properties

Each edge has a unique identifier and can have:

* An outgoing vertex
* An incoming vertex
* A text label that describes the relationship between the two vertices
* A collection of properties

Now that the tables are created and populated let's create a graph representation of the dataset.

## **STEP 1** : Install Oracle Graph Server

Oracle Graph Server and Client is a software package that works with Oracle Database. It includes the in-memory analytics server (PGX) and client libraries required to work with the Property Graph feature in Oracle Database. The package simplifies installation and provides access to the latest graph features and updates.

1. In the previous SSH connection logged in as the **oracle** user, create a new folder to stage the Oracle Graph software.

````
<copy>mkdir -p /home/oracle/software
cd /home/oracle/software</copy>
````

2. Download the Graph Server using a Pre-Authenticated Request (PAR) URL as follows.

````
<copy>wget https://objectstorage.us-phoenix-1.oraclecloud.com/n/oraclepartnersas/b/oracle_pg/o/oracle-graph-20.2.0.x86_64.rpm</copy>
````

3. Install the Graph Server.

````
<copy>sudo yum install -y oracle-graph-20.2.0.x86_64.rpm</copy>
````

[PIC]

4. The Graph server is a web application that listens on port 7007 by default. The Linux firewall will block this port, unless specifically opened by a rule, using the following commands:

````
<copy>sudo firewall-cmd --permanent --zone=public --add-port=7007/tcp</copy>
````
````
<copy>sudo firewall-cmd --reload</copy>
````

[PIC]

5. Edit the graph server config file **`/etc/oracle/graph/server.conf`** using your favorite text editor.

```
<copy>vi /etc/oracle/graph/server.conf</copy>
```

6. Change the line  **`"enable_tls": true`** to  **`"enable_tls": false`**. Save the file and exit.

[PIC]

## **STEP 2** : Start Graph Server

1. In the previous SSH connection logged in as the **oracle** user, ensure **JAVA\_HOME** and **JAVA11\_HOME** environment variables point to **JDK1.8** and **JDK-11**, respectively.

````
<copy>env | grep JAVA</copy>
````

[PIC]

2. As the **`oracle`** user, start the server using **`/opt/oracle/graph/pgx/bin/start-server`**:

```
<copy>/opt/oracle/graph/pgx/bin/start-server</copy>
```

3. Once the Graph server is fully started, you should see the following message displayed:

>INFO: Starting ProtocolHandler ["http-nio-7007"]

[PIC]

> Keep the above SSH session to the Graph server running and start a new session for the remaining lab steps.

## **STEP 3**: Start Graph Client Shell

1. Once the Graph Server is up, open a new SSH connection to the same compute instance and switch user to **`oracle`**.

````
<copy>sudo su - oracle</copy>
````

2. Start a Graph Client JShell session that connects to the Graph server at the default port **7007**.

````
<copy>/opt/oracle/graph/bin/opg-jshell --base_url http://localhost:7007</copy>
````

[PIC]

## **STEP 4**: Create the Property Graph

Once the Graph Client JShell starts, create the property graph by completing the following steps.

### Setup the Database Connection

Setup the database connection using JDBC and the Wallet.

>The Autonomous Database Wallet has been pre-downloaded and extracted to **/home/oracle/wallets/ADB_Wallet** folder on the compute instance.

1. In the JShell prompt, copy/paste the following command replacing the **`<<ADB_Service_Name>>`** with yours, to set the JDBC URL **`"jdbcUrl"`** variable (note the command needs to be in one line).

```
var jdbcUrl = "jdbc:oracle:thin:@<<ADB_Service_Name>>?TNS_ADMIN=/home/oracle/wallets/ADB_Wallet";
```

2. Set the database user for the JDBC connection.

````
<copy>var user = "RETAIL";</copy>
````

3. Set the password for the database user, replacing **`<<Retail_Password>>`** with the **RETAIL** user's password (keeping the quotes **" "** intact)

```
var pass = "<<Retail_Password>>";
```

4. Define the final JDBC connection variable and observe the output is as expected.

```
<copy>var conn = DriverManager.getConnection(jdbcUrl, user, pass);</copy>
// example jshell output
// conn ==> oracle.jdbc.driver.T4CConnection@54d11c70
```

5. Set the connection properties (mainly, set auto commit to false) and create the PGQL connection.

```
<copy>conn.setAutoCommit(false);
var pgql = PgqlConnection.getConnection(conn);</copy>
```

6. Observe the output and verify the connection was successful.

[PIC]

### Create the Property Graph

1. Copy/paste the following command to your JShell session. This will set **cpgStmtStr** variable with the required **CREATE PROPERTY GRAPH** command.

>Note that **Customers** and **Products** become the graph vertices and **Purchases** become the edges.

```
<copy>// create the graph from the existing tables
var cpgStmtStr = "CREATE PROPERTY GRAPH retail " +
"  VERTEX TABLES (" +
"    customers " +
"      LABEL Customer " +
"      PROPERTIES ( " +
"        customer_id AS customer_id " +
"      ) " +
"  , products " +
"      LABEL Product " +
"      PROPERTIES (" +
"        stock_code AS stock_code " +
"      ) " +
"  ) " +
"  EDGE TABLES (" +
"    purchases " +
"      KEY (purchase_id)" +
"      SOURCE KEY(customer_id) REFERENCES customers " +
"      DESTINATION KEY(stock_code) REFERENCES products " +
"      LABEL purchased " +
"      PROPERTIES (" +
"        quantity AS quantity " +
"      , unit_price AS unit_price " +
"      )" +
"  )";</copy>
```

[PIC]

2. Execute the PGQL DDL to first DROP any existing graph of the same name before creating it.

>Ignore the drop error when the graph doesn't exist.

```
// drop any existing graph
<copy>var dropPgStmt = "DROP PROPERTY GRAPH retail";
pgql.prepareStatement(dropPgStmt).execute();</copy>
```

3. Finally, create the graph using **prepareStatement**.

```
<copy>pgql.prepareStatement(cpgStmtStr).execute();</copy>
```
[PIC]

>The create graph process can take 3-4 minutes depending on various factors such as network bandwidth and database load.

## **STEP 4**: Validate the Property Graph

1. Check that the graph was successfully created by observing the output.

[PIC]

2. Copy/paste and run the following statements in the JShell.

3. Copy the following code, paste, and execute it in the JShell.

```
<copy>
// create a helper method for preparing, executing, and printing the results of PGQL statements
Consumer&lt;String&gt; query = q -> { try(var s = pgql.prepareStatement(q)) { s.execute(); s.getResultSet().print(); } catch(Exception e) { throw new RuntimeException(e); } }
</copy>
// sample jshell output
// query ==> $Lambda$583/0x0000000800695c40@65021bb4
```

4. Then copy, paste, and execute the following.

```
<copy>
// query the graph

// what are the edge labels i.e. categories of edges
query.accept("select distinct label(e) from customer_360 match ()-[e]->()");

// what are the vertex types i.e. values of the property named "type"
query.accept("select distinct v.type from customer_360 match (v)-[e]->()");

// how many vertices with each type/category
query.accept("select count(distinct v), v.type from customer_360 match (v) group by v.type");

// how many edges with each label/category
query.accept("select count(e), label(e) from customer_360 match ()-[e]->() group by label(e)");

</copy>
```

[PIC]

## Summary

You have successfully created the property graph representation of sample dataset using Oracle Autonomous Database and Oracle Graph Server. Please proceed to the next lab using the **Lab Contents** menu ![](./images/lab-contents-icon.png) on your right.
