# Connectivity analytics for telemetry data in telecom networks

## Vocabulary

| Term | Definition |
|------|------------|
| **CMP**  | Connectivity Management Platform — a cloud-based system for managing IoT/M2M device connectivity, SIMs, and data usage. |
| **IoT**  | Internet of Things — a network of connected devices that collect and exchange data. |
| **SIM**  | Subscriber Identity Module — a smart card that stores network authorization information. |
| **M2M**  | Machine-to-Machine — direct communication between devices over a network without human intervention. |
| **telemetry data**  | Refers to the automated collection and transmission of data from network devices, equipment, or endpoints to a central system for monitoring, analysis, and management. |

## Overview

This example demonstrates how to use **SAS Event Stream Processing** to analyze telemetry data from a Connectivity Management Platform (CMP), detect anomalies for specific devices, and raise real-time alerts.
For more information about how to install and use example projects, see [Using the Examples](https://github.com/sassoftware/esp-studio-examples#using-the-examples).

## Use Case

This demonstration showcases different approaches for monitoring CMP data. Synthetic data is used with three event samples, and for each sample a specific stream processing technique is applied:
- Device outside coverage area (e.g., tunnel): Events are detected using processing rules on streaming event data.
- Attacked device or malware: Detection is based on rules applied to aggregated data within a specific time frame.
- Equipment failures due to high demand: Change Detection using the Kullback–Leibler (KL) Divergence approach is applied.

## Source Data and Other Files

- The source window uses a File and Socket Connector to ingest a stream of synthetic data from `anomalies.csv` that includes:

  - `timestamp`: Unique identifier
  - `device_id`: Unique device identifier
  - `iccid`: Unique network cell identifier
  - `status`: Device status
  - `data_usage_mb`: Device network traffic usage
  - `signal_strength`: Device connection signal strength
  - `latency_ms`: Latency in microseconds in receiving/sending network packages
  - `jitter_ms`: Variation in latency (packets delay) over time
  - `session_status`: Connection session status for currently registered network
  - `location`: Device home region - country name
  - `roaming`: If current region is home region for the device
  - `operator`: Network provider where device currently registered
  - `plan`: Network provider tariff

- To make changes in synthetic data can be used source Python script in Jupyter Notebok format: `Generate_anomalies.ipynb`
- (Optional) Grafana Dashboards can be imported from file `grafana.json`
- Demonstration recording `Grafana.mp4`

<img alt="Demo process" src="img/demo.png" width="700">

## Prerequisites

There are no special prerequisites to run this example. It uses standard components of SAS Event Stream Processing and runs in both local and edge server environments.
However, it also includes optional Grafana dashboards for visualization of alerts and  streaming data. To be able to use this component follow the instructions in the [Visualizing Alerts in Grafana](#Visualizing-Alerts-in-Grafana) chapter:

## Workflow

The following figure shows the diagram of the project:

<img alt="Diagram of the project" src="img/diagram.png" width="300">

In this workflow we generate alerts on streaming data using conditions on parsed input events data as well as aggregation and unsupervised analytic model. In following description we will focus on the main project logic, for simple transformations like `Source`, `Filter`, `Aggregate`, `Copy` and `Join`,  please  refer to relevant examples for more details. 

### w_cmp_stream

Receives synthetic CMP data from the file; the message data is written to json_data text field in JSON format.

### w_parsing

Applies standard JSON message parsing using `esp_parseJsonFrom` function.
```lua
function create(data,context)
    local   message_info = esp_parseJsonFrom("json_data")
    if message_info == nil then
        print("ERROR: Failed to parse message")
        return {}
    end
    local   e = {}
    e.timestamp = esp_getSystemMicro()
    e.device_id = message_info.device_id
    e.iccid = message_info.iccid
    e.status = message_info.status
    e.data_usage_mb = message_info.data_usage_mb
    e.signal_strength = message_info.signal_strength
    e.latency_ms = message_info.latency_ms
    e.jitter_ms = message_info.jitter_ms
    e.session_status = message_info.session_status
    e.location = message_info.location
    e.roaming = message_info.roaming
    e.operator = message_info.operator
    e.plan = message_info.plan
    return(e)
end
```
### w_retention/w_usage_profile/w_join_avg

Standard design pattern to enrich incoming event with aggregated information.
This applies data retention for the last 4 days (w_retention), aggregates average data usage `data_usage_mb` for each `device_id` and whether the device is in roaming or not - `roaming` (w_usage_profile). Each current average value is joined back to the input event as a new field `data_usage_avg_mb` (w_join_avg).

### w_change_latency

For anomaly detection in latency data from CMP we are using one of supported algorithms based on [Kullback-Leibler (KL) Divergence]( https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence).


<p align="center"><img alt="Diagram of the project" src="https://go.documentation.sas.com/api/docsets/espan/v_047/content/images/equation67.svg?locale=en" width="300"/></p>

It helps in identifying sudden or unexpected deviations in a time series or data stream. 
For more details, refer to [SAS Help Center: Change Detection](https://go.documentation.sas.com/doc/en/espcdc/v_063/espan/n0jveogr5iwzyxn1w7imhj73d4zf.htm).
The Calculate Window `w_change_latency` uses the KLDivergenceDiff measure, periodically updates the model using incoming data and provides decisions on change detection.

Explore the setting for the `w_change_latency window` window by doing the following steps:
1. Select the w_change_latency window.
2. Expand **Settings** and then expand **Parameters**.  
Parameters:
    - `slidingAlpha`: 0.98, Specifies the fading factor for the sliding window. Value range is 0 < α<=1.
    - `slidingHalfLifeSteps`: 0, Specifies the number of steps at which the weight of the input reaches half of its original weight for the sliding window.
    - `refWindowSize`: 100, Specifies the size of the reference window.
    - `changeThreshold`: 0.2, Specifies the threshold to determine whether a change occurred.
    - `nBins`: 50, Specifies the maximum number of bins in the histogram for both reference and sliding windows, and the number of bins when computing KL divergence. 
    - `maxEvalSteps`: 100, Specifies the maximum number of steps before performing a new evaluation.
    - `adaptiveEval`: 1, Specifies whether to use the adaptive evaluation step size or not.
    - `measure`: KLDivergenceDiff, Specifies the measure used to compare the data streams from the reference window and the sliding window. 
    - `showEval`: 1, Specifies whether to show evaluation events regardless of whether a change is detected.
    - `showAll`: 1, Specifies whether to show all events, regardless of whether an evaluation occurs.
      
3. Expand **Input Map**.  
Input Map:
    - `inputs`: `latency_ms`, Specifies the input variable for change detection. In our example we are analysing registered signal latency.


4. Expand **Output Map**.  
Output Map:
    - `evaluatedOut`: `eval`, Specifies the name of the output variable that indicates whether an evaluation occurred.
    - `changeValueOut`: `changeVal`, Specifies the name of the output variable that contains the change value (the difference between two KL divergence values)
    - `changeDetectedOut`: `changeDetected`, Specifies the name of the output variable that indicates whether a change has been detected. We use this variable in the `w_latency_spike` window below.
      
### w_latency_spike

Filters out events, keeping only where `changeDetected` is equal 1.

### w_cells/add_cell_pos
Reads latitude and longitude data for cell_id (w_cells) and joins back these values to the input event (add_cell_pos).
New fields are `cell_id_lat` and `cell_id_lon`.
For demo purposes, only one location will be added.

### w_rules

Lua window which implements alerts generation logic.
Here there will be 3 types of alerts:


| alert_type | alert_description |
|------|------------|
| **ANOMALY_NO_SIGNAL**  | Device outside coverage area |
| **ANOMALY_USAGE**  | Attacked device or malware |
| **ANOMALY_LATENCY_SPIKE**  | Equipment failures due to high demand |

As you can see in the code below we simply apply rules for alert generation on 
- input event fields which we parsed from input JSON
- fields  which we aggregated based on previous events
- events where change detected by analytics algorithm 

```lua
local alert_id = 1
function create(data,context)
    local   alerts = {}
    local array_idx = 1
    -- Monitoring rules --
    -- RULE 1: zero data, no signal
    if context.input=="w_join_avg" and data.session_status == "dropped" and data.status == "inactive" and data.data_usage_mb == 0.0 then
        local alert = {}
        alert.alert_id = alert_id
        alert.id = data.id
        alert.timestamp = data.timestamp
        alert.device_id = data.device_id
        alert.iccid = data.iccid
        alert.alert_type = esp_toString("ANOMALY_NO_SIGNAL")
        alert.alert_description = esp_toString("Device outside coverage area")
        alert.alert_priority = 2
        alert.cell_id_lat =  0
        alert.cell_id_lon = 0
        alerts[array_idx] = alert
        alert_id = alert_id + 1
        array_idx = array_idx + 1
    end
    -- RULE 2: unusual high traffic consumption
    if context.input=="w_join_avg" and ((data.data_usage_mb -  data.data_usage_avg_mb)>1000) and (data.data_usage_mb ~= 0)   then
        local alert = {}
        alert.alert_id = alert_id
        alert.id = data.id
        alert.timestamp = data.timestamp
        alert.device_id = data.device_id
        alert.iccid = data.iccid
        alert.alert_type = esp_toString("ANOMALY_USAGE")
        alert.alert_description = esp_toString("Attacked device or malware")
        alert.alert_priority = 3
        alert.cell_id_lat =  0
        alert.cell_id_lon = 0
        alerts[array_idx] = alert
        alert_id = alert_id + 1
        array_idx = array_idx + 1
    
    end
    
    -- Detected anomalies --
    -- higher latency, jitter, dropped session
    if context.input == "add_cell_pos" and data.changeDetected==1  then

        local alert = {}
        alert.alert_id = alert_id
        alert.id = data.id
        alert.timestamp = data.timestamp
        alert.device_id = data.device_id
        alert.iccid = data.iccid
        alert.alert_type = esp_toString("ANOMALY_LATENCY_SPIKE")
        alert.alert_description = esp_toString("Equipment failures due to high demand")
        alert.alert_priority = 1
        alert.cell_id_lat =  24.7743 + math.random(0, 1)/100
        alert.cell_id_lon = 46.7386 + math.random(0, 1)/100
        alerts[array_idx] = alert
        alert_id = alert_id + 1
        array_idx = array_idx + 1
    end
    if array_idx > 1 then
        return(alerts)
    else 
        return false
    end
end
```

### w_rate/w_aggr_stats

Collect aggregated data for few of Grafana dashboards, described  below.

## Test the Project and View the Results
When you test the project in SAS Event Stream Processing Studio, the results of generated alerts will appear after few seconds of processing in the `w_rules` window tab:
![w_rules tab](img/w_rules.png "w_rules tab")

## Visualizing Alerts in Grafana
The alerts, model performance and streaming data can be visualized using the [SAS Event Stream Processing Data Source Plug-in for Grafana](https://github.com/sassoftware/grafana-esp-plugin). Import the [grafana.json](grafana.json) dashboard file to Grafana. 

![streaming data and distribution](img/grafana-1.png "streaming data and distribution")

#### Real-Time Monitor Panel
- **Input Data Table** – Displays CMP events (Uses data from ESP Window `w_cmp_stream`).
- **Rate Gauge** – Shows the events processing rate (`w_rate`). 
#### Analytic Panel
- **Latency distribution histogram** – Displays distribution of jitter_ms, latency_ms and signal_strength (`w_aggr_stats`).
- **KLDivergenceDiff bargauge** - Displays model evaluation (`w_change_latency`).
  
![alerts](img/grafana-2.png "alerts")

#### Alerts Panel
- **group:latency table** – Displays alerts on changes in latency pattern, indicating possible equipment failures due to high demand (`w_rules`).
- **group:signal table** - Displays no signal alerts when a device is outside the coverage area (`w_rules`).
- **group:usage table** – Displays unusual traffic usage pattern alerts when a device probably has malware (`w_rules`).
- **Latency spikes geomap** - Shows the location of devices where latency anomalies are detected (`w_rules`).
- **Logs** – Displays aggregated alert data by alert type (`w_rules`).

[Grafana video](grafana.mp4)

---
**NOTE:**
This dashboard was created using standalone SAS Event Stream Processing, running in the same namespace as Grafana. If you are using a different environment, such as the SAS Viya platform, you must recreate the queries because the connection URLs will differ.

---


## High level target solution architecture
Here is an example of a possible general architecture for a CMP data analysis system using SAS Event Stream Processing:  
![architecture](img/architecture.png "architecture")


## Next Steps

You can enhance this project by:
- Replacing the CSV source with a live sensor feed.
- Adjusting rules logic for new types of anomalies which observed in the input stream.
- Using other change detection methods in the Calculate window to improve detection accuracy where required by the input data.


## Additional Resources

- [SAS Help Center: Change Detection](https://go.documentation.sas.com/doc/en/espcdc/v_063/espan/n0jveogr5iwzyxn1w7imhj73d4zf.htm#p0fydwd77vhobwn1c4c89qbv6r7f)
