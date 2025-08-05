# Introduction to Copy Window and types of Retention

## Overview
This SAS Event Stream Processing (ESP) project demonstrates the use of the Copy window and the different types of retentions it supports.  Copy window as the name suggests, copies the schema and the output of a connected parent window as it is. By using this window, you can add retention of events to the data flow which is very important to introduce state to the project as well as to manage the size of the state.

For more information about how to install and use example projects, see [Using the Examples](https://github.com/sassoftware/esp-studio-examples#using-the-examples).

## Use Case 
* Copy window generally goes with Aggregate window. It is required if the model till the Aggregate window has been stateless. Since Aggregate window is always stateful, Copy window helps to manage the state of the Aggregate window and the rest of the model.
* Multiple parallel copy windows are used to add multiple retentions, mostly time based, to achieve different retention based use cases. Think of the scenario where you want to monitor averages over 5 second as well as 10 second periods.

## Source Data
The file, input.csv, contains a dummy motor vibration data where the fields motor and time in microseconds together identify a unique event. The value here is dummy vibration data of motors which are emitting the events at a rate of 1 event per second.

## Workflow
The example shows what are the different retention options that you can use with Copy window. The following figure shows the diagram of the project:

![Diagram of the project](img/model.png)	

Here each copy window shows a different retention format.
- The Source window, as the name suggests is the entry point of the data. It reads the events at the rate of 1 event per second which is driven through the settings of the publish connector.
- The Copy_Count_Jumping window demonstrates the Jumping retention based on the number of events stored in memory.
- The Copy_Count_Sliding window shows the Sliding retention based on the number of events stored in memory.
- The Copy_SystemTime_Jumping window shows Jumping retention based on the system time.
- The Copy_SystemTime_Sliding window shows the Sliding retention based on the system time
- The Copy_EventTime_Sliding window shoes the Sliding retention based on the timestamp field in the event.

### Copy_Count_Jumping
You can select the Copy_Count_Jumping window to see the properties of the window in the properties pane on the right side.

![Retention settings for Copy_Count_Jumping window](img/copy_count_jumping_properties.png)

The window is stateful. This can be noticed from the index which is set as pi_HASH.

The retention is set as “By row count, jumping” with a Row limit of 10.

This means that the window will keep a track of at max 10 events and as soon as the 11th event comes, for all 10 events before it, a corresponding delete event will be generated.
You can confirm this by looking at the output of this window. It will have 21 events where 11 are input events and 10 are deletes from the first 10 input events.

NOTE: This type of behaviour is useful when you want to reset the whole state of the model based on the number of events being published.

### Copy_Count_Sliding
You can select the Copy_Count_Sliding window to see the properties of the window in the properties pane on the right side.

![Retention settings for Copy_Count_Sliding window](img/copy_count_sliding_properties.png)

The window is stateful. This can be noticed from the index which is set as pi_HASH.

The retention is set as “By row count, sliding” with a Row limit of 10.

This means that the window will keep a track of at max 10 events and as soon as the 11th event comes, a delete event will be generated for the 1st event. Similarly, when the 12th event comes, a delete event will be generated for the 2nd event and so on.
You can confirm this by looking at the output of this window. It will have 12 events where 11 are input events and 1 delete event for the corresponding 1st input event. Also notice the “Currently retained events” which remains at 10.

NOTE: This type of behaviour is useful when you want to maintain a fixed state of the model based on the number of events being published.

### Copy_SystemTime_Jumping
You can select the Copy_SystemTime_Jumping window to see the properties of the window in the properties pane on the right side.

![Retention settings for Copy_SystemTime_Jumping window](img/copy_systemtime_jumping_properties.png)

The window is stateful. This can be noticed from the index which is set as pi_HASH.

The retention is set as “By time, jumping” with a Time limit of 5 seconds and Time field, which tells ESP the clock to follow to determine time being depended on system clock.

In this case, there is an internal timer started with the project. Once the timer completes the specified time limit, in this case 5 seconds, the Copy window generates delete events for all the events in it’s state. Keep in mind that the timer is internal to ESP and is independent of the events being pushed. 

NOTE: This kind of retention is useful when you want to clear the whole state at fixed interval of times irrespective of the events that came in.

### Copy_SystemTime_Sliding
You can select the Copy_SystemTime_Sliding window to see the properties of the window in the properties pane on the right side.

![Retention settings for Copy_SystemTime_Sliding window](img/copy_systemtime_sliding_properties.png)

The window is stateful. This can be noticed from the index which is set as pi_HASH.

The retention is set as “By time, sliding” with a Time limit of 5 seconds and Time field, which tells ESP the clock to follow to determine time being depended on system clock.

In this case, there is an internal timer started with the project. At every heartbeat (which is by default 1 second), the Copy window generates delete events for all the events, in it’s state, which are older than the specified time which in this case is 5 seconds. So for an event that came before 5 seconds a delete event is generated. In the next second, the window again checks for events older than 5 seconds and generate a delete event for them and so on. In this case the timer is internal to ESP and is independent of the events being pushed.

NOTE: This type of retention is very useful where you want to consistently manage the state of the window through time as the internal timer keeps on running even when there are no events or more than average events.

### Copy_EventTime_Sliding
You can select the Copy_SystemTime_Sliding window to see the properties of the window in the properties pane on the right side.

![Retention settings for Copy_EventTime_Sliding window](img/copy_eventtime_sliding_properties.png)

The window is stateful. This can be noticed from the index which is set as pi_HASH.

The retention is set as “By time, sliding” with a Time limit of 5 seconds and Time field, which tells ESP the clock to follow to determine time being depended on field in the schema named “time”. The UI only shows the fields which are stamp type or date type.

In this case, the Copy window monitors the time based on the field specified. For the window, a time limit is reached when it encounters an event where the difference between the value of the time field of the latest event and the first event in its state is more than or equal to the specified time limit, in this case 5 seconds. When such an event comes, the window generates a delete event for all events which matches the criteria for the time.

NOTE: This kind of retention is very useful where events may not follow exact system time either because you are developing the model and passing the events very fast or in production there can be inconsistent delays in arrival of the events because of network issues. 









