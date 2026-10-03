# Laboratory Activity 4: Analog Input, PWM, and DAC

This repository contains the sketches, measurements, and written results for Laboratory Activity 4. The activity covers reading an analog input with the ADC, producing a PWM output, and producing a true analog voltage with the DAC on an ESP32.

## Equipment

1. ESP32 DevKit V1 (original ESP32, ESP32 WROOM 32)
2. Arduino IDE with the ESP32 board package version 3.x
3. 10 kOhm potentiometer
4. One LED and one 330 Ohm resistor
5. Digital multimeter set to DC volts
6. Breadboard and jumper wires

An oscilloscope was not available for this activity.

## Wiring Used

| Part | Connection |
|---|---|
| Potentiometer outer terminals | 3V3 and GND |
| Potentiometer center wiper | GPIO4 |
| LED anode | GPIO19 through the 330 Ohm resistor |
| LED cathode | GND |
| Multimeter red probe (DAC test) | GPIO25 |
| Multimeter black probe (DAC test) | GND |

### Change from the lab guide

The lab guide places the potentiometer wiper on GPIO34. The student moved it to GPIO4 because 3V3, GND, and GPIO4 are on the same side of the DevKit V1, which made the single row breadboard easier to use. GPIO4 is an ADC2 pin, and ADC2 readings only become unreliable when Wi-Fi is on. Wi-Fi was not used, so the readings were not affected. The same GPIO4 setup was used for Task 1 and Task 2 so the results can be compared.

For Task 3, the potentiometer and the LED were removed. Only the multimeter was connected to GPIO25 and GND, and the board was powered through USB.

## Task 1: Potentiometer Readings (Example 3)

The sketch reads the potentiometer at 12 bit resolution with 11 dB attenuation and prints the raw code and the millivolt reading. The knob was turned slowly to five positions and held still before each reading was recorded.

```cpp
#include <Arduino.h>
const uint8_t POT_PIN = 4;

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);
  analogSetPinAttenuation(POT_PIN, ADC_11db);
}

void loop() {
  const int raw = analogRead(POT_PIN);
  const uint32_t millivolts = analogReadMilliVolts(POT_PIN);
  Serial.print("Raw:"); Serial.print(raw);
  Serial.print("\tMillivolts:"); Serial.println(millivolts);
  delay(100);
}
```

| Position | Raw | Millivolts |
|---|---|---|
| Min | 0 | 128 |
| 1/4 | 1023 | 973 |
| 1/2 | 2047 | 1800 |
| 3/4 | 3071 | 2622 |
| Max | 4095 | 3130 |

The raw values are very close to the ideal values of 0, 1024, 2048, 3072, and 4095. The millivolt values do not follow a perfect straight line. The lowest position reads 128 mV instead of 0 mV, and the highest position reads 3130 mV instead of 3300 mV. The raw code and the millivolt value come from two separate conversions, so they do not have to match exactly.

## Task 2: Predicted and Observed PWM Duty (Example 4)

The sketch scales the raw reading to a duty value from 0 to 255 using integer math, then sends it to the LED on GPIO19 at 5 kHz with 8 bit resolution. The student added two Serial print lines so the raw value and the duty could be recorded.

The prediction used this formula, rounded down because the sketch uses integer math:

`duty = raw x 255 / 4095`

```cpp
#include <Arduino.h>
const uint8_t POT_PIN = 4;
const uint8_t PWM_LED_PIN = 19;
bool pwmReady = false;

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);
  analogSetPinAttenuation(POT_PIN, ADC_11db);
  pinMode(PWM_LED_PIN, OUTPUT);
  digitalWrite(PWM_LED_PIN, LOW);
  pwmReady = ledcAttach(PWM_LED_PIN, 5000, 8);
  if (pwmReady) {
    ledcWrite(PWM_LED_PIN, 0);
  } else {
    Serial.println("PWM setup failed. Check board/core and pin.");
  }
}

void loop() {
  if (!pwmReady) {
    return;
  }
  const int raw = analogRead(POT_PIN);
  const int duty = constrain(map(raw, 0, 4095, 0, 255), 0L, 255L);
  ledcWrite(PWM_LED_PIN, duty);
  Serial.print("Raw:"); Serial.print(raw);
  Serial.print("\tDuty:"); Serial.println(duty);
  delay(20);
}
```

| Position | Raw used for prediction (from Task 1) | Predicted duty | Predicted percent | Observed duty | LED brightness |
|---|---|---|---|---|---|
| Min | 0 | 0 | 0% | 0 | (add observation) |
| 1/4 | 1023 | 63 | 24.7% | 63 | (add observation) |
| 1/2 | 2047 | 127 | 49.8% | 127 | (add observation) |
| 3/4 | 3071 | 191 | 74.9% | 191 | (add observation) |
| Max | 4095 | 255 | 100% | 255 | (add observation) |

