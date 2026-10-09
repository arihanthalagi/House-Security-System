# Task 5 – House Security System Using ESP32-CAM, PIR Sensor, Buzzer, and Blynk

## 1. Aim

To develop a smart house security system using an ESP32-CAM, PIR motion sensor, and buzzer to detect motion, capture images of potential intruders, activate an alarm, and provide remote monitoring through the Blynk IoT application.

## 2. Introduction

Home security is important for protecting people and property from unauthorized access. Traditional security systems may require continuous monitoring or expensive surveillance equipment. An IoT-based house security system can provide a cost-effective solution by combining motion detection, image capture, audible alarms, and remote notifications.

In this project, a PIR (Passive Infrared) sensor detects motion by sensing changes in infrared radiation. When motion is detected, the ESP32-CAM activates the buzzer and captures an image of the area. The system can also send a notification to the Blynk application over Wi-Fi, allowing the user to monitor potential security events remotely.

## 3. Components Required

1. ESP32-CAM module
2. PIR motion sensor (HC-SR501 or equivalent)
3. Active buzzer compatible with the selected driver circuit
4. USB-to-serial programmer or compatible ESP32-CAM programmer
5. Jumper wires
6. Stable power supply
7. Wi-Fi connection
8. Smartphone with the Blynk application

## 4. Circuit Connections

### A. PIR Sensor to ESP32-CAM

| PIR Sensor Pin | ESP32-CAM Connection |
|---|---|
| VCC | 5 V, if supported by the PIR module |
| GND | GND |
| OUT | GPIO 13 |

GPIO 13 is used as the motion detection input. Ensure that the PIR output voltage does not exceed the ESP32's 3.3 V GPIO limit.

### B. Buzzer to ESP32-CAM

| Buzzer Connection | ESP32-CAM Connection |
|---|---|
| Control input | GPIO 14 through a suitable driver, if required |
| GND | GND |

A small, low-current buzzer may be suitable for direct GPIO control if its electrical specifications permit it. For a higher-current buzzer, use a transistor or MOSFET driver and an appropriate power supply. Do not connect a high-current buzzer directly to an ESP32 GPIO pin.

**Important:** ESP32-CAM boards have limited available GPIO pins because some are used by the camera, flash, or boot functions. GPIO 13 and GPIO 14 may be used with the camera configuration below, but verify that these pins are free on your specific board. Do not connect external circuitry in a way that interferes with booting or camera operation.

## 5. Working Principle

1. The PIR sensor continuously monitors the surrounding area for motion.
2. When movement is detected, the sensor sends a HIGH signal to the ESP32-CAM.
3. The ESP32-CAM activates the buzzer to provide an audible warning.
4. The camera captures an image of the monitored area.
5. The ESP32-CAM connects to Wi-Fi and communicates with the Blynk IoT platform.
6. The system sends a security notification to the user.
7. The user can access the camera stream or captured image through a configured interface.

The captured image and notification functions depend on the implemented software and network connection. A Blynk event notification alone does not automatically upload a photograph; image delivery or viewing must be configured separately.

## 6. ESP32-CAM Program Code

The following is a basic starting point for an AI-Thinker ESP32-CAM board. It detects motion, activates a buzzer output, and captures an image. The Blynk integration sends an event notification.

Install the **Blynk** library and select the appropriate ESP32 board in Arduino IDE. Ensure that the ESP32 board package includes the camera libraries.

```cpp
#define BLYNK_TEMPLATE_ID "YOUR_TEMPLATE_ID"
#define BLYNK_TEMPLATE_NAME "House Security"
#define BLYNK_AUTH_TOKEN "YOUR_AUTH_TOKEN"

#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include "esp_camera.h"

// Wi-Fi credentials
char ssid[] = "YOUR_WIFI_NAME";
char pass[] = "YOUR_WIFI_PASSWORD";

// External components
#define PIR_PIN 13
#define BUZZER_PIN 14

// AI-Thinker ESP32-CAM camera pin configuration
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

BlynkTimer timer;

bool previousMotion = false;
unsigned long lastAlert = 0;

void checkMotion() {
  bool motion = digitalRead(PIR_PIN) == HIGH;

  if (motion && !previousMotion) {
    Serial.println("Motion detected!");

    digitalWrite(BUZZER_PIN, HIGH);

    camera_fb_t *fb = esp_camera_fb_get();

    if (fb != nullptr) {
      Serial.printf("Image captured: %u bytes\n", fb->len);
      // The image is held in memory here.
      // Upload it to a server to view it remotely.
      esp_camera_fb_return(fb);
    } else {
      Serial.println("Camera capture failed");
    }

    if (Blynk.connected()) {
      Blynk.logEvent("motion_detected",
                     "Motion detected by the house security system");
    }

    lastAlert = millis();
  }

  if (!motion && previousMotion) {
    digitalWrite(BUZZER_PIN, LOW);
    Serial.println("Motion ended");
  }

  // Safety timeout for the buzzer
  if (digitalRead(BUZZER_PIN) == HIGH &&
      millis() - lastAlert > 5000) {
    digitalWrite(BUZZER_PIN, LOW);
  }

  previousMotion = motion;
}

void setup() {
  Serial.begin(115200);

  pinMode(PIR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sccb_sda = SIOD_GPIO_NUM;
  config.pin_sccb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  if (psramFound()) {
    config.frame_size = FRAMESIZE_VGA;
    config.jpeg_quality = 10;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_QVGA;
    config.jpeg_quality = 12;
    config.fb_count = 1;
  }

  if (esp_camera_init(&config) != ESP_OK) {
    Serial.println("Camera initialization failed");
    while (true) {
      delay(1000);
    }
  }

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  timer.setInterval(200L, checkMotion);

  Serial.println("House Security System Ready");
}

void loop() {
  Blynk.run();
  timer.run();
}
```

**Important limitations of this starter code:**

- The image is captured but not uploaded to Blynk or another server. The code releases the image buffer after capture.
- `Blynk.logEvent()` requires a matching event named `motion_detected` configured in the Blynk template.
- The buzzer timeout is included as a safety measure.
- If the PIR sensor remains HIGH for an extended period, the buzzer is switched off after five seconds, but a new alert is generated only after motion returns LOW and then HIGH again.
- The exact camera configuration depends on your ESP32-CAM board model and installed ESP32 Arduino core version.

## 7. Blynk IoT Application Setup

### Step 1: Create a Blynk account

Open https://blynk.cloud and sign in or create an account.

### Step 2: Create a template

Create a new template named **House Security System**. Select ESP32 as the hardware and Wi-Fi as the connection type.

### Step 3: Create a device

Create a device using the template and copy the Template ID and Auth Token into the Arduino program.

### Step 4: Configure the event

Open the template's Events section and create an event with the event code:

`motion_detected`

Enable push notifications for the event if supported by your account and application settings.

### Step 5: Configure the dashboard

Add suitable widgets to display the system status. A notification event can alert the user when motion is detected.

To view live camera footage or photographs remotely, configure a camera stream endpoint or an image-upload service. The camera's JPEG buffer must be transmitted to that service, and the Blynk dashboard must be configured to display the resulting stream or image URL.

## 8. Advantages

1. Detects motion automatically.
2. Captures images when motion occurs.
3. Provides an audible warning through a buzzer.
4. Enables remote security notifications through Blynk.
5. Offers a low-cost prototype for smart home monitoring.

## 9. Applications

- Home and apartment security
- Office and laboratory monitoring
- Restricted-area access monitoring
- Small shops and storage rooms
- IoT-based surveillance prototypes

