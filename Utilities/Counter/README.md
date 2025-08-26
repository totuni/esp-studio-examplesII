# Using Counter Windows to Compare Data Flow
## Overview

This SAS Event Stream Processing project demonstrates the use of the Counter window. A Counter window is a simple but powerful analytic tool used to track the number of events passing through a data stream. This project showcases three different types of source connectors: CSV file, video input, and timed data stream. It pairs each source connector with a corresponding Counter window to monitor throughput. The Counter window is useful for observing data flow rates, detecting anomalies (for example, data dropouts or surges), and measuring performance. It is commonly used in monitoring, debugging, and benchmarking real-time event pipelines.

## Workflow

The following figure shows the workflow of this project:

![image-20250717102057965](img/image-20250717102057965.png)	

This project has three Source windows:

- Source_CSV: Ingests location data from a CSV file using a file system connector. The schema includes a timestamped key along with latitude and longitude coordinates.
- Source_video: Uses a videocap connector to input video frames from a sample video file. Each frame is treated as an event, with an id and image blob.
- Source_timed: Uses a timer connector to emit synthetic events at fixed intervals. Each event includes an id, timestamp, and label.

Each Source window feeds directly into its corresponding Counter window. The Counter windows increment a count each time they receive a new event from their upstream source. This provides a clear and real-time metric of how many events have passed through each stream. This setup allows direct comparison of data flow behavior across different source types.

## Test the Project and View the Results

To test the project, do the following steps:
1. Select the **Source_CSV** window.
2. Expand **Subscriber Connectors**.
3. Deselect the **Active** check box for the **MQTToutput** connector.
4. Click save.
5. Click **Enter Test Mode**.
6. Click **Run Text**.

When you test the project, the results for each window appear on separate tabs. The main idea of this project is how event volume and counter output varies significantly depending on the Source window type and connector configuration.

###  Source_CSV with File Connector

The following figure shows the results for the Source_CSV tab:

![image-20250717102824215](img/image-20250717102824215.png)

- The Source_CSV window reads a file containing many rows of data.

- When using an unrestricted file connector (e.g., repeatcount without throttling), it can emit hundreds or thousands of events very quickly.

- As a result, the Counter_CSV window often shows the highest count in the shortest time.

- This makes it ideal for stress testing or measuring batch ingestion rates.

###  Source_video with VideoCap Connector

The following figure shows the results for the Source_video tab:

![image-20250717102847154](img/image-20250717102847154.png)

- The Source_video window reads a video file and emits one event per frame, based on the inputrate and blocksize.

- In this case, the video connector is configured to emit at 10 frames per second.

- As a result, the Counter_video window reflects a moderate and consistent stream of events.

- The count increases steadily but more slowly than the CSV source.

### Source_timed with Timer Connector

The following figure shows the results for the Source_timed tab:

![image-20250717102919833](img/image-20250717102919833.png)

- The Source_timed window is designed to emit one event every 60 seconds using the timer connector.
- This makes it by far the slowest source in terms of event generation.
- Accordingly, Counter_timed increases very gradually, making it useful for low-rate event simulation and benchmarking long-running processes.

Understanding the differences between source connectors is critical for debugging, system benchmarking, and monitoring. By comparing the event rates across the Counter windows, this project clearly shows how different source types affect the overall streaming pipeline performance.

## Video Credits and Copyright



| File Name          | Copyright                                      | Notes                                        |
| ------------------ | ---------------------------------------------- | -------------------------------------------- |
| `intersection.mp4` | © 2024 SAS Institute Inc. All Rights Reserved. | To be used only in the context of this demo. |

## Video and Image Restrictions

The videos and images provided in this example are to be used only with the project provided. Using or altering these videos and images beyond the example for any other purpose is prohibited.

## Additional Resources

For more information about the Counter window, see [SAS ESP Counter Window Documentation](https://go.documentation.sas.com/doc/en/espcdc/default/espcreatewindows/titlepage.htm).
