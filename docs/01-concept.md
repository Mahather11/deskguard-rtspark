# 1. Consumer Product Concept and Design

## 1.1 Product Overview

**Product Name:** DeskGuard

**Target User:** Students, office workers, and individuals who spend extended periods sitting at a desk.

**Problem Statement:** Prolonged sitting and poor posture (slouching) lead to musculoskeletal pain, reduced focus, and long-term health issues. Users often lose track of time and forget to take breaks, and may also work in inadequate lighting conditions that cause eye strain.

**Operating Environment:** Indoor, placed on a desk near the user.

**Main Product Goal:** To provide real-time, on-device feedback regarding the user's posture, movement (fidgeting/restlessness), and environmental conditions, prompting healthier desk habits without requiring an internet connection.

## 1.2 Alternative Ideas Considered

1. **Smart Storage Box Monitor:** Uses an IMU to detect if a box is opened or closed, and a light sensor to see if the box is exposed to light.
   - *Why rejected:* Too similar to the reference repository provided for this activity. It also offers limited continuous user feedback.
2. **Smart Plant Waterer:** Uses AHT10 (humidity) to determine if soil is dry and triggers a small pump.
   - *Why rejected:* Does not utilize the powerful 6-axis IMU, and the classification task (wet vs. dry) is too simple for a machine learning model.
3. **DeskGuard (Selected):** Utilizes the IMU for motion/posture classification and the light sensor for environmental context, providing a rich, continuous machine learning problem with immediate, visible feedback on the LCD.

## 1.3 Classification States

The machine learning model will run a sliding window (e.g., 2 seconds) over the IMU data to classify the user's state into three categories:

1. **`POSTURE_GOOD`:** The user is relatively still (low angular variance), and the pitch angle indicates an upright posture. The ambient light is sufficient (> 300 lux).
2. **`POSTURE_SLOUCH`:** The user's pitch angle exceeds a healthy threshold (e.g., leaning forward > 25°), indicating poor ergonomics.
3. **`SEDENTARY_ALERT`:** The IMU detects almost zero movement or variance for an extended period (e.g., > 45 minutes), regardless of the current posture. The user needs to stand up and stretch.

*(Note: If the AP3216C detects low light, the system will override posture classification and display an `ENV_DARK` warning, as poor lighting affects focus.)*

## 1.4 System Architecture & Block Diagram

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                               DESKGUARD SYSTEM                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                   │
│  │   SENSING    │    │  PROCESSING  │    │   INFERENCE  │                   │
│  ├──────────────┤    ├──────────────┤    ├──────────────┤                   │
│  │ ICM20608     │───▶│ Windowing &  │───▶│ Trained      │                   │
│  │ (I2C2 - IMU) │    │ Feature      │    │ Model (TFLM) │                   │
│  │              │    │ Extraction   │    │ on STM32F407 │                   │
│  ├──────────────┤    │              │    │              │                   │
│  │ AP3216C      │───▶│              │    │              │                   │
│  │ (I2C2 - Light│    │              │    │              │                   │
│  └──────────────┘    └──────────────┘    └──────┬───────┘                   │
│                                                  │                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────▼───────┐                   │
│  │    INPUT     │    │   FEEDBACK   │    │   LOGGING    │                   │
│  ├──────────────┤    ├──────────────┤    ├──────────────┤                   │
│  │ Onboard      │───▶│ ST7789 LCD   │◀───│ RTC /        │                   │
│  │ Buttons      │    │ Buzzer       │    │ SPI Flash    │                   │
│  └──────────────┘    └──────────────┘    └──────────────┘                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1.5 Component-to-Function Matrix

| Component / Module | Function Category | Primary Use in DeskGuard |
| --- | --- | --- |
| **ICM20608 (IMU)** | Primary Sensing | Collects accelerometer and gyroscope data to determine posture pitch and movement variance. |
| **AP3216C (Light/Prox)** | Context Sensing | Measures ambient light to determine if the environment is suitable for studying. |
| **AHT20 (Temp/Humidity)** | Context Sensing | Logs ambient room conditions (Diagnostic data). |
| **ST7789 LCD** | User Feedback | Displays the current state (e.g., "Good Posture", "Sit Up!", "Time to Move!"). |
| **Buzzer & LEDs** | User Feedback | Provides audible and visual alerts when poor posture or sedentary states are detected. |
| **Push Buttons** | User Interaction | Allows the user to acknowledge alerts, reset the sedentary timer, or enter diagnostic mode. |
| **SPI Flash / SD Card** | Logging | Stores timestamps of when bad posture or sedentary events occurred. |
| **RTC** | Logging | Provides accurate timestamps for the logging system. |

## 1.6 Acceptance Criteria

**Functional Behavior:**

- The system must classify the three states with a held-out test accuracy (macro-F1) of **≥ 0.80**.
- Inference must execute completely on-device (STM32F407) with no cloud dependency.
- The LCD UI must update within **500 ms** of a state change.

**Error Types That Matter:**

- **False Negatives (Missing a slouch):** Causes health issues. The model must be tuned to prioritize recall for the `POSTURE_SLOUCH` and `SEDENTARY` classes.
- **False Positives (Alerting when the user is fine):** Causes annoyance; the user might turn off the device.

**Measurement Plan:**

- Train the model on data collected from 3 different individuals (varying heights/desk setups).
- Collect at least 30 training windows and 10 test windows per class.
- Record inference latency over 50 executions to ensure it meets the 500 ms threshold.

**Risks & Mitigations:**

- **Privacy:** The device does not use a camera or microphone. Only motion and light data are collected. No personal data leaves the board.
- **Safety:** This is an experimental prototype for behavioral assistance, not a certified medical or ergonomic device.
- **Accessibility:** Visual (LCD), auditory (buzzer), and physical (buttons) feedback ensure the device is usable in various environments.
