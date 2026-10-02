# Autonomous Multi-Sensor Environmental Analytics & Weather Station
> **Academic Benchmark**: Awarded **57/60 (95%)** | International Foundation Programme, University of York, UK

---

## 📌 Executive Summary
This project delivers a field-deployable, high-precision environmental monitoring station engineered around an **Arduino microcontroller**. The system executed real-time multi-site data acquisition for ambient temperature, relative humidity, barometric pressure, and localized wind velocity across an uninterrupted 8-hour testing window (09:00 - 17:00). 

To ensure rugged field portability, the hardware was integrated into a custom-designed, weather-proof **plastic enclosure**. This compact packaging rendered the unit highly portable for hand-carried field measurements across **three distinct testing terrains on the University of York campus**:
1. **Lake Shore** (High humidity, aquatic thermal buffering)
2. **Open Space Field** (Exposed grassy area, unshielded solar radiation)
3. **Campus Plaza Buildings** (Urban micro-climate, thermal mass, wind-tunnel effects)

Data was sampled at **30-minute intervals** across all three campus locations (each separated by a ~5-minute walking distance), enabling a comprehensive cross-sensor evaluation between the **DHT22 and BME280 sensor modules**. The analytics pipeline generated **26 comparative response graphs** to rigorously map environmental micro-climates and sensor performance metrics.

---

## 🛠️ Detailed Hardware & Sensor Architecture

### 1. Temperature & Humidity Probe: DHT22 (AM2302)
* **Operating Mechanism**: Uses a capacitive humidity sensing element and a high-precision Negative Temperature Coefficient (NTC) thermistor.
* **Interface Protocol**: Single-bus custom digital signal protocol (OneWire-style timing).
* **Specifications**:
  * **Temperature Range**: -40°C to +80°C (Accuracy: ±0.5°C)
  * **Humidity Range**: 0% to 100% RH (Accuracy: ±2% RH)
  * **Sampling Rate**: Low frequency (Maximum 0.5 Hz / 1 sample every 2 seconds)
* **Engineering Context**: Deployed as the secondary/baseline environmental probe to evaluate response lag against thermal fluctuations.

### 2. Micro-Electromechanical System (MEMS) Sensor: BME280
* **Operating Mechanism**: Integrated environmental chip combining piezo-resistive pressure, capacitive humidity, and low-noise temperature sensing.
* **Interface Protocol**: High-speed **I2C Bus Protocol** (Device Address: `0x76` / `0x77`).
* **Specifications**:
  * **Temperature Accuracy**: ±1.0°C (Optimized for internal thermal drift compensation)
  * **Humidity Accuracy**: ±3% RH with minimal hysteresis
  * **Barometric Pressure**: 300 hPa to 1100 hPa (Absolute accuracy: ±1.0 hPa)
  * **Sampling Rate**: Up to 157 Hz (Programmed via non-blocking timing loops)
* **Engineering Context**: Primary sensor for barometric pressure and high-frequency thermal response testing.

### 3. Mechanical Design & Auxiliary Integration
* **Portable Plastic Enclosure**: Custom-modified, water-resistant plastic housing engineered for rapid hand-held transit between campus site locations while exposing sensing elements to external airflow.
* **Anemometer**: Pulse-counting optoelectronic sensor mounted externally for measuring localized wind dynamics.
* **Display Output**: LCD 1602 driven via PCF8574 I2C Backpack (reducing pin count from 6 GPIOs down to 2 I2C lines: `SDA` / `SCL`).
* **Power Architecture**: Standalone 5V DC regulated lithium power bank housed within the plastic chassis for untethered field deployment.

## 📊 Comparative Analysis: DHT22 vs. BME280

