# SAS ESP Join Window 

## Overview
This example demonstrates all the various ways to use SAS ESP join windows with three sample data sources. Join windows in SAS ESP allow you to combine streaming data from multiple sources based on specified join conditions.

### Understanding Fact and Dimension Inputs

In SAS ESP streaming analytics, understanding the distinction between facts and dimensions is crucial for designing effective join operations.

**Dimensions** = Small to large, reference/lookup tables. When tables are large, indexing may provide improved performance.

**Facts** = Large, transactional streams.

In a **join window** in SAS ESP:

- The join window has a **left input** (often called the *fact* table) and a **right input** (often called the *dimension* table).
- The **left (fact) input** is usually the stream of events you want to enrich — e.g., sensor readings, transactions, clickstream.
- The **right (dimension) input** is typically a lookup or reference stream with the fields you’ll use to join on — e.g., customer master data, machine configuration.

This fundamental difference in data characteristics drives how SAS ESP optimizes join operations and determines which stream should be assigned to which input for best performance.

## Workflow

![image-20250903131648794](img/image-20250903131648794.png)	

The project shown above will be used to show how different join types behave when combining a streaming fact table with dimension/lookup tables.

------

### **Top part: Dimension (lookup) tables**

- **Customers** → static (lookup) info about customers (e.g., ID, name, city).
- **Loyalty** → static (lookup) info about loyalty status (e.g., Silver, Gold, Bronze).

These are *dimension windows*, meaning they provide reference data to enrich the fact stream.

------

### **Middle part: Fact (streaming) table**

- **Orders** → the *fact window*, which receives a continuous stream of events (e.g., new customer orders).

This is the “driver” stream that you want to enrich with data from Customers and Loyalty.

------

### **Join windows**

#### **Why FullOuterJoin is used in the middle**

**FullOuterJoin**: combines **Customers** + **Loyalty**, so that downstream you have a unified view of customer + loyalty info. This is often used as a *pre-join step* to simplify your pipeline.   That way, instead of joining Orders against two tables separately, you have a single unified dimension table containing all the info. Then you can test different join strategies between streaming orders and this dimension.   It should also be noted that the Full outer joins only can only combine two dimension tables.  Therefore, it is not possible to show the full outer join using the Orders fact table.  

#### **Bottom part: Different join types**

From there, the **Orders** fact stream is joined with the customer+loyalty data using all three join types, so you can compare behavior:

1. **InnerJoin**
2. **LeftOuterJoin**
3. **RightOuterJoin**



## Sample Data Sources

![image-20250902151250980](img/image-20250902151250980.png)

In this example, we’re working with the following data sources: **Orders**, **Loyalty** and **Customers**. The **CustomerID** column serves as the common key. Here, the **Customers** and **Loyalty** tables provide additional information to enrich the data, while the **Orders** table represents the continuous stream of events that we want to enhance with that information.

## Join Window Types and Configurations

In this section, we’ll explore four common types of joins: **Inner Join, Left Outer Join, Right Outer Join, and Full Outer Join**. Joins are used to combine data from multiple sources based on a shared key, allowing us to bring together related information in meaningful ways. Each join type determines which records from the input tables are included in the result, and understanding the differences is key to choosing the right approach for your data.  

### Full Outer Join

**Purpose**:  A **full outer join** brings together **all rows from both tables**, matching them where keys line up, and filling in `NULL` (or blanks) where they don’t.  In our customer example, we want to include all customers whether they are part of the loyalty program or not.  

**ESP Configuration**:

![image-20250826163711614](img/image-20250826163711614.png)	

Note that the schema contains elements from the left and right side of the join and are denoted by a "l" or "r" in the Expression column of the schema. 

![image-20250902171628292](img/image-20250902171628292.png)	

Here you can see that Loyalty is coming from the right side of the join while CustomerName and City come from the left. 

![image-20250902151811114](img/image-20250902151811114.png)		

- **Matched records (111, 222, 333):** Both tables had these CustomerIDs, so all columns are populated
- **Customer 555 (Diana):** Exists only in Customers table, so Loyalty is NULL
- **Customer 666:** Exists only in Loyalty table, so CustomerName and Amount are NULL
- This new combined table will be used for the remaining join examples.

**Test the Project and View the Results**:

When running this project in test mode the FullOuterJoin tab shows the following: 

![image-20250902160511702](img/image-20250902160511702.png)	

Note the Customer table records are inserted first and later updated when the Loyalty data is received.  The deletes of the records before the update can be ignored.  	

### Inner Join

**Purpose**: Shows only records where CustomerID exists in both tables. Records are matched based on the CustomerID key. 

