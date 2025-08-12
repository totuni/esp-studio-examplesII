# Introduction to Aggregate Window and Aggregate Functions
## Overview

This SAS Event Stream Processing (ESP) project demonstrates how to use the Aggregate window, and the different types of aggregation functions it supports. Aggregate window allows you to add simple statistics like sum, mean, std deviation and more to the streaming data. Since these calculations are provided out of the box and implemented on the streaming data, these are very fast, matching the rate of the incoming data. 

Aggregate windows are always Stateful which means they need to retain the events. Therefore, Aggregate windows, almost always, go with Copy window preceding them with retention to manage the size of the state.

For more information about how to install and use example projects, see [Using the Examples](https://github.com/sassoftware/esp-studio-examples#using-the-examples).

## Use Case
- You need to add basic statistics to the Streaming data. These statistics are used in the ESP model as filtering criteria or for further calculations.
- The Aggregate window can also be used to capture the first occurrence in a group which will be an Insert event.

## Source Data
The file, input.csv, contains a dummy motor vibration data where the fields motor and time in microseconds together identify a unique event. The value here is dummy vibration data of motors which are emitting the events at a rate of 1 event per second.