| Performance Metric | DHT22 Probe | BME280 MEMS Sensor | Engineering Insight |
| :--- | :--- | :--- | :--- |
| **Communication Protocol** | Custom Single-Wire | I2C (`SDA` / `SCL`) | BME280 allows multi-device daisy-chaining on 2 wires. |
| **Response Latency** | High (~2.0 seconds) | Very Low (< 10 ms) | BME280 captures rapid ambient thermal spikes significantly faster. |
| **Sensor Footprint** | Large casing, exposed grill | Compact PCB surface-mount | BME280 requires protective enclosure vents due to sensitivity. |
| **Parameter Scope** | Temperature + Humidity | Temp + Humidity + Pressure | BME280 provides barometric trend analysis for weather forecasting. |

---

## 📈 Key Findings & Environmental Impact

* **Data Visualization & Graphing**: Constructed a comprehensive suite of **26 comparison graphs** analyzing cross-sensor response lag, thermal inertia, relative humidity gradients, and atmospheric pressure dynamics across field locations.
* **Enclosure Design & Mobility**: The custom plastic box protected internal circuitry from humidity while providing an ultra-portable form factor, allowing seamless 5-minute transit walks between University of York testing points without disturbing internal wiring connections.
* **University Campus Terrain Variance**:
  * **Lake Shore**: Exhibited the highest average relative humidity (~8% above open field) and thermal resistance due to the lake's heat capacity.
  * **Open Space Field**: Experienced the steepest thermal gradients during midday peak solar irradiance (12:00 - 14:00).
  * **Campus Plaza Buildings**: Recorded micro-climate wind acceleration (tunneling effect between structures) and elevated thermal radiation retained by building walls.
* **Sampling Protocol Insights**: Logged consistent 30-minute intervals from 09:00 to 17:00, factoring in 5-minute transit delays between adjacent site testing points without sensor calibration drift.
* **Sensor Inertia**: The DHT22 exhibited a ~12-second latency delay in capturing rapid temperature drops compared to the real-time responses of the BME280.
---

## 💻 Firmware Logic & Algorithm Design

The code is optimized for multi-site field mobility and non-blocking sensor acquisition:

```cpp
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BME280.h>
#include "DHT.h"

// Hardware Pin Definitions
#define DHTPIN 2
#define DHTTYPE DHT22

// Sensor Instances
DHT dht(DHTPIN, DHTTYPE);
Adafruit_BME280 bme; // I2C Communication

// Non-blocking Timing Control
unsigned long previousMillis = 0;
const long sampleInterval = 5000; // Real-time logging pulse

void setup() {
  Serial.begin(9600);
  
  // Initialize Sensors
  dht.begin();
  bool bme_status = bme.begin(0x76);
  
  if (!bme_status) {
    Serial.println(F("[ERROR] BME280 initialization failed. Check I2C wiring."));
    while (1); // Halt execution on critical hardware error
  }
  
  Serial.println(F("Timestamp(ms),Location,DHT_Temp(C),BME_Temp(C),DHT_Hum(%),BME_Hum(%),Pressure(hPa)"));
}

void loop() {
  unsigned long currentMillis = millis();

  // Execution loop running without delay() to prevent bus blocking
  if (currentMillis - previousMillis >= sampleInterval) {
    previousMillis = currentMillis;

    // Acquire DHT22 Data
    float dhtTemp = dht.readTemperature();
    float dhtHum  = dht.readHumidity();

    // Acquire BME280 Data via I2C
    float bmeTemp = bme.readTemperature();
    float bmeHum  = bme.readHumidity();
    float bmePres = bme.readPressure() / 100.0F; // Convert Pa to hPa

    // Sanity Check & Data Logging
    if (!isnan(dhtTemp) && !isnan(dhtHum)) {
      Serial.print(currentMillis); Serial.print(",");
      Serial.print(dhtTemp);       Serial.print(",");
      Serial.print(bmeTemp);       Serial.print(",");
      Serial.print(dhtHum);        Serial.print(",");
      Serial.print(bmeHum);        Serial.print(",");
      Serial.println(bmePres);
    }
  }
}


