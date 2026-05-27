# Context-Aware Confidence Threshold (CACT) Pest Detection Robot

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Python 3.8+](https://img.shields.io/badge/Python-3.8+-blue.svg)](https://www.python.org/)
[![ROS Melodic](https://img.shields.io/badge/ROS-Melodic-orange.svg)](http://wiki.ros.org/melodic)
[![TensorFlow Lite](https://img.shields.io/badge/TensorFlow-Lite-FF6F00.svg)](https://www.tensorflow.org/lite)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.XXXXXXX.svg)](https://doi.org/10.5281/zenodo.XXXXXXX)

> **Paper:** "Context-Aware IoT–CNN Fusion for False-Positive Reduction in Edge-Based Agricultural Pest Detection: A Tropical Smallholder Robot System"
> Abubakar Rabiu Maihadisi — [Journal Name, 2026] — [DOI: PAPER DOI]

---

## Overview

This repository contains all code, model weights, and configuration files for a low-cost agricultural robot that fuses real-time IoT environmental sensor data with a MobileNetV2 edge-CNN to reduce false-positive pest detections under variable tropical field lighting.

The core contribution is the **Context-Aware Confidence Threshold (CACT)** algorithm: a computationally negligible late-fusion method that dynamically raises or lowers the CNN's detection confidence threshold based on whether current temperature, humidity, and illuminance fall within the known biological favorability window for the target pest species. No model retraining is required.

**Key results (on artificial-decoy proxy dataset):**
- F1-score: 0.862 → **0.917** (+6.4%)
- False-positive rate: 14.1% → **7.2%** (−49% relative)
- Field sprayed area reduced by **62%** with equivalent pest targeting accuracy
- Total hardware cost: ~US$290 (locally sourced, Nigeria)

---

## Repository Structure

```
cact-pest-detection-robot/
│
├── cact/                          # Core CACT fusion algorithm
│   ├── cact_filter.py             # Main CACT class and threshold logic
│   ├── favorability_windows.py    # Environmental favorability definitions per pest
│   └── threshold_sweep.py        # Validation-set grid search for τ_fav / τ_unfav
│
├── vision/                        # CNN training and inference
│   ├── train_mobilenetv2.py       # MobileNetV2 fine-tuning (TensorFlow 2.11)
│   ├── convert_to_tflite.py       # TFLite + Edge TPU quantisation pipeline
│   ├── inference_coral.py         # On-device inference with Coral Edge TPU
│   └── augmentation.py            # Data augmentation pipeline
│
├── ros_nodes/                     # ROS Melodic nodes
│   ├── vision_node.py             # Camera capture → CNN inference → confidence pub
│   ├── iot_node.py                # ESP32 MQTT subscriber → sensor vector pub
│   ├── cact_fusion_node.py        # Subscribes to both → publishes spray command
│   └── navigation_node.py         # A* path planning + GPS waypoints + obstacle avoidance
│
├── esp32/                         # ESP32 firmware
│   └── sensor_publisher/
│       └── sensor_publisher.ino   # DHT22 + BH1750 → MQTT at 0.5 Hz
│
├── configs/                       # Configuration files
│   ├── robot_params.yaml          # Hardware parameters (spray rate, cone angle, speed)
│   ├── cact_params.yaml           # CACT thresholds and favorability windows
│   └── ros_launch/
│       └── full_system.launch     # Full system launch file
│
├── evaluation/                    # Offline evaluation scripts
│   ├── run_offline_eval.py        # Reproduces Table 3–5 results from paper
│   ├── per_class_metrics.py       # Generates Table 5 (per-class precision/recall/F1)
│   ├── statistical_tests.py       # Paired t-test, Wilcoxon, Cohen's d, Bonferroni
│   └── threshold_sensitivity.py   # Generates Supplementary Figure S3
│
├── dataset/                       # Dataset access (images hosted separately)
│   └── README_dataset.md          # Dataset description and download instructions
│
├── models/                        # Pre-trained model weights
│   ├── mobilenetv2_tomato_pest.tflite      # Standard TFLite model
│   └── mobilenetv2_tomato_pest_edgetpu.tflite  # Compiled for Coral Edge TPU
│
├── requirements.txt               # Python dependencies
├── requirements_training.txt      # Additional dependencies for training only
├── LICENSE                        # MIT License
└── README.md                      # This file
```

---

## Hardware Requirements

| Component | Specification |
|---|---|
| Single-board computer | Raspberry Pi 4 (4 GB RAM) |
| AI accelerator | Google Coral USB Accelerator |
| Camera | Raspberry Pi Camera Module v2 (8 MP) |
| Microcontroller | ESP32-WROOM-32D |
| Temperature/Humidity | DHT22 (±0.5°C, ±2% RH) |
| Illuminance | BH1750 (I²C, 1–65,535 lux) |
| GPS | u-blox NEO-6M |
| Motor driver | L298N dual H-bridge (×2) |

Full hardware BOM with Nigerian market prices is provided in the paper (Table 1). Total cost ≈ ₦230,000 (~US$290).

---

## Software Requirements

### On the Raspberry Pi (Robot)

- Ubuntu 18.04 (64-bit, Raspberry Pi)
- ROS Melodic
- Python 3.8+
- PyCoral runtime: https://coral.ai/docs/accelerator/get-started/

```bash
# Install Python dependencies
pip3 install -r requirements.txt

# Install ROS Melodic
# Follow: http://wiki.ros.org/melodic/Installation/Ubuntu
```

### For Training Only (Desktop/GPU workstation)

- Python 3.8+
- TensorFlow 2.11
- CUDA 11.x + cuDNN 8.x (for GPU training)
- Edge TPU Compiler v16: https://coral.ai/docs/edgetpu/compiler/

```bash
pip3 install -r requirements_training.txt
```

### On the ESP32

- Arduino IDE 1.8+ or PlatformIO
- Libraries: `DHT sensor library`, `BH1750`, `PubSubClient` (MQTT), `WiFi`

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/[YOUR-USERNAME]/cact-pest-detection-robot.git
cd cact-pest-detection-robot
pip3 install -r requirements.txt
```

### 2. Flash the ESP32

Open `esp32/sensor_publisher/sensor_publisher.ino` in the Arduino IDE. Update the WiFi credentials and MQTT broker IP, then flash to the ESP32.

### 3. Configure the system

Edit `configs/cact_params.yaml` to set your CACT thresholds and favorability windows:

```yaml
# CACT thresholds (validated on our dataset; tune for your conditions)
tau_favorable: 0.55      # Lower threshold when environment favors pests
tau_unfavorable: 0.85    # Higher threshold when environment is unfavorable

# Environmental favorability windows
favorability:
  whitefly:
    temp_min: 20
    temp_max: 32
    rh_min: 55
    rh_max: 85
    lux_min: 300
  aphid:
    temp_min: 15
    temp_max: 30
    rh_min: 50
    rh_max: 90
    lux_min: 200
  thrips:
    temp_min: 18
    temp_max: 35
    rh_min: 60
    rh_max: 90
    lux_min: 400
```

### 4. Launch the full system

```bash
# Source ROS
source /opt/ros/melodic/setup.bash
source ~/catkin_ws/devel/setup.bash

# Launch all nodes
roslaunch configs/ros_launch/full_system.launch
```

This starts the vision node, IoT node, CACT fusion node, and navigation node simultaneously.

---

## Running Inference Only (No Robot Hardware)

You can run the CACT filter offline on any image + environmental reading:

```python
from cact.cact_filter import CACTFilter
from vision.inference_coral import CoralInference

# Load model
model = CoralInference("models/mobilenetv2_tomato_pest_edgetpu.tflite")

# Initialise CACT
cact = CACTFilter(tau_fav=0.55, tau_unfav=0.85)

# Run inference
image_path = "path/to/your/image.jpg"
env_vector = {"temperature": 28.5, "humidity": 72.0, "lux": 850}

probs = model.predict(image_path)          # Returns softmax probabilities
label, confident = cact.decide(probs, env_vector)  # Returns class + bool

print(f"Detection: {label} | Confident: {confident}")
```

---

## Reproducing Paper Results

All offline evaluation results (Tables 3–6, Supplementary Figure S3) can be reproduced from the test set:

```bash
# Download dataset first (see dataset/README_dataset.md)

# Run full offline evaluation (reproduces Tables 3 and 4)
python evaluation/run_offline_eval.py \
    --model models/mobilenetv2_tomato_pest_edgetpu.tflite \
    --test-dir data/test/ \
    --env-log data/test/env_vectors.csv

# Per-class metrics (reproduces Table 5)
python evaluation/per_class_metrics.py --results-dir results/

# Statistical tests (paired t-test, Wilcoxon, Cohen's d, Bonferroni)
python evaluation/statistical_tests.py --results-dir results/

# Threshold sensitivity analysis (reproduces Supplementary Figure S3)
python evaluation/threshold_sensitivity.py \
    --val-dir data/val/ \
    --env-log data/val/env_vectors.csv \
    --output figures/supp_fig_s3.png
```

---

## Training Your Own Model

To fine-tune MobileNetV2 on your own pest dataset:

```bash
python vision/train_mobilenetv2.py \
    --data-dir data/train/ \
    --val-dir data/val/ \
    --epochs 60 \
    --batch-size 32 \
    --lr 1e-4 \
    --output models/my_model.h5
```

Convert to TFLite and compile for Coral:

```bash
# Convert to TFLite (full integer quantisation)
python vision/convert_to_tflite.py \
    --model models/my_model.h5 \
    --output models/my_model.tflite \
    --representative-data data/train/

# Compile for Edge TPU (requires Edge TPU Compiler installed)
edgetpu_compiler models/my_model.tflite -o models/
```

---

## Dataset

The tomato pest proxy dataset (3,000 images; whitefly, aphid, thrips, no_pest; morning and evening lighting; Chikun LGA, Kaduna State, Nigeria; October–December 2025) is hosted separately due to size.

**Download:** [INSERT DATASET DOI / DOWNLOAD LINK]

See `dataset/README_dataset.md` for the full dataset description, class distribution, environmental vector format, and labelling protocol.

---

## CACT Algorithm — How It Works

```
For each captured frame:

1. Run MobileNetV2 → softmax probabilities p = [p_whitefly, p_aphid, p_thrips, p_no_pest]
2. Read latest sensor values: T (°C), H (% RH), L (lux) from ESP32 via MQTT
3. Check favorability for each pest species:
      F_i = 1  if  T_min,i ≤ T ≤ T_max,i
               AND H_min,i ≤ H ≤ H_max,i
               AND L ≥ L_min,i
4. Set union flag:  F = max(F_whitefly, F_aphid, F_thrips)
5. Set dynamic threshold:
      τ_dyn = τ_fav   (0.55)  if F = 1
      τ_dyn = τ_unfav (0.85)  if F = 0
6. Detection decision:
      if max(p) ≥ τ_dyn  →  spray (predicted class)
      else                →  no spray
```

**Latency:** < 0.5 ms for steps 2–6. Total end-to-end latency dominated by ROS inter-node communication (~300 ms).

---

## Citation

If you use this code or dataset in your research, please cite:

```bibtex
@article{maihadisi2026cact,
  title     = {Context-Aware IoT--CNN Fusion for False-Positive Reduction in
               Edge-Based Agricultural Pest Detection: A Tropical Smallholder
               Robot System},
  author    = {Maihadisi, Abubakar Rabiu},
  journal   = {[JOURNAL NAME]},
  year      = {2026},
  doi       = {[PAPER DOI]},
  note      = {Code and data: \url{https://github.com/[YOUR-USERNAME]/cact-pest-detection-robot}}
}
```

---

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

## Acknowledgements

The author thanks Alh. Dahiru Maigwari for field access and the Department of Electrical Engineering, Bayero University Kano, for laboratory support.

---

## Contact

**Abubakar Rabiu Maihadisi**
Department of Electrical Engineering, Bayero University Kano, Nigeria
maihadisiabubakar@gmail.com

*Issues and pull requests are welcome.*
