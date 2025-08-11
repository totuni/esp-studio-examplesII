# Introduction to Copy Window and Types of Retention

## Overview
This SAS Event Stream Processing (ESP) project shows how to use the Copy window and the different types of retention that it supports. The Copy window replicates the schema and output of its connected parent window without modifying the data. Its primary purpose is to introduce retention policies that determine how long events are kept in memory. This enables you to add state to an otherwise stateless portion of the model, enabling features like aggregation over time. Retention settings help control the size of the maintained state, which is crucial for managing performance and resource usage in streaming applications.

**What is State in ESP?** By default, streaming data is ephemeral: each event passes through the system and is immediately discarded unless something explicitly retains it. The Copy window changes that by introducing state—a memory of past events—into the data flow. This is essential for operations that rely on historical context, like time-based aggregations. Retention settings also help manage how much of the historical data is kept, balancing memory and performance.

For more information about how to install and use example projects, see [Using the Examples](https://github.com/sassoftware/esp-studio-examples#using-the-examples).

## Use Case 
* Copy window generally goes with Aggregate window. It is a model requirement until the Aggregate window that is used first is stateless. Since Aggregate window is always stateful, Copy window helps manage the state of the Aggregate window and the rest of the model.
* Multiple parallel Copy windows are used to add time-based retentions to achieve different retention-based use cases. Think of a scenario where you want to monitor averages over 5 second and 10-second periods.

## Source Data
<!-- is "a dummy motor vibration data" supposed to be singular? this makes it sound like it's one piece of data -->
Input.csv contains a dummy motor vibration data where the fields motor and time in microseconds together identify a unique event. The value here is dummy vibration data of motors that are emitting the events at a rate of 1 event per second. 

## Workflow
The example shows the different retention options that you can use with Copy window. The following figure shows the diagram of the project:

![Diagram of the project](img/model.png)	

Here each copy window shows a different retention format.
- The Source window is the entry point of the data. It reads the events at a rate of 1 event per second, which is driven through the settings of the publish connector.
- The Copy_Count_Jumping window demonstrates the Jumping retention based on the number of events stored in memory.
- The Copy_Count_Sliding window shows the Sliding retention based on the number of events stored in memory.
- The Copy_SystemTime_Jumping window shows Jumping retention based on the system time.
- The Copy_SystemTime_Sliding window shows the Sliding retention based on the system time.
- The Copy_EventTime_Sliding window shoes the Sliding retention based on the timestamp field in the event.

### Copy_Count_Jumping
You can select the Copy_Count_Jumping window to see the properties of the window in the right pane.

![Retention settings for Copy_Count_Jumping window](img/copy_count_jumping_properties.png)

The window is stateful. You see this from the index, which is set to pi_HASH.

The retention is set as “By row count, jumping” with a **Row limit** of 10.

This means that the window keeps track of 10 events at maximum, and as soon as the 11th event comes, a delete event is generated for the 10 events before it.
You can confirm this by viewing the output. It has 21 events. There are 11 input events and 10 deletes from the first 10 input events.

NOTE: This type of behavior is useful when you want to reset the whole state of the model based on the number of events being published.

### Copy_Count_Sliding
You can select the Copy_Count_Sliding window to see the properties of the window in the right pane.

![Retention settings for Copy_Count_Sliding window](img/copy_count_sliding_properties.png)

The window is stateful. You see this from the index, which is set to pi_HASH.

The retention is set as “By row count, sliding” with a **Row limit** of 10.

This means that the window keeps track of 10 events at maximum, and as soon as the 11th event comes, a delete event is generated for the 1st event. When the 12th event comes, a delete event is generated for the 2nd event and so on.
You can confirm this by viewing the output. It has 12 events. There are 11 input events and 1 delete event for the 1st input event. Also notice the **Currently retained events** value, which remains at 10.

NOTE: This type of behavior is useful when you want to maintain a fixed state of the model based on the number of events being published.

### Copy_SystemTime_Jumping
You can select the Copy_SystemTime_Jumping window to see the properties of the window in the right pane.

![Retention settings for Copy_SystemTime_Jumping window](img/copy_systemtime_jumping_properties.png)

The window is stateful. You see this from the index, which is set to pi_HASH.

The retention is set as “By time, jumping” with a **Time limit** of 5 seconds and a **Time field** of (use system clock), which tells ESP which clock to follow.

In this case, there is an internal timer started with the project. Once the timer completes the specified time limit, in this case 5 seconds, the Copy window generates delete events for all the events in its state. Keep in mind that the timer is internal to ESP and is independent of the events being pushed. 

NOTE: This type of retention is useful when you want to clear the whole state at fixed intervals regardless of the events that came in.

### Copy_SystemTime_Sliding
You can select the Copy_SystemTime_Sliding window to see the properties of the window in the right pane.

![Retention settings for Copy_SystemTime_Sliding window](img/copy_systemtime_sliding_properties.png)

The window is stateful. You see this from the index, which is set to pi_HASH.

The retention is set as “By time, sliding” with a **Time limit** of 5 seconds and a **Time field** of (use system clock), which tells ESP which clock to follow.
<!-- is heartbeat a specific unit of measurement associated with this model? If not, let's just say "Every second" -->
In this case, there is an internal timer that starts with the project. At every heartbeat (which is by default 1 second), the Copy window generates delete events for all the events in its state that are older than the specified time, which in this case is 5 seconds. For an event that comes before 5 seconds a delete event is generated. In the next second, the window checks again for events older than 5 seconds and generates a delete event for them. In this case the timer is internal to ESP and is independent of the events being pushed.

NOTE: This type of retention is useful when you want to consistently manage the state of the window through time. The internal timer keeps running even when there are no events or more than average events.
<!-- what does "more than average events" mean? the wording sounds odd here -->
### Copy_EventTime_Sliding
You can select the Copy_SystemTime_Sliding window to see the properties of the window in the right pane.

![Retention settings for Copy_EventTime_Sliding window](img/copy_eventtime_sliding_properties.png)

The window is stateful. You see this from the index, which is set to pi_HASH.
<!-- again with the "T" in Time -->
The retention is set as “By time, sliding” with a **Time limit** of 5 seconds and **Time field** of time, which tells ESP which clock to follow. The **Time field** drop-down list shows only fields whose data type is stamp or date.
<!-- I don't understand this: "determine time being depended on field in the schema named “time” -->

In this case, the Copy window monitors the time based on the specified field. The Copy window reaches a time limit when it encounters an event where the difference between the value of the time field of the latest event and the first event in its state, is more than or equal to the specified time limit. In this case the difference is 5 seconds. When an event like this comes, the window generates a delete event for all events that match the criteria for the time.

NOTE: This type of retention is useful when events might not follow the exact system time. This might be because you are developing the model and passing the events rapidly, or there are inconsistent delays in production, which affect the arrival of the events due to network issues. 

## Test the Project and View the Results
When you test the project, the results for each window appear on separate tabs.

- Source tab lists raw events published into the project.
- Copy_SystemTime_Sliding tab lists insert and delete events generated because of system time based sliding retention.
- Copy_EventTime_Sliding tab lists insert and delete events generated because of event time based sliding retention.
- Copy_SystemTime_Jumping tab lists insert and delete events generated because of system time based jumping retention.
- Copy_Count_Sliding tab lists insert and delete events generated because of count based sliding retention.
- Copy_Count_Jumping tab lists insert and delete events generated because of count based jumping retention.

The following figure shows the results of the Source Window tab:

![Results for Source window](img/source_window_result.png)

Keep a note of the number of events published into the project.

The following figure shows the results of the Copy_SystemTime_Sliding window:

![Results for Copy_SystemTime_Sliding window](img/copy_systemtime_sliding_result.png)

Notice that for all events, deletes are generated. You can also confirm this with the **Current retained events** value, which is 0.

The following figure shows the results of the Copy_EventTime_Sliding window:

![Results for Copy_EventTime_Sliding window](img/copy_eventtime_sliding_result.png)

Notice that there are 2 deletes generated whose time field values are exactly 5 seconds less than the last event published. Because other events have time values less than 5 seconds from the last event, their deletes are not generated.

The following figure shows the results of the Copy_SystemTime_Jumping window:

![Results for Copy_SystemTime_Jumping window](img/copy_systemtime_jumping_result.png)

Notice in this case, all deletes are generated. However, the difference from the Copy_SystemTime_Sliding is that all deletes are generated at the same time.

The following figure shows the results of the Copy_Count_Sliding tab:

![Results for Copy_Count_Sliding window](img/copy_count_sliding_result.png)

Notice that there is a single delete event generated. The 11th insert event crosses the retention limit of 10 events. This leads to the deletion of the first event in the state. The number of retained events remains at 10.

The following figure shows the results of the Copy_Count_Jumping tab:

![Results for Copy_Count_Jumping window](img/copy_count_jumping_result.png)

Notice that only 1 event is shown as **Currently etained events**. This happened because as soon as the 11th event arrived, the retention of 10 events was crossed, which leads to the deletion of all previous events.

## Additional Resources
For more information, see [SAS Help Center: Using Copy Windows](https://documentation.sas.com/?cdcId=espcdc&cdcVersion=default&docsetId=espcreatewindows&docsetTarget=n03rea4fhvohcgn15o9970mq9jea).











