## Configure the Lab Environment

The lab environment needs additional configuration before you can build the property graph for real-time recommendations.

In this step you will load a retail dataset in the Oracle Autonomous database and install Oracle Graph software in the compute instance, along with other required tools.

## **STEP 1** : Download the Retail Dataset

A dataset of retail transactions is available on [UCI](https://archive.ics.uci.edu/ml/datasets/online+retail) and [Kaggle](https://www.kaggle.com/jihyeseo/online-retail-data-set-from-uci-ml-repo). This dataset contains real-world transactions of customer purchases along with product and customer data - quite suitable for the Graph-based recommendation engine use case.

>According to [UCI Machine Learning Repository](https://archive.ics.uci.edu/ml/datasets/online+retail), "the Online Retail Dataset is a transnational data set which contains all the transactions occurring between 01/12/2010 and 09/12/2011 for a UK-based and registered non-store online retail. The company mainly sells unique all-occasion gifts. Many customers of the company are wholesalers."

1. Using **Cloud Shell** or your SSH tool of choice, start an SSH session and log in to the compute instance using your **Private Key** and **VM\_IP\_Address**:

>Note that the commands that require you to replace the strings enclosed in <<...>> do not have the **Copy** [PIC] button. This is to ensure that you replace the string with the values unique to your environment.

```
ssh -i <<private_key>> opc@<<vm_ip_address>>

```

2. In the SSH session, switch the user to **oracle** and change directory to **/home/oracle/dataset**.

````
<copy>sudo su - oracle
cd /home/oracle/dataset</copy>
````

ADD PIC

3. Download the dataset using **wget** with a direct download URL from UCI.

````
<copy>wget https://archive.ics.uci.edu/ml/machine-learning-databases/00352/Online%20Retail.xlsx -O OnlineRetail.xlsx</copy>
````

4. Once the download completes, convert the file (Excel) to CSV format using the open source **libreoffice**, as the data needs to be in plain text for loading.

````
<copy>libreoffice --headless --convert-to csv OnlineRetail.xlsx</copy>
````

## **STEP 2** : Create the Database Schema

1. In the SSH session to the compute instance and connected as the **oracle** user, start a SQL Plus session and connect as the **ADMIN** user using **ADB\_Admin\_Password** and the autonomous database **Service Name**.

```
sqlplus ADMIN/<<ADB_Admin_Password>>@<<Service_Name>>
```

2. Create the **RETAIL** database user by choosing a suitable **```<<Retail_Password>>```** that follows the [password rules](https://docs.oracle.com/en/cloud/paas/autonomous-data-warehouse-cloud/user/manage-users-admin.html#GUID-B227C664-EBA0-4B5E-B11C-A56B16567C1B)):

```
CREATE USER retail IDENTIFIED BY <<Retail_Password>>;
```

3. Grant the required privileges to the **RETAIL** user.

````
<copy>ALTER USER retail QUOTA UNLIMITED ON data;
GRANT CONNECT, RESOURCE, CREATE VIEW TO retail;
EXIT;
</copy>
````

[[PIC]]

4. Start another SQL Plus session and this time log in as the **RETAIL** user using **```<<Retail_Password>>```**.

```
sqlplus retail/<<Retail_Password>>@<<Service_Name>>
```

4. Create the tables for the dataset.

>Ignore any **ORA-00942: table or view does not exist** errors.

````
<copy>DROP TABLE transactions;
DROP TABLE customers;
DROP TABLE purchases;
DROP TABLE products;
DROP TABLE purchases_distinct;

CREATE TABLE transactions (
  invoice_no VARCHAR2(255)
, stock_code VARCHAR2(255)
, description VARCHAR2(255)
, quantity NUMBER(10)
, invoice_date VARCHAR2(255)
, unit_price NUMBER(10)
, customer_id NUMBER(10)
, country VARCHAR2(255)
);

CREATE TABLE customers (
  customer_id VARCHAR2(255)
, country VARCHAR2(255)
);

CREATE TABLE products (
  stock_code VARCHAR2(255)
, description VARCHAR2(255)
);

CREATE TABLE purchases (
  purchase_id NUMBER
, stock_code VARCHAR2(255)
, customer_id VARCHAR2(255)
, quantity NUMBER(10)
, unit_price NUMBER(10)
);

CREATE TABLE purchases_distinct (
  purchase_id NUMBER
, stock_code VARCHAR2(255)
, customer_id VARCHAR2(255)
);

EXIT;
</copy>
````
PIC

## **STEP 4** : Load the Retail Dataset

The retail dataset was earlier converted from Excel to the CSV format to make it readable by data load utilities.

In this step you will use Oracle **SQL Loader** to load the CSV file into the **TRANSACTIONS** table of **RETAIL** schema.

1. In the previous SSH connection logged in as the **oracle** user, ensure you are in the right folder.

````
<copy>cd /home/oracle/datasets</copy>
````

2. Create a new file named **sqlldr.ctl** using a command-line text editor of your choice (e.g. **vi** or **vim**).

````
<copy>vi sqlldr.ctl</copy>
````

3. Copy/paste the following content into the editor and save the **sqlldr.ctl** file. This file is the control file used by SQL Loader to make sense of the data file format.

````
<copy>OPTIONS (SKIP=1)
LOAD DATA
CHARACTERSET UTF8
TRUNCATE INTO TABLE transactions
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
(
  invoice_no
, stock_code
, description
, quantity
, invoice_date
, unit_price
, customer_id
, country
)</copy>
````

4. Invoke SQL Loader using the following command line, replacing ```<<Retail_Password>>``` and ```<<Service_Name>>```.

```
sqlldr userid=retail/<<Retail_Password>>@<<Service_Name>> data=OnlineRetail.csv control="sqlldr.ctl" log=sqlldr.log bad=sqlldr.bad direct=true
```

PIC

5. Next step is to pull the transactional data from **TRANSACTIONS** and denormalize it in a relational set comprising of **CUSTOMERS**, **PRODUCTS**, **PURCHASES** and **PURCHASES_DISTINCT**. These tables will be used to build the Property Graph in a later step.

  Start a SQL Plus session as the **RETAIL** user.

```
sqlplus retail/<<Retail_Password>>@<<Service_Name>>
```

6. Run the following SQL script to populate CUSTOMERS, PRODUCTS and PURCHASES.

````
<copy>SET LINESIZE 200
COL stock_code FOR a20
COL customer_id FOR a20
COL country FOR a20
COL description FOR a40

TRUNCATE TABLE customers;
TRUNCATE TABLE products;
TRUNCATE TABLE purchases;
TRUNCATE TABLE purchases_distinct;

INSERT INTO customers (customer_id, country)
SELECT 'cust_'||customer_id, MAX(country)
FROM transactions
WHERE stock_code IS NOT NULL
  AND stock_code < 'A'
  AND customer_id IS NOT NULL
  AND quantity > 0
GROUP BY customer_id
;

INSERT INTO products (stock_code, description)
SELECT 'prod_'||stock_code, MAX(description)
FROM transactions
WHERE stock_code IS NOT NULL
  AND stock_code < 'A'
  AND customer_id IS NOT NULL
  AND quantity > 0
GROUP BY stock_code
;

INSERT INTO purchases (purchase_id, stock_code, customer_id, quantity, unit_price)
SELECT ROWNUM AS purchase_id, 'prod_'||stock_code, 'cust_'||customer_id,
       quantity, unit_price
FROM transactions
WHERE stock_code IS NOT NULL
  AND stock_code < 'A'
  AND customer_id IS NOT NULL
  AND quantity > 0
;

INSERT INTO purchases_distinct (purchase_id, stock_code, customer_id)
SELECT ROWNUM AS purchase_id, stock_code, customer_id
FROM (SELECT DISTINCT 'prod_'||stock_code AS stock_code,
      'cust_'||customer_id AS customer_id
      FROM transactions
     WHERE stock_code IS NOT NULL
       AND stock_code < 'A'
       AND customer_id IS NOT NULL
       AND quantity > 0)
;

SET ECHO ON;

SELECT * FROM customers WHERE ROWNUM <= 5;
SELECT * FROM products WHERE ROWNUM <= 5;
SELECT * FROM purchases WHERE ROWNUM <= 5;
SELECT * FROM purchases_distinct WHERE ROWNUM <= 5;

EXIT;
</copy>
````

7. Verify the above script was successfully run.

[PIC]

## **STEP 5** : Install Oracle Graph

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

4. Open the firewall port 7007 for the Graph Server.

````
<copy>sudo firewall-cmd --permanent --zone=public --add-port=7007/tcp</copy>
````
````
<copy>sudo firewall-cmd --reload</copy>
````

[PIC]

## Summary

You have successfully loaded the Autonomous Database with the dataset and installed Oracle Graph on the compute instance. Please proceed to the next lab using the **Lab Contents** menu on your right.
