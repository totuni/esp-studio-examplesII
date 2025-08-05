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
The example shows what are the different retention options that you can use with Copy Window. The following figure shows the diagram of the project:

![model](img/model.png)	