The observed duty matched the predicted duty at all five positions. This confirms that the sketch scales the raw reading as predicted. The brightness of an LED does not change in direct proportion to the duty value, so the brightness at the middle positions should not be expected to look like exactly 25 percent or 50 percent.

## Task 3: DAC Voltage at GPIO25 (Example 5)

The sketch writes five DAC values to GPIO25 and holds each one for two seconds before repeating. The multimeter was set to DC volts, with the red probe on GPIO25 and the black probe on GND. Nothing else was connected to GPIO25. The measured value was read near the end of each two second hold.

The predicted voltage was calculated as `value / 255 x 3.3 V`.

```cpp
#include <Arduino.h>

const int DAC_PIN = 25;

void setup() {
  dacWrite(DAC_PIN, 0);
}

void loop() {
  dacWrite(DAC_PIN, 0);
  delay(2000);

  dacWrite(DAC_PIN, 64);
  delay(2000);

  dacWrite(DAC_PIN, 128);
  delay(2000);

  dacWrite(DAC_PIN, 192);
  delay(2000);

  dacWrite(DAC_PIN, 255);
  delay(2000);
}
```

DAC voltage measured at GPIO25:

| dacWrite value | Predicted voltage (V) | Measured voltage (V) | Difference from prediction |
|---|---|---|---|
| 0 | 0.00 | 0 | none |
| 64 | 0.83 | 0.876 | 0.05 V higher |
| 128 | 1.66 | 1.652 | 0.01 V lower |
| 192 | 2.48 | 2.409 | 0.08 V lower |
| 255 | 3.30 | 3.166 | 0.13 V lower |

The voltage increased with each step. The reading at 128 was about half of the reading at 255 (1.652 V compared with 3.166 V). The measured values were close to the predictions near the middle of the range, but the highest level was about 0.13 V under the expected 3.3 V.

All values in this table are DAC measurements taken at GPIO25. No PWM measurement was taken with the multimeter, and none is reported as a DAC measurement.

## Task 4: Oscilloscope Comparison of GPIO19 and GPIO25

Oscilloscope not available. No waveform was captured for GPIO19 or GPIO25, so no waveform result is reported here.

Based on the lab reference, the expected difference is that GPIO19 under PWM would switch quickly between LOW and HIGH at 5 kHz, with a period of 200 microseconds. GPIO25 under the DAC would show a steady voltage that changes in steps every two seconds. This is only the expected behavior and was not observed.

## Task 5: Explanation

### Why PWM is not the same signal as DAC output

PWM only switches a pin between LOW (about 0 V) and HIGH (about 3.3 V). The duty cycle only changes how much of each cycle the pin stays HIGH. At 5 kHz, one cycle is 200 microseconds, so a duty of about 50 percent means roughly 100 microseconds HIGH and 100 microseconds LOW. The pin never holds a steady voltage in between.

The DAC works differently. It outputs a steady voltage at the level selected by the value from 0 to 255. The multimeter readings for PWM and for the DAC may look similar, but the signals are different. An LED reacts to the average power from the PWM pulses, while the DAC gives an actual voltage level. For this reason, the DAC can be used as a signal output, and PWM readings must not be reported as DAC measurements.

### Why an ADC endpoint may saturate

On the original ESP32 with 11 dB attenuation, the documented measurable range is about 150 mV to 3100 mV. A potentiometer connected to 3V3 can go above that range near its upper end, so the ADC reaches its maximum code before the real voltage stops rising. In the Task 1 results, the Max position gave a raw code of 4095 but only 3130 mV, and the Min position gave 128 mV instead of 0 mV. Because of this, a raw reading of 4095 should not be treated as an exact measurement of 3.3 V.

## Comparison of Predicted and Observed Results

1. **Potentiometer readings:** The raw codes were very close to the ideal values. The millivolt readings were not evenly spaced, and they flattened near both ends because of the limited ADC range.
2. **PWM duty:** The predicted and observed duty values were the same at all five positions (0, 63, 127, 191, 255). The sketch scales the raw reading the way the formula predicts.
3. **DAC voltage:** The measured voltages followed the expected order and were close to the predictions. The largest difference was 0.13 V at the highest level, which shows that the ESP32 DAC is not perfectly accurate.
4. **PWM and DAC:** PWM switches between two logic levels, while the DAC holds a steady voltage. The two should not be treated as the same signal.

## Repository Contents

| Folder | Description |
|---|---|
| Example3 | Reads the potentiometer and prints raw and millivolt values |
| Example4 | Controls LED brightness with PWM using the potentiometer |
| Example5 | Steps the DAC output on GPIO25 through five levels |
