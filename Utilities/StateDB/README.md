# Using StateDB Windows with the In-Memory Key–Value Database Redis

## Overview

This example demonstrates how to use StateDB Writer and StateDB Reader Windows in SAS Event Stream Processing.
For more information about installing and using example projects, see [Using the Examples](https://github.com/sassoftware/esp-studio-examples#using-the-examples).

## Use Case

This project demonstrates how to perform insert, lookup, and aggregate operations on an in-memory Redis hash table. This functionality is especially useful when:
- you need to perform fast lookups against large tables with millions of records,
- you want to reduce the startup time of ESP projects by minimizing the container’s memory footprint, and
- you want to build an ESP compute cluster where ESP pods share common data.

## Prerequisites

This example requires [**Redis database**](https://redis.io/) to be installed and available in SAS ESP environment.
Here is an example of setup steps for Redis.

1)	First we will start the test Redis server. You can do that using the following command:
```bash
sudo docker run --name test-redis -p 6379:6379 -d redis
```
2)	Next we will start [RedisInsight](https://redis.io/insight/). This is a Redis client to browse the data in the Redis server and fire CLI commands for the server.
```bash
sudo docker run -v /home/espuser/espstatedb_workshop/redisinsight:/db -p 8002:8001 -d redislabs/redisinsight:latest
```
3)	You can point your browser to http://localhost:8002. This will open the RedisInsight client. Your local Redis instance is already added with the name “localRedis”. If you want to add a different Redis instance, you can do so as well.
4)	You can open the Browser by clicking the “Browser” option on the left side. Since nothing has run yet, this should be blank.
   
> [!IMPORTANT]
> After the Redis setup is finished and running, two variables must be configured in **ESP Studio**
> - `REDIS_HOST`: Set this to Redis database server IP or hostname. Default is 127.0.0.1.
> - `REDIS_PORT`: Set this to Redis database connection port. Default is 6379.

![Properties for Redis connection](img/properties.png "Properties for Redis connection")


## Source Data

For streaming data simulation during testing, we use an InputRate Source window with a Timer Connector that produces one event per second, followed by a Lua window that generates dummy sensor data, which consists of:
  - `sensor_id`: Sensor identifier, can be from 0 to 9
  - `sensor_group`: One of three values ['mezzanine','reception','workshop']
  - `sensor_stmp`: Sensor reading timestamp

## Workflow
Here is a diagram of the project:

![Diagram of the project](img/diagram.png "Diagram of the project")

- **inputRate**: A Source window that ingests dummy events using Timer Connector.
- **sensorsData**: A Lua window that generates dummy sensor events.
- **saveToRedis**: A StateDB Writer window that saves and updates records in Redis hash table.
- **getSavedStamp**: A StateDB Reader window that performs lookup by key in  Redis hash table and selects value.
- **getMaxByGroup**: A StateDB Reader window that performs aggregate function on Redis hash table by selected group value and selects result.

### saveToRedis

This StateDB RWriter window saves (or updates if  exist) key-value record in Redis table using incoming data from sensorsData window.

Explore the settings for the saveToRedis window by doing the following steps:
1. Open the project in SAS Event Stream Processing Studio and select the saveToRedis window.
2. In the right pane, click **Properties** and expand **Database Write Properties**.
   - `Redis prefix`: stream
     
   Here you can set up a unique prefix for the Redis hash table. For simplicity, you can think of it as the table name.
   - `Time to live`: 60
   
   Here you can set the retention period (in seconds) for each Redis key-value record. This is useful for retention-based aggregations. For example, calculating the maximum value per group over the last 1 hour (you need to set `Time to live` equel 3600).
   - `Time field`: (use system clock)
   
   This defines the time reference used for retention counting. By default, the system clock is used, but you can also specify a timestamp field from the input event.
   - `Secondary Indexes`: Index SENSOR_GROUP - Fields sensor_group 

   Here we assign an additional index on SENSOR_GROUP, since another window will perform aggregation based on this field.
### getSavedStamp

