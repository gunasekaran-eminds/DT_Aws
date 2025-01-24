# Configuring IoT Core Rules and SiteWise Asset Hierarchy for Fryer Data

This document outlines the steps to configure AWS IoT Core rules and establish a hierarchy in AWS IoT SiteWise to process sensor data from the `DT/Fryer` topic, which contains two partitions: **Single Vat** and **Double Vat**.

---

## 1. **Data Structure**

### **Single Vat Example:**
```json
{
  "partition": "Single Vat",
  "sensors": {
    "temperature": {"sensor_id": "sensor_1", "value": 23.43, "timestamp": 1737352762506},
    "humidity": {"sensor_id": "sensor_2", "value": 50.21, "timestamp": 1737352762506},
    "electricity": {"sensor_id": "sensor_3", "value": 120.89, "timestamp": 1737352762506},
    "oil_level": {"sensor_id": "sensor_4", "value": 300.63, "timestamp": 1737352762506}
  }
}
```

### **Double Vat Example:**
```json
{
  "partition": "Double Vat",
  "sensors": {
    "temperature": {"sensor_id": "sensor_1", "value": 25.12, "timestamp": 1737352762506},
    "humidity": {"sensor_id": "sensor_2", "value": 55.80, "timestamp": 1737352762506},
    "electricity": {"sensor_id": "sensor_3", "value": 125.78, "timestamp": 1737352762506},
    "oil_level": {"sensor_id": "sensor_4", "value": 310.45, "timestamp": 1737352762506},
    "vibration": {"sensor_id": "sensor_5", "value": 0.45, "timestamp": 1737352762506}
  }
}
```

---

## 2. **IoT SiteWise Asset Hierarchy**

### **Hierarchy Structure**

#### **Top-Level Asset Model:** DT
- Represents the root of the hierarchy (e.g., a factory or equipment group).

#### **Second-Level Asset Model:** Fryer
- Represents individual fryers.

#### **Third-Level Asset Models:**
- **Single Vat**: Represents fryers with a single vat.
- **Double Vat**: Represents fryers with double vats.

### **Hierarchy Diagram**
```
DT (Asset Model)
└── Fryer (Asset Model)
    ├── Single Vat (Asset Model)
    └── Double Vat (Asset Model)
```

---

### **Detailed Asset Model Definitions**

#### **1. Asset Model: DT**
- **Name**: DT Model
- **Properties**:
  - None (this is purely for hierarchy).
- **Child Assets**:
  - Fryer (inherits Fryer Model).

#### **2. Asset Model: Fryer**
- **Name**: Fryer Model
- **Properties**:
  - Fryer ID (string).
  - Location (string).
  - Status (enum: {Active, Inactive, Maintenance}).
- **Child Assets**:
  - Single Vat (inherits Single Vat Model).
  - Double Vat (inherits Double Vat Model).

#### **3. Asset Model: Single Vat**
- **Name**: Single Vat Model
- **Properties**:
  - Temperature (double, unit: °C).
  - Humidity (double, unit: %).
  - Electricity (double, unit: kWh).
  - Oil Level (double, unit: liters).
- **No Child Assets**.

#### **4. Asset Model: Double Vat**
- **Name**: Double Vat Model
- **Properties**:
  - Temperature (double, unit: °C).
  - Humidity (double, unit: %).
  - Electricity (double, unit: kWh).
  - Oil Level (double, unit: liters).
  - Vibration (double, unit: mm/s).
- **No Child Assets**.

---

## 3. **IoT Core Rule Configuration**

### **SQL Query for Rule**

#### **General Query:**
```sql
SELECT
  sensors.temperature.value AS temperature_value,
  sensors.temperature.timestamp AS temperature_timestamp,
  sensors.humidity.value AS humidity_value,
  sensors.humidity.timestamp AS humidity_timestamp,
  sensors.electricity.value AS electricity_value,
  sensors.electricity.timestamp AS electricity_timestamp,
  sensors.oil_level.value AS oil_level_value,
  sensors.oil_level.timestamp AS oil_level_timestamp,
  sensors.vibration.value AS vibration_value,
  sensors.vibration.timestamp AS vibration_timestamp,
  partition
FROM
  'DT/Fryer'
```

#### **Single Vat Rule:**
```sql
SELECT * FROM 'DT/Fryer' WHERE partition = 'Single Vat'
```

#### **Double Vat Rule:**
```sql
SELECT * FROM 'DT/Fryer' WHERE partition = 'Double Vat'
```

---

### **Rule Actions**

#### **Single Vat Action:**
Map properties for the Single Vat asset to IoT SiteWise.
- **Temperature** → Asset: `SingleVatAsset`, Property: `Temperature`
- **Humidity** → Asset: `SingleVatAsset`, Property: `Humidity`
- **Electricity** → Asset: `SingleVatAsset`, Property: `Electricity`
- **Oil Level** → Asset: `SingleVatAsset`, Property: `OilLevel`

#### **Double Vat Action:**
Map properties for the Double Vat asset to IoT SiteWise.
- **Temperature** → Asset: `DoubleVatAsset`, Property: `Temperature`
- **Humidity** → Asset: `DoubleVatAsset`, Property: `Humidity`
- **Electricity** → Asset: `DoubleVatAsset`, Property: `Electricity`
- **Oil Level** → Asset: `DoubleVatAsset`, Property: `OilLevel`
- **Vibration** → Asset: `DoubleVatAsset`, Property: `Vibration`

---

## 4. **Steps to Create Asset Models in IoT SiteWise**

1. **Create Asset Models**:
   - Navigate to **AWS IoT SiteWise Console > Models**.
   - Create models in the following order:
     1. DT Model.
     2. Fryer Model (set DT Model as parent).
     3. Single Vat Model (set Fryer Model as parent).
     4. Double Vat Model (set Fryer Model as parent).

2. **Define Properties**:
   - Add relevant properties to each model.
   - Specify types, units, and data types (e.g., double for measurements).

3. **Create Assets**:
   - Create an instance of the **DT Model**.
   - For each fryer, create an instance of the **Fryer Model** under the DT asset.
   - Attach **Single Vat** or **Double Vat** assets as children.

4. **Test Setup**:
   - Publish sample MQTT messages to the `DT/Fryer` topic.
   - Verify that data flows correctly to the respective asset properties.

---

## 5. **Example JSON for IoT Rules (CLI/API)**

### **Single Vat Rule**
```json
{
  "sql": "SELECT * FROM 'DT/Fryer' WHERE partition = 'Single Vat'",
  "ruleDisabled": false,
  "actions": [
    {
      "iotSiteWise": {
        "putAssetPropertyValueEntries": [
          {
            "assetId": "SingleVatAssetId",
            "propertyId": "TemperaturePropertyId",
            "propertyValues": [
              {
                "value": { "doubleValue": "${sensors.temperature.value}" },
                "timestamp": { "timeInSeconds": "${sensors.temperature.timestamp}" },
                "quality": "GOOD"
              }
            ]
          }
        ]
      }
    }
  ]
}
```

---

By following the steps above, you can configure AWS IoT Core rules and SiteWise assets to handle and process data from your fryers, maintaining a clear and logical hierarchy.
