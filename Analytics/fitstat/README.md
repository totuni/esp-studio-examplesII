# FitStat in SAS Event Stream Processing (ESP)

## Overview

The FitStat capability in **SAS Event Stream Processing (ESP)** is not exposed as a standard window in the ESP Studio graphical user interface like Pattern, Calculate, or Filter windows. Instead, FitStat is a **function inside the Calculate window** and is specifically associated with **model scoring windows** when working with analytics models like logistic regression, decision trees, or neural networks.

### Where You Encounter FitStat in ESP

- **Model Reader / Score Windows**  
  If you load a predictive model (for example, from SAS Viya using an ASTORE or ONNX model), the scoring window will output a FitStat field. This field contains fit statistics relevant to the model, such as:  
  - Misclassification rate  
  - Log-likelihood  
  - R-squared  
  - AIC/BIC (depending on the model)

- **Calculate Window**  
  You can invoke FitStat as an algorithm within a Calculate window to compute error metrics on predictions flowing through the stream.

- **XML/JSON Code View**  
  While the ESP Studio UI doesn’t explicitly show a "FitStat window," the XML or JSON definition of an ESP project can include FitStat as a configured algorithm within a Calculate window or as an output field from a model scoring window.

---

## Source

### How to See or Use FitStat

1. Add a **Score window**, connected to a Model Reader that loads an analytic store (like ASTORE or ONNX).
2. After running the project or inspecting the schema, FitStat appears among the output fields.
3. Downstream windows (Filter, Calculate, Alert, etc.) can reference FitStat values for thresholds, monitoring, or alerts.

---

## Workflow

### Window Definition Example

```xml
<window-calculate algorithm="FitStat" pubsub="true" index="pi_EMPTY" name="w_calculate">
```
