# Exploring ESP Source Connectors
## Overview

In SAS Event Stream Processing (ESP), *connectors* (also known as *adapters*) serve as the vital interface between external data systems and your streaming analytics engine. They enable real-time ingestion of data from a wide variety of sources — whether that's sensors via MQTT, logs via Kafka, files on disk, or live video streams over RTSP.

Understanding connectors is fundamental for anyone getting started with ESP, as nearly every real-world application begins with getting data into the system. This example project is designed to help new users visualize and explore how ESP handles this through a set of simple, illustrative connector configurations. You can load this project into ESP Studio or deploy it via the XML directly to see how data flows into a source window from different origins.

For more information about how to install and use example projects, see [Using the Examples](https://github.com/sassoftware/esp-studio-examples#using-the-examples).

### 1. **CSV File Input (File/Socket Connector) — `Source_CSV`**

Uses the `fs` connector to simulate data streaming from static CSV files.

- Multiple versions are shown:
  - Basic file read
  - Repeating input
  - Rate-controlled input
- Also includes a *sub* MQTT connector attached to this window (used to simulate updates from the same source schema).

🔹 **Use for:** Testing, log replay, geospatial tracking, and historical simulations.
 🔹 **Note:** Data is read from a local path and can be repeated or throttled.

------

### 2. **Timed Input — `Source_timed`**

The `timer` connector generates events at a fixed interval (e.g., every 60 seconds).

- Includes timestamped fields
- Can be used to simulate heartbeats, time triggers, or scheduled events

🔹 **Use for:** Time-based testing, sampling simulations, or keeping other windows "alive."

------

### 3. **Kafka Input — `Source_Kafka`**

Demonstrates two Kafka connectors for consuming messages:

- One starts from the latest offset
- One reads all messages from the beginning of the topic
- Uses opaque strings to simulate unstructured message ingestion

🔹 **Use for:** Real-time event data, cloud-scale streaming, telemetry pipelines.
 🔹 **Note:** Set as inactive by default; requires a valid Kafka configuration (currently pointing to Azure Event Hubs).

------

### 4. **MQTT Input — `Source_MQTT`**

Configured to read messages from a public MQTT broker (`test.mosquitto.org`).

- Subscribes to a topic named `ExampleforESP`
- Reads opaque strings
- Matches lightweight IoT scenarios

🔹 **Use for:** Edge device data, smart home sensors, gateway telemetry.
 🔹 **Note:** Set as inactive; you can activate and publish messages to this topic to test.

------

### 5. **Azure Event Hubs — `Source_Eventhub`**

This is an example placeholder for reading from Azure Event Hubs.

- Disabled by default
- Requires user setup with an Azure namespace, access keys, and event path

🔹 **Use for:** Cloud event ingestion, scalable IoT data pipelines.

------

### 6. **Video Stream Input — `Source_video`**

Uses the `videocap` connector to read frames from a video file.

- Designed to mimic live camera input
- Publishes image blobs for computer vision pipelines
- Optional resizing and frame rate settings included

🔹 **Use for:** Surveillance analysis, object detection, video streaming analytics.
 🔹 **Note:** Disabled by default; file path points to a demo MP4 video.

## Source Data

The [InputRemove.csv](InputRemove.csv) file contains a list of stock market trades. 

## Workflow
The following figure shows the diagram of the project:

![image-20250708160552348](img/image-20250708160552348.png)	

### sourceWindow

Explore the settings for the sourceWindow window:
1. Open the project in SAS Event Stream Processing Studio and select the sourceWindow window. 
2. In the right pane, expand **State and Event Type**. Observe that the index type is PI_HASH. That is, the window is stateful. The window retains all incoming events.
3. To examine the window's output schema, on the right toolbar, click ![Output Schema](img/output-schema-icon.png "Output Schema"). Observe the following fields: 
   - `ID`: This field is the stock trade's ID, which is also selected as the key field.
   - `symbol`: This field is the stock symbol. A stock symbol is a series of letters that are assigned to a security for trading purposes.
   - `true_test`: This field is the "true_test" value for each trade.
   - `price`: This field is the stock price.
4. Click ![Properties](img/show-properties-icon.png "Properties"). 

### removestateWindow

Explore the settings for the removestateWindow window:
1. Select the removestateWindow window.
2. In the right pane, expand **Settings**. Observe that Updates, Deletes, Retention Updates, and Retention Deletes are the opcodes that are to be removed by this window.
3. Observe that the **Include opcode and flag fields** check box is selected. This causes two fields to be added to the output schema.
4. To examine the window's output schema, on the right toolbar, click ![Output Schema](img/output-schema-icon.png "Output Schema"). Observe the following details:
   - `eventNumber`: This field is an event number that is assigned to each event. The event number is a monotone-increasing sequential integer. The output schema for a Remove State window always includes this field. This field is the only key of a Remove State window.
   - `originalOC` and `originalFL`: These fields are present in the output schema because the **Include opcode and flag fields** check box is selected for this window. For each event, these fields contain the event's original opcode and the event's original flag.
   - `ID`, `symbol`, `true_test`, and `price`: These fields are present in the sourceWindow window and are passed through, but the `ID` field is no longer a key field.
5. Click ![Properties](img/show-properties-icon.png "Properties"). 

### copyWindow

Explore the settings for the copyWindow window:
1. Select the copyWindow window.
2. In the right pane, expand **State**. Observe that the window is stateful. The window retains all incoming events, except when a retention policy is used.
3. Expand **Retention**. Observe that time-based retention is used. Events are retained for 30 seconds.

## Test the Project and View the Results

When you test the project, the results for each window appear on separate tabs. The following figure shows the results for the sourceWindow tab. This tab displays all events, with various opcodes. 

![sourceWindow tab](img/sourceWindow.png "sourceWindow tab")

The following figure shows the results for the removestateWindow tab. The eventNumber, originalOC, and originalFL columns are present. The originalOC column shows that only Insert events entered into the window, as specified in the window's settings.

![removestateWindow tab](img/removestateWindow.png "removestateWindow tab")

The following figure shows the results for the copyWindow tab. This tab initially displays 13 Insert events. The window retains these events for 30 seconds. After this, 13 Delete events appear.

![copyWindow tab](img/copyWindow.png "copyWindow tab")

## Next Steps

To understand more about how Remove State windows work:
1. Stop the test and return to the removestateWindow window. 
2. In **Settings**, clear all checkboxes for the **Opcodes to be removed** option.
3. Save the project and test it again. 
4. Select the removestateWindow tab. The originalOC column shows that events with various opcodes entered into the window. However, the Opcode column shows that each event has been turned into an Insert. A Remove State window converts all events that it receives into Inserts.

## Additional Resources
For more information, see [SAS Help Center: Using Remove State Windows](https://documentation.sas.com/?cdcId=espcdc&cdcVersion=default&docsetId=espcreatewindows&docsetTarget=p0usk3uf3bcnebn1m99g1jbvvhxu).