**ESP Configuration**:

![image-20250902171209414](img/image-20250902171209414.png)	

Note that the schema contains elements from the left and right side of the join and are denoted by a "l" or "r" in the Expression column of the schema.

![image-20250902171802303](img/image-20250902171802303.png)	



![image-20250902162749219](img/image-20250902162749219.png)	

![image-20250902155417514](img/image-20250902155417514.png)	

With Orders acting as the **fact table** and Customers/Loyalty as the combined **dimension table**, a record is generated for each order received where the CustomerID matches. 

- **Matched records (111, 222, 555):** Both tables had these CustomerIDs, so all columns are populated
- **Customer 555 (Diana):** Loyalty is set to NULL
- **Order 999 ** Does not exist in the dimension table and is therefore dropped

**Test the Project and View the Results**:

![image-20250902163002309](img/image-20250902163002309.png)

### Left Outer Join
**Purpose**: Returns all records from the fact stream, with matching records from the dimension stream where available.  

![image-20250902163649185](img/image-20250902163649185.png)		

![image-20250902163855496](img/image-20250902163855496.png)		

Here we create a record for every event which enters the steam on the fact side, which in this case is the left side.  Therefore, ORD005 creates a joined event even when all the dimension fields are NULL.

**Test the Project and View the Results**:

![image-20250902164233160](img/image-20250902164233160.png)

### Right Outer Join
**Purpose**: Returns all records from the right stream, with matching records from the left stream where available.

This join type is not compatible with our project because ESP joins Fact tables (streaming) with Dimension tables (lookup).  Since our Orders table is on the left side, when you try to pick the join type of Right Outer join ESP studio will give you the following error. 

![image-20250902165914825](img/image-20250902165914825.png)	

The only way to fix this issue is to re-arrange the project so that the Fact tables are streaming in on the right side.  In order to fix this we need to rebuild the schema entries so that the fact tables are coming from the right side.  Under settings for the RightOuterJoin you will see the following: 

![image-20250903113545299](img/image-20250903113545299.png)	

This clearly states that the left window is Orders while the right window is the FullOuterJoin.   We need to flip this around by clicking those double arrows between the two.  This will generate the following message: 

![image-20250903113749133](img/image-20250903113749133.png)	

After clicking yes you can now see that the Left and Right designations are reversed. In the version of the project you’re working with, this fix has already been implemented — the fact stream is on the right side and the schema rebuilt — so you can run the Right Outer Join without re-arranging anything yourself.

![image-20250903113850309](img/image-20250903113850309.png)	

This action also deletes the old schema so it will need to be rebuilt.  Once rebuilt the orders fact table are now prefixed with "r_" .

![image-20250903114822704](img/image-20250903114822704.png)	

This means the streaming or fact side is now on the right instead of the left.  Logically this is the same as doing a left outer join as show in the the results below: 

![image-20250903130541976](img/image-20250903130541976.png)	

## Join Window and State Management 

In SAS Event Stream Processing (ESP), the concept of state typically refers to the stored event data that a window maintains in memory over time or by count.  When you see a lightning bolt in the lower right of the ESP project window it indicates that the window is storing events in memory or has become stateful. ![image-20250903133635158](img/image-20250903133635158.png)	In a production environment the fact side of the project is infinite.   No program can store an infinite amount of data therefore the size of the records maintained in memory needs to be managed.  The Join window in ESP allows you to set the state for the left and right inputs separately.  Under join criteria you will see the following: 

![image-20250903135717569](img/image-20250903135717569.png)	

The fact side of the window is set to stateless, meaning no events are stored except for the current record.   The only memory which will be used to store event data will be on the dimension  or right side in this case.  This effectively removes any unbounded memory growth issues from the project.  

### Large lookup tables

When a record is received in the orders window it will need to be matched to a corresponding record in the dimension data.  A lookup is issued for the CustomerID to see if there is a match in the dimension table.   If  the dimension table  is large this lookup can be sped up by creating a secondary index.  Therefore, when lookup tables are large check the Use a secondary index check box in the Join Criteria section.  



## Summary

This demo shows how SAS Event Stream Processing (ESP) join windows let you **enrich live data streams with contextual information** from reference tables in real time. By combining fast-moving “fact” data (like orders, sensor readings, or transactions) with “dimension” data (like customer records or machine configurations), you can make decisions and detect patterns as events happen instead of after the fact. The walkthrough explains why to pre-join dimension tables, how to choose the right join type for your outcome, and how to configure state so only necessary records are held in memory. Following these practices yields lower latency, avoids unbounded memory growth, and makes streaming pipelines easier to maintain and scale.

