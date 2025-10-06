# Connectivity Analytics for Telemetry Data in Telecom Networks

## Overview

This example demonstrates how to use SAS Event Stream Processing to analyze telemetry data from a Connectivity Management Platform (CMP). Telemetry data refers to the automated collection and transmission between devices over a network without human intervention. The Connectivity Management Platform is a cloud-based system that manages the Internet of Things (IoT) and machine-to-machine (M2M) device connectivity. The CMP also manages subscriber identity modules (SIM) and data usage.

For more information about how to install and use example projects, see [Using the Examples](https://github.com/sassoftware/esp-studio-examples#using-the-examples).

## Use Case

This example shows different approaches for monitoring CMP data. Three event samples use synthetic data, and for each sample, a specific stream processing technique is applied: <!-- so the event samples are outside coverage area, attacked device, and equipment failures? A better way to word these bullets would be: For the ___ event sample, the ____ stream processing technique is applied. -->
- Device outside coverage area: An example of this would be a tunnel, where there is no signal. Events are detected using processing rules on streaming event data.
- Attacked device or malware: Event detection is based on rules applied to aggregated data within a specific time frame.
- Equipment failures due to high demand: Change detection using the Kullback-Leibler (KL) Divergence approach is applied.

## Source Data and Other Files

- The source window uses a file and socket connector to ingest a stream of synthetic data from an input file called `anomalies.csv` that includes:

  - `timestamp`: Unique identifier
  - `device_id`: Unique device identifier
  - `iccid`: Unique network cell identifier
  - `status`: Device status
  - `data_usage_mb`: Device network traffic usage
  - `signal_strength`: Device connection signal strength
  - `latency_ms`: Latency in microseconds in receiving or sending network packages
  - `jitter_ms`: Variation in latency over time
  - `session_status`: Connection session status for the currently registered network
  - `location`: Country name of the device home region
  - `roaming`: If current region is home region for the device
  - `operator`: Network provider where device currently registered
  - `plan`: Network provider tariff

The following diagram shows the main demo components and the data flow. Prepared CMP data is published to SAS ESP at a specific rate. SAS ESP preprocesses the data, detects anomalies, applies rules, and finally generates alerts. Grafana dashboards connects to various SAS ESP windows to visualize alerts and statistical data from the project.

<!-- please add one or two sentence describing the diagram below --><!-- A: added-->
<img alt="Demo process" src="img/demo.png" width="700">

<!-- A: Where is Python Notebok file?  Here is proposed text:

To make changes to the synthetic data sample, follow these steps:
1. Navigate to the Git project.
2. Download the file [Grafana video](Generate_anomalies.ipynb).
3. Open the notebook in **Jupyter Notebook**.
4. Make the necessary changes to the data generation steps and run the code.
5. Download the new version of **`anomalies.csv`**.
6. Replace the existing file in the **SAS ESP project package** with the updated one.
-->

## Workflow

The following figure shows the diagram of the project:

<img alt="Diagram of the project" src="img/diagram.png" width="300">

- w_cmp_stream: A Source window that receives synthetic CMP data from the file. 
- w_parsing: A Lua window that applies standard JSON message parsing using the **esp_parseJsonFrom** function.
- w_retention: A Copy window that applies a data retention policy of four days.
- w_usage_profile: An Aggregate window that aggregates average data usage for each device ID and indicates whether a device is roaming.
- w_join_avg: A Join window that joins back current average values to the Input event as a new field.
- w_change_latency: A Calculate window that detects anomalies in latency data from CMP. The supported algorithms are based on the Kullback-Leibler (KL) Divergence approach. 
- w_latency_spike: A Filter window that filters out events and keeps only the events where `changeDetected` is equal one.
- w_cells: A Source window that reads latitude and longitude data for `cell_id`.
- add_cell_pos: A Join window that joins back the values from w_cells to the Input event.
- w_rules: A Lua window that implements alert generation logic.
- w_rate: A Counter window that calculates the event processing rate for visualization on a Grafana Gauge chart. 
- w_copy: A Copy window that retains input events for aggregation in the subsequent `w_aggr_stat` Window.
- w_aggr_stats: An Aggregate window that computes data metrics for Grafana charts.
<!-- fill in missing information above please --><!--updated  -->
### w_cmp_stream

Explore the settings for the w_cmp_stream window:
1. Open the project in SAS Event Stream Processing Studio and select the w_cmp_stream window.
2. In the right pane, expand **Input Data (Publisher) Connectors**. Notice the connector called **anomalies_Connector**, which is the file and socket connector used to ingest the synthetic CMP data.  
3. Click **output schema icon**. Notice the following fields:
    - `id`: Autogenerated ESP event key field. <!-- explain ID here -->
    - `json_data`: The message data text field written in JSON format.
<!-- is message data the same as output data? --> <!-- A: Yes. At this step, we only generate an ESP event based on an input row from the CSV file.  -->

### w_parsing

Explore the settings for the w_parsing window:
1. Open the project in SAS Event Stream Processing Studio and select the w_parsing window.
2. In the right pane, expand **Lua Settings**.
3. Under **Code source**, you see the following window of code:

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

In this code, we use the built-in ESP JSON parser function `esp_parseJsonFrom` to extract important JSON data fields into ESP event fields for further processing.

<!-- ok so what is the significance of this code? Does this code the logic of the function that is applied in this window? Try to write a sentence or two here explaining. --><!-- A:  added -->
### w_retention, w_usage_profile, and w_join_avg

These three windows use a standard design pattern to enrich incoming events with aggregated information.

Explore the settings for the w_retention window:
1. Open the project in SAS Event Stream Processing Studio and select the w_retention window.
2. In the right pane, expand **Retention**. Notice that the **Time limit** is set to four days, which means that data will be retained for the previous four days.

Explore the settings for the w_usage_profile window:
1. Open the project in SAS Event Stream Processing Studio and select the w_usage_profile window.
2. Click **output schema icon**. Notice the following fields:
    - `device_id`:The unique ID of each device. <!-- is this correct? --><!-- A: yes -->
    - `roaming`: Displays true of false to indicate if a device is roaming.
    - `data_usage_md`: The average data usage in megabytes of each device. <!-- A: yes -->

Explore the settings for the w_join_avg window:
1. Open the project in SAS Event Stream Processing Studio and select the w_join_avg window.
2. In the right pane, expand **Join Criteria**. Notice that the **Join type** is set to left outer.
3. Expand **Join Conditions**. Notice that this join combines data from w_parsing and w_usage_profile.

<!-- is there anything you want to point out in the settings of these windows? I wrote all of these steps based on the descriptions that you had of the windows. Feel free to add more if you would like. --> <!-- A:  I wanted to avoid overwhelming the reader with too much detail, while still keeping the main logic clear -->

### w_change_latency

This window identifies sudden or unexpected deviations in a time series or data stream. It uses the KLDivergenceDiff measure that is pictured directly below: <!-- is this correct? --><!-- A:yes -->
<p align="center"><img alt="Diagram of the project" src="https://go.documentation.sas.com/api/docsets/espan/v_047/content/images/equation67.svg?locale=en" width="300"/></p>

For more details, refer to [SAS Help Center: Change Detection](https://go.documentation.sas.com/doc/en/espcdc/v_063/espan/n0jveogr5iwzyxn1w7imhj73d4zf.htm).

Explore the setting for the w_change_latency window:
1. Open the project in SAS Event Stream Processing Studio and select the w_change_latency window.
2. Expand **Settings** and then expand **Parameters**. See the following parameters:  
    - `slidingAlpha`: Specifies the fading factor for the sliding window. The value range is 0<α<=1, and the default value is **0.98**.
    - `slidingHalfLifeSteps`: Specifies the number of steps at which the weight of the input reaches half of its original weight for the sliding window. The default value is **0**.
    - `refWindowSize`: Specifies the size of the reference window. The default value is **100**.
    - `changeThreshold`: Specifies the threshold that determines whether a change occurred. The default value is **0.2**. 
    - `nBins`: Specifies the maximum number of bins in the histogram for reference and sliding windows. This value also determines the number of bins when computing KL divergence. The default value is **50**. 
    - `maxEvalSteps`: Specifies the maximum number of steps before performing a new evaluation. The default value is **100**.
    - `adaptiveEval`: Specifies whether to use the adaptive evaluation step size or not. The default value is **1, which means the adaptive evaluation step size is used. <!-- I'm guessing 1 is yes and 0 is no? Like binary? --> <!-- A: right ,  according to doc: adaptiveEval is int32 and default value is 1 which means "true" -->
    - `measure`: Specifies the measure used to compare the data streams from the reference window and from the sliding window. The default value is **KLDivergenceDiff**.
    - `showEval`: Specifies whether to show evaluation events regardless of whether a change is detected. The default value is **0, which means evaluation events are not shown. We set it to `1` for debugging purposes. <!-- fact check this please --> <!-- A: fixed , Valid values are 1 for true and 0 for false. Default is 0  -->
    - `showAll`: Specifies whether to show all events, regardless of whether an evaluation occurs. The default value is **0, which means all events are not shown. We set it to `1` for debugging purposes. <!-- same here --> <!-- A: fixed , Valid values are 1 for true and 0 for false. Default is 0  -->
      
3. Expand **Input Map**.  
    - `input`: Specifies the input variable for change detection. In this example, it analyzes **latency_ms, which is the registered signal latency.
      
4. Expand **Output Map**.  
    - `evaluatedOut`: Specifies the name of the output variable that indicates whether an evaluation occurred. This value is used as a supporting indicator of the anomaly detection algorithm’s performance on the Grafana dashboard.<!-- how does "eval" factor in? --><!-- A: added -->
    - `changeValueOut`: Specifies the name of the output variable that contains the change value. This value is used as a supporting indicator of the anomaly detection algorithm’s performance on the Grafana dashboard. <!-- how does "changeVal" factor in? --><!-- A: added -->
    - `changeDetectedOut`: Specifies the name of the output variable that indicates whether a change has been detected. This value indicates whether an anomaly has been detected in the observed `latency_ms`. It is used in the next `w_latency_spike` window to filter out latency anomalies.  <!-- how does "changeDetected" factor in? --><!-- A: added -->
      
### w_latency_spike

Explore the setting for the w_latency_spike window:
1. Open the project in SAS Event Stream Processing Studio and select the w_latency_spike window.
2. In the right pane, expand **Filter**.
3. Under **Code source**, you see the following window of code:

```lua
function filter(event,context)
    return (event.changeDetected==1)
end
```
This code filters out events and keeps the events where `changeDetected` is equal to one.

### w_cells and add_cell_pos

These two windows... <!-- fill this in with an explanation of how they work together -->

Explore the settings for the w_cells window:
1. Open the project in SAS Event Stream Processing Studio and select the w_cells window.
2. In the right pane, expand **Input Data (Publisher) Connectors**. Notice there is a Lua connector called **geo_data**.
3. Click **output schema icon**. Notice the following fields:
    - `cell_id`: The unique cell ID.
    - `cell_id_lat`: The latitude data that the window reads from `cell_id`.
    - `cell_id_lon`: The longitude data that the window reads from `cell_id`.
  
Explore the settings for the add_cell_pos window:
1. Open the project in SAS Event Stream Processing Studio and select the add_cell_pos window.
2. In the right pane, expand **Join Criteria**. Notice that the **Join type** is set to left outer.
3. Expand **Join Conditions**. Notice that this join combines data from w_latency_spike and w_cells.

### w_rules

There are three types of alerts: ANOMALY_NO_SIGNAL, ANOMALY_USAGE, and ANOMALY_LATENCY_SPIKE. The ANOMALY_NO_SIGNAL alert means that the device is outside the coverage area. The ANOMALY_USAGE alert means that the device is being attacked or there is malware. The ANOMALY_LATENCY_SPICE alert means there are equipment failures due to high demand. This Lua window applies alert generation logic to Input event fields, aggregated fields, and events where change is detected by the analytics algorithm. 

Explore the setting for the w_rules window:
1. Open the project in SAS Event Stream Processing Studio and select the w_rules window.
2. In the right pane, expand **Lua Settings**.
3. Under **Code source**, you see the following window of code:
   
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
This code is the alert generation logic used to create alerts.

### w_rate, w_copy, and w_aggr_stats
<!-- Is there anything in the settings of this window you want the user to explore? If so, please add steps like I did with the  w_retention, w_usage_profile, and w_join_avg windows. --><!-- A:No, they simply serve to illustrate data liveliness and visualize the data distribution. -->
These windows collect aggregated data for Grafana dashboards.
1. The w_rate window window that calculates the event processing rate for visualization on a Grafana Gauge chart. The refresh rate is configured with `count-interval` equel to `1 second`.
2. The w_copy window retains input events for aggregation in the subsequent `w_aggr_stat` Window. It is an ESP best practice to provide any streaming aggregation with an appropriate copy window to limit the number of events retained in ESP memory for aggregation.
3. The w_aggr_stats window that computes data metrics for Grafana charts. We use the output of this window in a Grafana histogram widget to visualize the synthetic data distribution.

<!-- fill in each window's role in this process and how each one contributes to collecting aggregated data -->
<!--A:

we already have it above:

- w_rate: A Counter window that calculates the event processing rate for visualization on a Grafana Gauge chart. 
- w_copy: A Copy window that retains input events for aggregation in the subsequent `w_aggr_stat` Window. 
- w_aggr_stats: An Aggregate window that computes data metrics for Grafana charts.



-->
## Test the Project and View the Results
When you test the project, the results appear on separate tabs. The following figure shows the results for the w_rules tab:  
![w_rules tab](img/w_rules.png "w_rules tab")
<!-- are there any other windows you want to show screenshots of? -->

## High-level Target Solution Architecture
Here is an example of a possible general architecture for a CMP data analysis system using SAS Event Stream Processing:  
![architecture](img/architecture.png "architecture")
<!-- this feels out of place. Could this go somewhere else? Not sure it's even necessary to include. -->

## Next Steps

Alerts, model performance, and streaming data can be visualized using the [SAS Event Stream Processing Data Source Plug-in for Grafana](https://github.com/sassoftware/grafana-esp-plugin). Import [grafana.json](grafana.json) to Grafana ("Dashboards"->"Manage"->"Import"). <!-- reworded. is this correct? --> <!--A: reworded again. grafana.json is a separate Dashboard, we are importing a whole new dashbord in Grafana terms--> The following figures show an example of a Grafana dashboard:

![streaming data and distribution](img/grafana-1.png "streaming data and distribution")
![alerts](img/grafana-2.png "alerts")

**Real-Time Monitor Pane**
- `Input Data`: This table displays CMP events using data from the w_cmp_stream window.
- `Rate (msg/sec)`: This gauge shows the events processing rate in messages per second. Based on the processing speed, you can tell whether SAS ESP is still reading and processing the input CSV or if the process has finished.<!-- is this correct? --><!--A:  yes,  but added new sentence-->

**Analytics Pane**
- `Latency distribution`: This histogram displays the distribution of `jitter_ms`, `latency_ms`, and `signal_strength`.
- `KLDivergenceDiff (latency_ms)`: This bar gauge displays the model evaluation from the w_change_latency window.

**Alerts Pane**
- `group:latency`: This table displays alerts on changes in latency pattern, indicating possible equipment failures due to high demand.
- `group:signal`: This table displays ANOMALY_NO_SIGNAL alerts when a device is outside the coverage area.
- `group:usage`: This table displays ANOMALY_USAGE alerts when a device has malware.
- `Latency spikes`: This geomap shows the location of devices where latency anomalies are detected.
- `Logs`: These logs display aggregated alert data by alert type.

**NOTE:** This dashboard was created using standalone SAS Event Stream Processing running in the same namespace as Grafana. If you are using a different environment, such as the SAS Viya platform, you must re-create the queries because the connection URLs are different.
<!--A:  i think this NOTE is  too far from the grafana.json import step. -->

For more information, see [Grafana video](grafana.mp4).

You can further enhance this project by doing any of the following:
- Replacing the CSV source with a live sensor feed.
- Adjusting rules logic for new types of anomalies that are observed in the Input stream.
- Using other change detection methods in the Calculate window to improve detection accuracy.

## Additional Resources

For more information, see [SAS Help Center: Change Detection](https://go.documentation.sas.com/doc/en/espcdc/default/espan/n0jveogr5iwzyxn1w7imhj73d4zf.htm#p0fydwd77vhobwn1c4c89qbv6r7f).
