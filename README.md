
# Borewell Rescue System using ESP32-CAM

## Project Overview

This project implements a **Borewell Rescue System** using **ESP32-CAM** for real-time video monitoring and robotic arm control to assist in rescuing trapped individuals (especially children) from borewells. The system operates **without cloud dependency**, making it suitable for remote areas with limited internet connectivity.

## Key Features

* **Live Video Streaming**: ESP32-CAM provides real-time video feed over local Wi-Fi
* **Robotic Arm Control**: Servo motor-based arm for rescue operations
* **Local Operation**: Works without cloud/IoT for reliability in low-connectivity areas
* **Web Interface**: Built-in control panel accessible via smartphone/laptop
* **Portable Design**: Battery-powered for field deployment

## Hardware Components

* ESP32-CAM module
* Servo motor (SG90/MG996R)
* DC gear motors (for movement, if implemented)
* L298N motor driver
* Power supply (Li-ion battery/power bank)
* High-power LED for illumination
* Robot chassis and mechanical components

## Software Requirements

* Arduino IDE (with ESP32 board support)
* Required libraries:

  * `esp_camera.h`
  * `WiFi.h`
  * `ESP32Servo.h`
  * `esp_http_server.h`

## Setup Instructions

### 1. Hardware Assembly

* Connect ESP32-CAM to servo motor and motor driver
* Mount camera and robotic arm on chassis
* Ensure proper power connections

### 2. Software Configuration

* Install required Arduino libraries
* Update Wi-Fi credentials in the code
* Upload firmware to ESP32-CAM

### 3. Operation

* Power on the system
* Connect to ESP32's Wi-Fi network
* Access web interface via browser at assigned IP
* Use controls to monitor video and operate robotic arm

## Project Structure

* **/src** – Main Arduino sketch and supporting files
* **/docs** – Project documentation and references
* **/hardware** – Schematics and mechanical designs

## Applications

* Emergency rescue operations in borewells
* Remote inspection of narrow shafts/pipes
* Educational demonstration of IoT/robotics

## Future Enhancements

* Integration with environmental sensors
* AI-based object detection
* Solar power support
* Automated rescue sequence

## Contributors

* Y. Sreenath (22691A04Q4)
* K. Ravali (22691A04K5)
* B. Sivaiah (22691A04P5)
  

**Guided by**: Dr. Smriti Baruah, M.Tech.
**Institution**: Madanapalle Institute of Technology & Science