This StateDB Reader window performs lookup of a record in Redis table using keys from incoming data from sensorsData window.
Explore the settings for the getSavedStamp window by doing the following steps:
1. Open the project in SAS Event Stream Processing Studio and select the getSavedStamp window.
2. In the right pane, click **Properties** and expand **Database Query**.
   - `Redis prefix`: stream
     
   Here you can set up a unique prefix for the Redis hash table. For simplicity, you can think of it as the table name. When we write above we used `stream` prefix so you need to point the same one  here to be able to perform lookup.
   - `Query`: sensor_id == SENSOR_ID
        
   Here you can set up query condition. In the example for input event we seach record in Redis Hash table where input field `sensor_id` equel to Data Store Field `SENSOR_ID`.
    - `Time field`: (use system clock)
   
   This defines the time reference used for retention counting. By default, the system clock is used, but you can also specify a timestamp field from the input event.
3. In the right pane, click **Output schema** and click **Edit fields** pencil.
    - `sensor_id` is mapped to Source=Input,  this  field  will be recieved from the event
    - `sensor_group`: is mapped to Source=Input,  this  field  will be recieved from the event
    - `sensor_stmp`: is mapped to Source=Input,  this  field  will be recieved from the event
    - `sensor_saved_stmp`: new field which is mapped to Source=Query,  this  field  will be recieved from the Redis `SENSOR_STMP` value

### getMaxByGroup

This StateDB Reader window aggregates Redis table data using keys from incoming data from sensorsData window.
Explore the settings for the getMaxByGroup window by doing the following steps:
1. Open the project in SAS Event Stream Processing Studio and select the getMaxByGroup window.
2. In the right pane, click **Properties** and expand **Database Query**.
   - `Redis prefix`: stream
     
   Here you can set up a unique prefix for the Redis hash table. For simplicity, you can think of it as the table name. When we write above we used `stream` prefix so you need to point the same one  here to be able to perform lookup.
   - `Query`: sensor_group == SENSOR_GROUP
        
   Here you can set up query condition. In this case for aggregation we need to set group key. In the example for input event we want to perform aggregation by `sensor_group` value as a key. 
   - `Time field`: (use system clock)
   
   This defines the time reference used for retention counting. By default, the system clock is used, but you can also specify a timestamp field from the input event.
3. In the right pane, click **Output schema** and click **Edit fields** pencil.
    - `sensor_id` is mapped to Source=Input,  this  field  will be recieved from the event
    - `sensor_group`: is mapped to Source=Input,  this  field  will be recieved from the event
    - `sensor_stmp`: is mapped to Source=Input,  this  field  will be recieved from the event
    - `sensor_group_max_id`: new field which is mapped to Source=Query,  this  field  will be calculated on data from the Redis. In the example we selected Aggregate function ESP_aMax and  source field `SENSOR_ID`. This means that for each input event for the specific `sensor_group` ('mezzanine','reception' or 'workshop') we will get the maximum `SENSOR_ID` which stored at the moment in Redis, according to 60 seconds retention policy.

## Test the Project and View the Results

When you test the project in SAS Event Stream Processing Studio, the results for each window appear in separate tabs:

- **saveToRedis**: Displays incoming data. The `sensor_id`, `sensor_group`, and `sensor_stmp` fields will be stored in a Redis hash. Redis will create indexes on `sensor_id` and `sensor_group` to support future aggregations.
  
  ![saveToRedis window output](img/saveToRedis.png "saveToRedis window output")
- **getSavedStamp**: Displays incoming data and data from Redis. In the `sensors_saved_stmp` column, you can see fetched data for a specific key (`sensor_id`) which currently stored in the Redis hash. At the beginning of the test, no data is cached, but as the saveToRedis window runs, the data starts appearing.
  
  ![getSavedStamp window output](img/getSavedStamp.png "getSavedStamp window output")
- **getMaxByGroup**:  Displays incoming data and data from Redis. In the `sensor_group_max_id` column, you can see aggregated data from Redis - maximum saved `SENSOR_ID` for the `sensor_group` value in the incoming event, where all records limited by 60 seconds TTL.
  
  ![getMaxByGroup window output](img/getMaxByGroup.png "getMaxByGroup window output")

## Next Steps

You can enhance this project by doing any of the following:
- Replacing the dummy source with a live sensor feed
- Experimenting with different aggregations in the StateDB Reader
- Adjusting the TTL (time-to-live event retention) in the StateDB Writer

## Additional Resources

- [SAS Help Center: Using StateDB Windows](https://go.documentation.sas.com/doc/en/espcdc/default/espcreatewindows/n01c9h6p6pmlcmn11w46am1xgnum.htm)
