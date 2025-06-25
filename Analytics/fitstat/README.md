# fitstat

The `FITSTAT` window in **SAS Event Stream Processing (ESP)** is not directly exposed as a standard window in the ESP Studio graphical user interface (UI) like Pattern, Calculate, or Filter windows. Instead, `FITSTAT` is a **function inside the Calculate window**, and it's specifically associated with **model scoring windows** when you are working with analytics models like logistic regression, decision trees, or neural networks.

### Where You See `FITSTAT` in the SAS ESP UI:

- **Model Reader / Score Windows**: If you load a predictive model (for example, from SAS Viya using an ASTORE or ONNX model), the scoring window will output a `FITSTAT` field. This field contains fit statistics relevant to the model, such as:
  - Misclassification rate
  - Log-likelihood
  - R-squared
  - AIC/BIC (depending on the model)
- **Calculate Window**: You can use functions similar to `FITSTAT` or access individual fit-related metrics as part of expressions, especially when doing post-scoring calculations or quality checks.
- **XML/JSON Code View**: While the drag-and-drop UI might not show something called `FITSTAT` explicitly as a window, if you inspect the XML or JSON definition of the ESP project, `FITSTAT` appears as an output field from model scoring operators.

### How to See or Use It:

1. When adding a **Score window**, connect it to a Model Reader that loads an analytic store.
2. After running the project or viewing the schema, you'll see `FITSTAT` among the output fields.
3. You can then use downstream windows (like Filter, Calculate, or Alert) to act on `FITSTAT` values for thresholds or error monitoring.

### Working example:

This XML block defines a **Calculate window** in SAS ESP that uses the **`FitStat` algorithm** to compute *goodness-of-fit statistics* for a regression or classification model running in streaming data.

Let’s break it down step-by-step:

------

## 🔧 **Window Definition**

```xml
<window-calculate algorithm="FitStat" ... name="w_calculate">
```

- `window-calculate`: This defines a Calculate window.
- `algorithm="FitStat"`: This specifies that the window uses the built-in `FitStat` algorithm, which is a special-purpose algorithm for calculating fit statistics like error metrics on the fly.
- `pubsub="true"`: This enables publishing and subscribing to events.
- `index="pi_EMPTY"`: No indexing is applied — it's a pass-through, stateless calculation.
- `name="w_calculate"`: The name of this window.

------

## 📝 **Description Block**

```xml
<description><![CDATA[...]]></description>
```

- Helpful documentation explaining that this window calculates fit statistics.
- Clarifies that in regression models, it expects:
  - A **predicted variable** (`inputs`) — e.g., predicted values from a model.
  - A **response variable** (`response`) — the actual observed value.
- Lists the output metrics and notes that some (like `mceOut` and `mcllOut`, which are classification metrics) will be blank when used for regression.

------

## 🗺️ **Schema Section**

```xml
<schema>
  <fields>
    ...
  </fields>
</schema>
```

This defines the **output fields** that the window will produce. Each field is either a key or a computed metric.

| Field      | Description                               |
| ---------- | ----------------------------------------- |
| `id`       | Unique key (int64)                        |
| `nOut`     | Number of observations (N)                |
| `nmissOut` | Number of missing values                  |
| `aseOut`   | Average squared error (ASE)               |
| `divOut`   | Divisor used for ASE                      |
| `raseOut`  | Root ASE                                  |
| `mceOut`   | Mean consequential error (classification) |
| `mcllOut`  | Multiclass log loss (classification)      |
| `maeOut`   | Mean absolute error                       |
| `rmaeOut`  | Root mean absolute error                  |
| `msleOut`  | Mean squared logarithmic error            |
| `rmsleOut` | Root MSLE                                 |



\* For regression, `mceOut` and `mcllOut` are typically unused.

------

## ⚙️ **Parameters Section**

```xml
<parameters>
  <properties>
    <property name="windowLength">0</property>
    <property name="labelLen">128</property>
  </properties>
</parameters>
```

- `windowLength=0`: No sliding window — it calculates over all data seen.
- `labelLen=128`: Maximum label length (probably related to string metadata, not the calculation itself).

------

## 🔗 **Input Map**

```xml
<input-map>
  <properties>
    <property name="inputs">y_c</property>
    <property name="response">x_c</property>
  </properties>
</input-map>
```

- `inputs="y_c"`: The **predicted values** from your model (y_hat equivalent).
- `response="x_c"`: The **actual observed target values** (ground truth).

This is key: it tells FitStat what to compare — predictions vs. actuals.

------

## 📤 **Output Map**

```xml
<output-map>
  <properties>
    <property name="nOut">nOut</property>
    ...
  </properties>
</output-map>
```

This maps internal computation results to the outgoing schema fields.

Example:

- Internal ASE result ➝ output field `aseOut`.

------

## 🔍 **Summary of How It Works**

- This window takes predicted (`y_c`) and actual (`x_c`) values from a stream.
- It computes fit statistics like RMSE (`raseOut`), MAE (`maeOut`), and others in real-time as data flows through.
- Outputs these stats as part of the event stream, where downstream windows or consumers can pick them up for monitoring, alerts, or dashboards.

------

## ✅ **Why Use FitStat in ESP?**

- **Live Model Monitoring:** See if model accuracy is degrading in production.
- **Streaming Validation:** Detect data drift or issues without batch jobs.
- **Alerts:** Trigger automated actions when errors cross a threshold.
