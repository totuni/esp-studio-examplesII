# SAS ESP Join Window 

## Overview
This example demonstrates all the various ways to use SAS ESP join windows with two sample data sources. Join windows in SAS ESP allow you to combine streaming data from multiple sources based on specified join conditions.

### Understanding Fact and Dimension Inputs

In SAS ESP streaming analytics, understanding the distinction between facts and dimensions is crucial for designing effective join operations.

**Dimensions** = Small to large, reference/lookup tables. When tables are large, indexing may provide improved performance.

**Facts** = Large, transactional streams.

In a **join window** in SAS ESP:

- The join window has a **left input** (often called the *fact* table) and a **right input** (often called the *dimension* table).
- The **left (fact) input** is usually the stream of events you want to enrich — e.g., sensor readings, transactions, clickstream.
- The **right (dimension) input** is typically a lookup or reference stream with the fields you’ll use to join on — e.g., customer master data, machine configuration.

This fundamental difference in data characteristics drives how SAS ESP optimizes join operations and determines which stream should be assigned to which input for best performance.

## Sample Data Sources

In this example, we’re working with two data sources: **Orders** and **Customers**. The **CustomerID** column connects the two, serving as the common key. Here, the **Customers** table provides additional information to enrich the data, while the **Orders** table represents the continuous stream of events that we want to enhance with that information.

## Join Window Types and Configurations

In this section, we’ll explore four common types of joins: **Inner Join, Left Outer Join, Right Outer Join, and Full Outer Join**. Joins are used to combine data from multiple sources based on a shared key, allowing us to bring together related information in meaningful ways. Each join type determines which records from the input tables are included in the result, and understanding the differences is key to choosing the right approach for your data.

### Inner Join
**Purpose**: Shows only records where CustomerID exists in both tables. Records are matched based on the CustomerID key (shown in colors).

![image-20250820132830786](img/image-20250820132830786.png)	

![image-20250820132903526](img/image-20250820132903526.png)	



### Left Outer Join
**Purpose**: Returns all records from the left stream, with matching records from the right stream where available.

![image-20250820133419283](img/image-20250820133419283.png)	

![image-20250820133450163](img/image-20250820133450163.png)	

**Result**:



### Right Outer Join
**Purpose**: Returns all records from the right stream, with matching records from the left stream where available.

![image-20250820133715349](img/image-20250820133715349.png)	



**Result**:

![image-20250820133746229](img/image-20250820133746229.png)	


### Full Outer Join
**Purpose**: Shows ALL customers and ALL orders. Missing data is filled with NULL values.

**ESP Configuration**:

![image-20250820133851284](img/image-20250820133851284.png)	

**result:**

![image-20250820133928451](img/image-20250820133928451.png)	







This demo provides a comprehensive overview of all SAS ESP join window capabilities using realistic sample data that clearly demonstrates the differences between each join type.