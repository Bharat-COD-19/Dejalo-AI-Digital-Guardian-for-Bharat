# Design Document

## Overview

Dialo is a wearable-first mental health monitoring system designed for rural India that combines edge computing, multimodal AI, and behavioral digital twins to detect stress escalation in real-time and enable preventive interventions. The system operates across five architectural layers: Wearable Layer (edge sensing and TinyML), Mobile Application Layer (offline-first patient/caregiver interface), Cloud AI Layer (multimodal inference and decision-making), Intervention Layer (alerts and guidance), and Continuous Learning Loop (model personalization).

The design prioritizes offline-first operation, sub-2-second latency, privacy-preserving architecture, and cultural appropriateness for rural Indian contexts. The system learns individual behavioral patterns through a Digital Twin that models trigger responses and predicts crisis escalation, enabling personalized interventions delivered to patients, caregivers, and healthcare providers.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          DIALO SYSTEM ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│  WEARABLE LAYER  │
│                  │
│  ┌────────────┐  │
│  │ MEMS Mic   │  │
│  │ HR Sensor  │  │         BLE
│  │ HRV        │──┼────────────────┐
│  │ Accel 3-ax │  │                │
│  │ TinyML     │  │                │
│  │ (50ms inf) │  │                ▼
│  └────────────┘  │      ┌─────────────────────┐
│                  │      │  MOBILE APP LAYER   │
│  Battery: 24hr   │      │  (Flutter)          │
│  BLE 4.0+        │      │                     │
│  AES-128         │      │  ┌───────────────┐  │
└──────────────────┘      │  │ BLE Receiver  │  │
                          │  │ Local ML      │  │
                          │  │ Offline Store │  │
                          │  │ Alert UI      │  │
                          │  │ Consent Mgmt  │  │
                          │  └───────────────┘  │
                          │                     │
                          │  Storage: 7 days    │
                          │  Sync: WiFi/2G      │
                          └──────────┬──────────┘
                                     │
                                     │ HTTPS/TLS 1.3
                                     │ (when online)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        CLOUD AI LAYER (AWS)                              │
│                                                                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │  AWS IoT     │───▶│   Lambda     │───▶│   FastAPI Service        │  │
│  │  Core        │    │  Ingestion   │    │   (ECS Fargate)          │  │
│  └──────────────┘    └──────────────┘    │                          │  │
│                                           │  ┌────────────────────┐  │  │
│  ┌──────────────┐                         │  │ Multimodal ML      │  │  │
│  │  DynamoDB    │◀────────────────────────┼──│ - Voice Encoder    │  │  │
│  │              │                         │  │ - HRV Encoder      │  │  │
│  │ - Users      │                         │  │ - Motion Encoder   │  │  │
│  │ - Events     │                         │  │ - Fusion Layer     │  │  │
│  │ - TriggerMap │                         │  │ - LSTM/Transform   │  │  │
│  │ - DigitalTwin│                         │  └────────────────────┘  │  │
│  │ - Consent    │                         │                          │  │
│  └──────────────┘                         │  ┌────────────────────┐  │  │
│                                           │  │ Digital Twin Eng   │  │  │
│  ┌──────────────┐                         │  │ - State Machine    │  │  │
│  │  S3 Storage  │◀────────────────────────┼──│ - Trigger Graph    │  │  │
│  │              │                         │  │ - Prediction       │  │  │
│  │ - Raw Data   │                         │  │ - RL Update        │  │  │
│  │ - Models     │                         │  └────────────────────┘  │  │
│  │ - Reports    │                         │                          │  │
│  └──────────────┘                         │  ┌────────────────────┐  │  │
│                                           │  │ Decision Engine    │  │  │
│                                           │  │ - Rule Evaluation  │  │  │
│                                           │  │ - Alert Routing    │  │  │
│                                           │  │ - Explanation Gen  │  │  │
│                                           │  └────────────────────┘  │  │
│                                           └──────────────────────────┘  │
└──────────────────────────────────────────┬───────────────────────────────┘
                                           │
                                           │ Firebase Cloud Messaging
                                           │ SMS Gateway (fallback)
                                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       INTERVENTION LAYER                                 │
│                                                                          │
│  ┌──────────────────────┐         ┌──────────────────────┐             │
│  │  Wearable Feedback   │         │  Caregiver App       │             │
│  │  - Vibration alert   │         │  - Push notifications│             │
│  │  - Breathing guide   │         │  - Stress trends     │             │
│  │  - Haptic patterns   │         │  - Trigger explain   │             │
│  └──────────────────────┘         │  - Guidance (local)  │             │
│                                   │  - Multi-language    │             │
│  ┌──────────────────────┐         └──────────────────────┘             │
│  │  Patient App         │                                               │
│  │  - Alert display     │         ┌──────────────────────┐             │
│  │  - Breathing ex      │         │  Doctor Dashboard    │             │
│  │  - Feedback input    │         │  (Web)               │             │
│  │  - Emergency button  │         │  - Stress analytics  │             │
│  └──────────────────────┘         │  - Trigger heatmap   │             │
│                                   │  - Event timeline    │             │
│                                   │  - PDF reports       │             │
│                                   └──────────────────────┘             │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    CONTINUOUS LEARNING LOOP                              │
│                                                                          │
│  User Feedback ──▶ Model Retraining ──▶ Digital Twin Update ──▶        │
│       ▲                                                          │       │
│       │                                                          ▼       │
│  Prediction Accuracy ◀── A/B Testing ◀── Model Deployment ◀─────┘       │
│                                                                          │
│  Frequency: Daily batch updates, Weekly model releases                  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Component Architecture

### 1. Wearable Device Architecture

**Hardware Components:**
- **Microcontroller**: ARM Cortex-M4F (100 MHz, 256 KB RAM, 1 MB Flash)
- **MEMS Microphone**: Omnidirectional, 16 kHz sampling, -26 dBFS sensitivity
- **Heart Rate Sensor**: PPG-based, 1 Hz sampling, ±2 bpm accuracy
- **3-Axis Accelerometer**: MEMS, 50 Hz sampling, ±2g range
- **BLE Module**: nRF52832, BLE 4.2, AES-128 hardware encryption
- **Battery**: 400 mAh LiPo, USB-C charging, 24+ hour runtime
- **Enclosure**: IP54-rated, silicone wristband, 40g weight

**Firmware Modules:**

```
┌─────────────────────────────────────────┐
│      Wearable Firmware Architecture     │
├─────────────────────────────────────────┤
│                                         │
│  ┌───────────────────────────────────┐  │
│  │   Sensor Manager (RTOS Task)     │  │
│  │   - HR: 1 Hz polling             │  │
│  │   - Accel: 50 Hz DMA buffer      │  │
│  │   - Mic: 16 kHz I2S stream       │  │
│  └───────────────┬───────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │   Feature Extraction Module      │  │
│  │   - MFCC (13 coeff, 25ms window) │  │
│  │   - HRV (SDNN, RMSSD, pNN50)     │  │
│  │   - Motion (variance, peaks)     │  │
│  └───────────────┬───────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │   TinyML Inference Engine        │  │
│  │   - Model: Quantized CNN (INT8)  │  │
│  │   - Size: 180 KB                 │  │
│  │   - Inference: 50ms              │  │
│  │   - Output: Stress score 0-100   │  │
│  └───────────────┬───────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │   BLE Communication Module       │  │
│  │   - MTU: 247 bytes               │  │
│  │   - Interval: 100ms              │  │
│  │   - Encryption: AES-128          │  │
│  │   - Buffer: 4 hours (2 MB)       │  │
│  └───────────────┬───────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │   Power Management               │  │
│  │   - Active: 25 mA                │  │
│  │   - Sleep: 5 µA                  │  │
│  │   - Adaptive sampling            │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### 2. Mobile Application Architecture (Flutter)

**Architecture Pattern**: Clean Architecture with BLoC state management

```
┌─────────────────────────────────────────────────────────────┐
│              Mobile Application Architecture                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              PRESENTATION LAYER                     │   │
│  │                                                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │   │
│  │  │ Home     │  │ Alerts   │  │ Settings         │ │   │
│  │  │ Screen   │  │ Screen   │  │ Screen           │ │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────────────┘ │   │
│  │       │             │             │               │   │
│  │       └─────────────┴─────────────┘               │   │
│  │                     │                             │   │
│  │  ┌──────────────────▼──────────────────────────┐  │   │
│  │  │         BLoC (Business Logic)              │  │   │
│  │  │  - StressMonitoringBloc                    │  │   │
│  │  │  - AlertBloc                               │  │   │
│  │  │  - SyncBloc                                │  │   │
│  │  │  - ConsentBloc                             │  │   │
│  │  └──────────────────┬──────────────────────────┘  │   │
│  └───────────────────────┼──────────────────────────────┘   │
│                          │                                  │
│  ┌───────────────────────▼──────────────────────────────┐   │
│  │              DOMAIN LAYER                           │   │
│  │                                                     │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  Use Cases                                   │  │   │
│  │  │  - MonitorStressUseCase                      │  │   │
│  │  │  - ProcessAlertUseCase                       │  │   │
│  │  │  - SyncDataUseCase                           │  │   │
│  │  │  - ManageConsentUseCase                      │  │   │
│  │  └──────────────────┬───────────────────────────┘  │   │
│  │                     │                              │   │
│  │  ┌──────────────────▼───────────────────────────┐  │   │
│  │  │  Entities                                    │  │   │
│  │  │  - StressEvent, Alert, User, Consent         │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └─────────────────────┬────────────────────────────────┘   │
│                        │                                    │
│  ┌─────────────────────▼────────────────────────────────┐   │
│  │              DATA LAYER                             │   │
│  │                                                     │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │   │
│  │  │ BLE          │  │ Local DB     │  │ Cloud    │ │   │
│  │  │ Repository   │  │ Repository   │  │ API Repo │ │   │
│  │  └──────┬───────┘  └──────┬───────┘  └────┬─────┘ │   │
│  │         │                 │                │       │   │
│  │  ┌──────▼───────┐  ┌──────▼───────┐  ┌────▼─────┐ │   │
│  │  │ flutter_blue │  │ Hive/SQLite  │  │ Dio/HTTP │ │   │
│  │  │ (BLE)        │  │ (Offline)    │  │ (Sync)   │ │   │
│  │  └──────────────┘  └──────────────┘  └──────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         CROSS-CUTTING CONCERNS                      │   │
│  │  - TFLite (Local ML inference)                      │   │
│  │  - Firebase Cloud Messaging                         │   │
│  │  - Localization (i18n)                              │   │
│  │  - Logging & Analytics                              │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Key Modules:**

1. **BLE Manager**: Handles device discovery, pairing, data reception
2. **Local ML Engine**: TensorFlow Lite for offline stress scoring
3. **Offline Storage**: Hive database for 7-day data buffering
4. **Sync Manager**: Intelligent synchronization prioritizing critical alerts
5. **Notification Manager**: Local and push notification handling
6. **Consent Manager**: Granular permission control and audit logging

### 3. Cloud Microservices Architecture

**Deployment**: AWS ECS Fargate with Application Load Balancer

```
┌─────────────────────────────────────────────────────────────┐
│           Cloud Microservices Architecture                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         API Gateway (AWS API Gateway)               │   │
│  │         - Authentication (Cognito)                  │   │
│  │         - Rate limiting                             │   │
│  │         - Request validation                        │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│           ┌─────────────┼─────────────┐                     │
│           │             │             │                     │
│  ┌────────▼────────┐ ┌──▼──────────┐ ┌▼────────────────┐   │
│  │ Ingestion       │ │ Inference   │ │ Decision        │   │
│  │ Service         │ │ Service     │ │ Service         │   │
│  │ (FastAPI)       │ │ (FastAPI)   │ │ (FastAPI)       │   │
│  │                 │ │             │ │                 │   │
│  │ - Data valid    │ │ - ML model  │ │ - Rule engine   │   │
│  │ - Enrichment    │ │ - Batch inf │ │ - Alert routing │   │
│  │ - DynamoDB      │ │ - Feature   │ │ - Explanation   │   │
│  │   write         │ │   store     │ │   generation    │   │
│  └─────────────────┘ └─────────────┘ └─────────────────┘   │
│                                                             │
│  ┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐   │
│  │ Digital Twin    │ │ Report      │ │ User Management │   │
│  │ Service         │ │ Service     │ │ Service         │   │
│  │ (FastAPI)       │ │ (FastAPI)   │ │ (FastAPI)       │   │
│  │                 │ │             │ │                 │   │
│  │ - State update  │ │ - Analytics │ │ - Auth          │   │
│  │ - Prediction    │ │ - PDF gen   │ │ - Consent       │   │
│  │ - RL training   │ │ - Export    │ │ - RBAC          │   │
│  └─────────────────┘ └─────────────┘ └─────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Shared Infrastructure                       │   │
│  │  - DynamoDB (data store)                            │   │
│  │  - S3 (object storage)                              │   │
│  │  - ElastiCache Redis (caching)                      │   │
│  │  - SQS (async processing)                           │   │
│  │  - CloudWatch (monitoring)                          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 4. ML Model Pipeline

**Training Infrastructure**: SageMaker for model training and deployment

```
┌─────────────────────────────────────────────────────────────┐
│              ML Model Pipeline Architecture                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Data Collection & Preprocessing             │   │
│  │  S3 Raw Data ──▶ Glue ETL ──▶ Feature Store        │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Model Training (SageMaker)                  │   │
│  │                                                     │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  Multimodal Stress Detection Model           │  │   │
│  │  │                                              │  │   │
│  │  │  Input: [MFCC(13), HRV(5), Motion(3)]       │  │   │
│  │  │         Shape: (batch, 21)                   │  │   │
│  │  │                                              │  │   │
│  │  │  Architecture:                               │  │   │
│  │  │  ┌────────────────────────────────────────┐  │  │   │
│  │  │  │ Voice Encoder (1D CNN)                 │  │  │   │
│  │  │  │ Input: MFCC(13) ──▶ Conv1D(32,3)      │  │  │   │
│  │  │  │                 ──▶ Conv1D(64,3)      │  │  │   │
│  │  │  │                 ──▶ GlobalAvgPool     │  │  │   │
│  │  │  │ Output: (64,)                          │  │  │   │
│  │  │  └────────────────────────────────────────┘  │  │   │
│  │  │                                              │  │   │
│  │  │  ┌────────────────────────────────────────┐  │  │   │
│  │  │  │ HRV Encoder (Dense)                    │  │  │   │
│  │  │  │ Input: HRV(5) ──▶ Dense(32, ReLU)     │  │  │   │
│  │  │  │               ──▶ Dense(64, ReLU)     │  │  │   │
│  │  │  │ Output: (64,)                          │  │  │   │
│  │  │  └────────────────────────────────────────┘  │  │   │
│  │  │                                              │  │   │
│  │  │  ┌────────────────────────────────────────┐  │  │   │
│  │  │  │ Motion Encoder (Dense)                 │  │  │   │
│  │  │  │ Input: Motion(3) ──▶ Dense(16, ReLU)  │  │  │   │
│  │  │  │                  ──▶ Dense(32, ReLU)  │  │  │   │
│  │  │  │ Output: (32,)                          │  │  │   │
│  │  │  └────────────────────────────────────────┘  │  │   │
│  │  │                                              │  │   │
│  │  │  ┌────────────────────────────────────────┐  │  │   │
│  │  │  │ Fusion Layer                           │  │  │   │
│  │  │  │ Concat(64+64+32) ──▶ Dense(128, ReLU) │  │  │   │
│  │  │  │                  ──▶ Dropout(0.3)      │  │  │   │
│  │  │  │                  ──▶ Dense(64, ReLU)   │  │  │   │
│  │  │  │ Output: (64,)                          │  │  │   │
│  │  │  └────────────────────────────────────────┘  │  │   │
│  │  │                                              │  │   │
│  │  │  ┌────────────────────────────────────────┐  │  │   │
│  │  │  │ Temporal Layer (LSTM)                  │  │  │   │
│  │  │  │ Input: (seq_len=10, 64)                │  │  │   │
│  │  │  │ LSTM(128, return_sequences=False)      │  │  │   │
│  │  │  │ Output: (128,)                         │  │  │   │
│  │  │  └────────────────────────────────────────┘  │  │   │
│  │  │                                              │  │   │
│  │  │  ┌────────────────────────────────────────┐  │  │   │
│  │  │  │ Output Layer                           │  │  │   │
│  │  │  │ Dense(64, ReLU) ──▶ Dense(1, Sigmoid)  │  │  │   │
│  │  │  │ Output: Stress probability [0,1]       │  │  │   │
│  │  │  │ Scaled to: Stress score [0,100]        │  │  │   │
│  │  │  └────────────────────────────────────────┘  │  │   │
│  │  │                                              │  │   │
│  │  │  Loss: Binary Cross-Entropy                  │  │   │
│  │  │  Optimizer: Adam (lr=0.001)                  │  │   │
│  │  │  Metrics: AUC-ROC, Precision, Recall         │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └─────────────────────┬───────────────────────────────┘   │
│                        │                                    │
│                        ▼                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Model Optimization & Deployment             │   │
│  │                                                     │   │
│  │  Cloud Model ──▶ SageMaker Endpoint (Real-time)    │   │
│  │  Edge Model   ──▶ TFLite Quantization (INT8)       │   │
│  │               ──▶ Model size: 180 KB               │   │
│  │               ──▶ Deploy to: Mobile + Wearable     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 5. Reporting Engine

**Technology**: Python (ReportLab) for PDF generation, Matplotlib for visualizations

**Report Components:**
- Stress trend graphs (7/30/90 days)
- Trigger frequency heatmap
- Event timeline with annotations
- Statistical summary (mean, std, percentiles)
- Clinical notes section
- Intervention history

## Data Flow Design

### End-to-End Data Flow

```
1. SENSOR CAPTURE (Wearable)
   ├─ HR sensor: 1 Hz → 1 sample/sec
   ├─ Accelerometer: 50 Hz → 50 samples/sec
   └─ Microphone: 16 kHz → 16000 samples/sec

2. EDGE PREPROCESSING (Wearable Firmware)
   ├─ MFCC extraction: 25ms windows, 10ms hop → 13 coefficients
   ├─ HRV calculation: 60s windows → SDNN, RMSSD, pNN50, LF/HF, HR
   └─ Motion features: 1s windows → variance, peak count, mean

3. TINYML INFERENCE (Wearable)
   ├─ Input: [MFCC(13), HRV(5), Motion(3)] = 21 features
   ├─ Model: Quantized CNN (INT8), 180 KB
   ├─ Inference time: 50ms
   └─ Output: Stress score (0-100)

4. BLE TRANSMISSION (Wearable → Mobile)
   ├─ Packet format: [timestamp(4), stress_score(1), features(21*4), checksum(2)]
   ├─ Packet size: 93 bytes
   ├─ Frequency: Every 1 second
   ├─ Encryption: AES-128
   └─ Buffering: Up to 4 hours if disconnected

5. MOBILE PROCESSING (Mobile App)
   ├─ BLE reception and decryption
   ├─ Local storage (Hive database)
   ├─ Alert evaluation (if stress > 70 for 5 min)
   ├─ UI update (real-time chart)
   └─ Queue for cloud sync

6. CLOUD INGESTION (When online)
   ├─ HTTPS POST to API Gateway
   ├─ Lambda trigger → Ingestion Service
   ├─ Data validation and enrichment
   ├─ DynamoDB write (Events table)
   └─ S3 write (raw data backup)

7. ML INFERENCE (Cloud AI Engine)
   ├─ Fetch recent data (10-minute window)
   ├─ Feature engineering (temporal patterns)
   ├─ Multimodal model inference
   ├─ Output: Stress probability, escalation risk
   └─ Latency: 500ms

8. DIGITAL TWIN UPDATE (Digital Twin Service)
   ├─ Update behavioral state machine
   ├─ Identify triggers (correlation analysis)
   ├─ Update trigger graph
   ├─ Predict future escalation (1-hour ahead)
   └─ Store updated twin in DynamoDB

9. DECISION ENGINE (Decision Service)
   ├─ Evaluate alert rules
   ├─ Check escalation thresholds
   ├─ Generate explanations (trigger analysis)
   ├─ Route alerts (patient, caregivers, doctor)
   └─ Log decision rationale

10. INTERVENTION DELIVERY
    ├─ Push notification (Firebase CM)
    ├─ SMS fallback (if offline)
    ├─ Wearable vibration (via BLE command)
    └─ Dashboard updates (real-time)

TOTAL LATENCY: Sensor → Alert = 1.5-2 seconds
```

### BLE Packet Format

```
┌────────────────────────────────────────────────────────────┐
│                    BLE Data Packet                         │
├────────────────────────────────────────────────────────────┤
│ Field            │ Size    │ Type    │ Description         │
├──────────────────┼─────────┼─────────┼─────────────────────┤
│ Timestamp        │ 4 bytes │ uint32  │ Unix epoch (sec)    │
│ Stress Score     │ 1 byte  │ uint8   │ 0-100               │
│ Heart Rate       │ 1 byte  │ uint8   │ BPM                 │
│ HRV SDNN         │ 2 bytes │ uint16  │ Milliseconds        │
│ HRV RMSSD        │ 2 bytes │ uint16  │ Milliseconds        │
│ HRV pNN50        │ 1 byte  │ uint8   │ Percentage          │
│ HRV LF/HF        │ 2 bytes │ float16 │ Ratio               │
│ Motion Variance  │ 2 bytes │ float16 │ Normalized          │
│ Motion Peaks     │ 1 byte  │ uint8   │ Count               │
│ MFCC[0-12]       │ 26 bytes│ float16 │ 13 coefficients     │
│ Battery Level    │ 1 byte  │ uint8   │ Percentage          │
│ Device Status    │ 1 byte  │ uint8   │ Flags (charging,etc)│
│ Checksum         │ 2 bytes │ uint16  │ CRC-16              │
├──────────────────┼─────────┼─────────┼─────────────────────┤
│ TOTAL            │ 46 bytes│         │                     │
└────────────────────────────────────────────────────────────┘

Transmission: Every 1 second
Bandwidth: 46 bytes/sec = 368 bps (well within BLE capacity)
```

## Digital Twin Design

The Digital Twin is a computational model that learns and predicts individual behavioral patterns, trigger responses, and crisis escalation trajectories.

### Behavioral State Machine

```
┌─────────────────────────────────────────────────────────────┐
│            Behavioral State Machine                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│         ┌──────────┐                                        │
│    ┌───▶│  CALM    │◀────┐                                 │
│    │    │ (0-30)   │     │                                 │
│    │    └────┬─────┘     │                                 │
│    │         │           │                                 │
│    │         │ stress↑   │ stress↓                         │
│    │         ▼           │                                 │
│    │    ┌──────────┐     │                                 │
│    │    │ ELEVATED │─────┘                                 │
│    │    │ (31-50)  │                                       │
│    │    └────┬─────┘                                       │
│    │         │                                             │
│    │         │ stress↑                                     │
│    │         ▼                                             │
│    │    ┌──────────┐                                       │
│    │    │  ALERT   │                                       │
│    │    │ (51-70)  │                                       │
│    │    └────┬─────┘                                       │
│    │         │                                             │
│    │         │ stress↑ + duration>5min                     │
│    │         ▼                                             │
│    │    ┌──────────┐                                       │
│    └────│ CRITICAL │                                       │
│         │ (71-100) │                                       │
│         └──────────┘                                       │
│                                                             │
│  State Transitions:                                         │
│  - Tracked per user                                         │
│  - Transition probabilities learned from history            │
│  - Dwell time statistics computed                           │
│  - Trigger associations recorded                            │
└─────────────────────────────────────────────────────────────┘
```

### Trigger Memory Graph

```
┌─────────────────────────────────────────────────────────────┐
│              Trigger Memory Graph Structure                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Node: Trigger                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ trigger_id: UUID                                     │  │
│  │ type: [voice_pattern, hrv_drop, motion_agitation,   │  │
│  │        time_of_day, location, social_context]       │  │
│  │ description: String (e.g., "raised voice detected") │  │
│  │ confidence: Float [0,1]                              │  │
│  │ frequency: Int (occurrence count)                    │  │
│  │ avg_stress_delta: Float (stress increase)           │  │
│  │ last_seen: Timestamp                                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Edge: Co-occurrence                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ trigger_a ──[weight]──▶ trigger_b                    │  │
│  │ weight: Float (co-occurrence probability)            │  │
│  │ temporal_lag: Int (seconds between triggers)         │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Example Graph:                                             │
│                                                             │
│      [Raised Voice]                                         │
│            │ 0.7                                            │
│            ▼                                                │
│      [HRV Drop]                                             │
│            │ 0.8                                            │
│            ▼                                                │
│      [Motion Agitation]                                     │
│            │ 0.6                                            │
│            ▼                                                │
│      [Critical State]                                       │
│                                                             │
│  Operations:                                                │
│  - Add trigger: Insert node when new pattern detected      │
│  - Update weights: Increment on co-occurrence              │
│  - Prune: Remove triggers not seen in 90 days              │
│  - Query: Find likely escalation paths                     │
└─────────────────────────────────────────────────────────────┘
```

### Escalation Prediction Model

**Approach**: Sequence-to-sequence LSTM predicting stress trajectory

```
Input: Historical stress scores (past 60 minutes, 1-min resolution)
       Shape: (60, 1)

Architecture:
  LSTM(64, return_sequences=True)
  LSTM(32, return_sequences=False)
  Dense(16, ReLU)
  Dense(12, Linear)  # Predict next 12 time steps (1 hour ahead)

Output: Predicted stress scores (next 60 minutes)
        Shape: (60, 1)

Training:
  - Loss: Mean Squared Error
  - Optimizer: Adam
  - Data: Per-user historical sequences
  - Update frequency: Daily

Prediction Confidence:
  - High: Prediction variance < 10
  - Medium: Prediction variance 10-20
  - Low: Prediction variance > 20
```

### What-If Simulation Engine

Enables testing hypothetical interventions:

```python
# Pseudocode
def simulate_intervention(user_id, current_state, intervention_type):
    """
    Simulate effect of intervention on stress trajectory
    
    Args:
        user_id: User identifier
        current_state: Current stress score and context
        intervention_type: [breathing_exercise, caregiver_call, 
                           medication_reminder, distraction]
    
    Returns:
        predicted_trajectory: Stress scores for next hour
        confidence: Prediction confidence
    """
    
    # Load user's digital twin
    twin = load_digital_twin(user_id)
    
    # Get historical intervention effectiveness
    intervention_effect = twin.get_intervention_effect(intervention_type)
    
    # Predict baseline trajectory (no intervention)
    baseline = twin.predict_trajectory(current_state, duration=60)
    
    # Apply intervention effect model
    intervened = baseline * (1 - intervention_effect.mean_reduction)
    
    # Add uncertainty
    confidence = intervention_effect.confidence
    
    return intervened, confidence

# Usage in decision engine
if stress_score > 70:
    # Test multiple interventions
    breathing_result = simulate_intervention(user, state, "breathing")
    caregiver_result = simulate_intervention(user, state, "caregiver_call")
    
    # Choose most effective
    best_intervention = max([breathing_result, caregiver_result], 
                           key=lambda x: x[0].min())
    
    # Execute
    execute_intervention(best_intervention)
```

### Reinforcement Learning Update

**Approach**: Update intervention policy based on outcomes

```
State: [stress_score, time_of_day, recent_triggers, location_context]
Action: [no_action, breathing_guide, alert_caregiver, alert_doctor]
Reward: -1 * (stress_increase) + 10 * (crisis_prevented)

Algorithm: Deep Q-Network (DQN)
  - State encoder: Dense(64) → Dense(32)
  - Q-value head: Dense(16) → Dense(4)  # 4 actions
  - Target network updated every 1000 steps
  - Experience replay buffer: 10,000 transitions

Training:
  - Batch size: 32
  - Learning rate: 0.0001
  - Discount factor: 0.95
  - Exploration: ε-greedy (ε decays from 0.3 to 0.05)

Update Frequency: Weekly batch training on aggregated user data
```

## AWS Infrastructure Design

### Infrastructure Components

```
┌─────────────────────────────────────────────────────────────┐
│                AWS Infrastructure Architecture              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              INGESTION LAYER                        │   │
│  │                                                     │   │
│  │  Mobile/Wearable                                    │   │
│  │       │                                             │   │
│  │       ▼                                             │   │
│  │  ┌──────────────┐         ┌──────────────────┐     │   │
│  │  │ AWS IoT Core │────────▶│ IoT Rules Engine │     │   │
│  │  │ (MQTT/HTTPS) │         │ (Route to Lambda)│     │   │
│  │  └──────────────┘         └────────┬─────────┘     │   │
│  │                                    │               │   │
│  │                                    ▼               │   │
│  │                           ┌──────────────────┐     │   │
│  │                           │ Lambda Ingestion │     │   │
│  │                           │ (Data validation)│     │   │
│  │                           └────────┬─────────┘     │   │
│  └────────────────────────────────────┼───────────────┘   │
│                                       │                    │
│  ┌────────────────────────────────────┼───────────────┐   │
│  │              COMPUTE LAYER         │               │   │
│  │                                    ▼               │   │
│  │  ┌──────────────────────────────────────────────┐ │   │
│  │  │ Application Load Balancer                    │ │   │
│  │  └────────────────┬─────────────────────────────┘ │   │
│  │                   │                               │   │
│  │     ┌─────────────┼─────────────┐                 │   │
│  │     │             │             │                 │   │
│  │  ┌──▼──────┐  ┌──▼──────┐  ┌──▼──────┐           │   │
│  │  │ ECS     │  │ ECS     │  │ ECS     │           │   │
│  │  │ Fargate │  │ Fargate │  │ Fargate │           │   │
│  │  │ Task 1  │  │ Task 2  │  │ Task N  │           │   │
│  │  │         │  │         │  │         │           │   │
│  │  │ FastAPI │  │ FastAPI │  │ FastAPI │           │   │
│  │  │ Service │  │ Service │  │ Service │           │   │
│  │  └─────────┘  └─────────┘  └─────────┘           │   │
│  │                                                   │   │
│  │  Auto Scaling: Target CPU 70%, Min 2, Max 20     │   │
│  └───────────────────────────────────────────────────┘   │
│                                                           │
│  ┌───────────────────────────────────────────────────┐   │
│  │              STORAGE LAYER                        │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ DynamoDB                                 │    │   │
│  │  │ - On-demand billing                      │    │   │
│  │  │ - Point-in-time recovery                 │    │   │
│  │  │ - Encryption at rest (KMS)               │    │   │
│  │  │ - Global tables (multi-region)           │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ S3                                       │    │   │
│  │  │ - Standard: Hot data (30 days)           │    │   │
│  │  │ - IA: Warm data (31-180 days)            │    │   │
│  │  │ - Glacier: Cold data (>180 days)         │    │   │
│  │  │ - Versioning enabled                     │    │   │
│  │  │ - Server-side encryption (AES-256)       │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ ElastiCache Redis                        │    │   │
│  │  │ - Cache layer for hot data               │    │   │
│  │  │ - Session storage                        │    │   │
│  │  │ - TTL: 1 hour                            │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  └───────────────────────────────────────────────────┘   │
│                                                           │
│  ┌───────────────────────────────────────────────────┐   │
│  │              ML LAYER                             │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ SageMaker                                │    │   │
│  │  │ - Training jobs (weekly)                 │    │   │
│  │  │ - Real-time endpoints (2 instances)      │    │   │
│  │  │ - Batch transform (daily)                │    │   │
│  │  │ - Model registry                         │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  └───────────────────────────────────────────────────┘   │
│                                                           │
│  ┌───────────────────────────────────────────────────┐   │
│  │              SECURITY LAYER                       │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ Cognito User Pools                       │    │   │
│  │  │ - User authentication                    │    │   │
│  │  │ - MFA for providers                      │    │   │
│  │  │ - OAuth 2.0 / OIDC                       │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ KMS (Key Management Service)             │    │   │
│  │  │ - Customer managed keys                  │    │   │
│  │  │ - Automatic rotation                     │    │   │
│  │  │ - Per-user data encryption keys          │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ WAF (Web Application Firewall)           │    │   │
│  │  │ - Rate limiting                          │    │   │
│  │  │ - SQL injection protection               │    │   │
│  │  │ - Geographic restrictions                │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  └───────────────────────────────────────────────────┘   │
│                                                           │
│  ┌───────────────────────────────────────────────────┐   │
│  │              MONITORING LAYER                     │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ CloudWatch                               │    │   │
│  │  │ - Metrics (CPU, memory, latency)         │    │   │
│  │  │ - Logs (centralized)                     │    │   │
│  │  │ - Alarms (threshold-based)               │    │   │
│  │  │ - Dashboards                             │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  │                                                   │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ X-Ray                                    │    │   │
│  │  │ - Distributed tracing                    │    │   │
│  │  │ - Service map                            │    │   │
│  │  │ - Latency analysis                       │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  └───────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### DynamoDB Schema Design

**Table 1: Users**
```json
{
  "TableName": "dialo-users",
  "KeySchema": [
    {"AttributeName": "user_id", "KeyType": "HASH"}
  ],
  "AttributeDefinitions": [
    {"AttributeName": "user_id", "AttributeType": "S"},
    {"AttributeName": "phone_number", "AttributeType": "S"}
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "phone-index",
      "KeySchema": [{"AttributeName": "phone_number", "KeyType": "HASH"}],
      "Projection": {"ProjectionType": "ALL"}
    }
  ],
  "BillingMode": "PAY_PER_REQUEST",
  "StreamSpecification": {
    "StreamEnabled": true,
    "StreamViewType": "NEW_AND_OLD_IMAGES"
  }
}

Sample Record:
{
  "user_id": "usr_a1b2c3d4",
  "phone_number": "+919876543210",
  "name_encrypted": "AES256_ENCRYPTED_DATA",
  "age": 32,
  "gender": "F",
  "language_preference": "hi",
  "created_at": "2026-01-15T10:30:00Z",
  "consent": {
    "data_collection": true,
    "data_sharing_caregivers": true,
    "data_sharing_doctor": true,
    "emergency_override": true,
    "version": "1.0",
    "timestamp": "2026-01-15T10:30:00Z"
  },
  "device_id": "dev_x9y8z7",
  "caregivers": ["usr_e5f6g7h8", "usr_i9j0k1l2"],
  "doctor_id": "doc_m3n4o5p6",
  "asha_worker_id": "asha_q7r8s9t0"
}
```

**Table 2: Events**
```json
{
  "TableName": "dialo-events",
  "KeySchema": [
    {"AttributeName": "user_id", "KeyType": "HASH"},
    {"AttributeName": "timestamp", "KeyType": "RANGE"}
  ],
  "AttributeDefinitions": [
    {"AttributeName": "user_id", "AttributeType": "S"},
    {"AttributeName": "timestamp", "AttributeType": "N"},
    {"AttributeName": "event_type", "AttributeType": "S"}
  ],
  "LocalSecondaryIndexes": [
    {
      "IndexName": "event-type-index",
      "KeySchema": [
        {"AttributeName": "user_id", "KeyType": "HASH"},
        {"AttributeName": "event_type", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }
  ],
  "TimeToLiveSpecification": {
    "Enabled": true,
    "AttributeName": "ttl"
  }
}

Sample Record:
{
  "user_id": "usr_a1b2c3d4",
  "timestamp": 1705315800,
  "event_type": "stress_measurement",
  "stress_score": 75,
  "features": {
    "hr": 92,
    "hrv_sdnn": 28,
    "hrv_rmssd": 22,
    "hrv_pnn50": 5,
    "hrv_lf_hf": 2.8,
    "motion_variance": 0.45,
    "motion_peaks": 12,
    "mfcc": [1.2, -0.5, 0.8, ...]
  },
  "device_id": "dev_x9y8z7",
  "location": {
    "lat": 28.6139,
    "lon": 77.2090,
    "accuracy": 50
  },
  "ttl": 1720867800
}
```

**Table 3: TriggerMap**
```json
{
  "TableName": "dialo-trigger-map",
  "KeySchema": [
    {"AttributeName": "user_id", "KeyType": "HASH"},
    {"AttributeName": "trigger_id", "KeyType": "RANGE"}
  ]
}

Sample Record:
{
  "user_id": "usr_a1b2c3d4",
  "trigger_id": "trg_raised_voice_001",
  "trigger_type": "voice_pattern",
  "description": "Raised voice detected in environment",
  "confidence": 0.85,
  "frequency": 23,
  "avg_stress_delta": 18.5,
  "first_seen": "2026-01-20T14:22:00Z",
  "last_seen": "2026-02-10T09:15:00Z",
  "co_occurring_triggers": [
    {"trigger_id": "trg_hrv_drop_002", "weight": 0.7, "temporal_lag": 120}
  ]
}
```

**Table 4: DigitalTwin**
```json
{
  "TableName": "dialo-digital-twin",
  "KeySchema": [
    {"AttributeName": "user_id", "KeyType": "HASH"}
  ]
}

Sample Record:
{
  "user_id": "usr_a1b2c3d4",
  "baseline_stress": 35.2,
  "state_machine": {
    "current_state": "ELEVATED",
    "state_history": [
      {"state": "CALM", "duration_minutes": 180, "timestamp": "2026-02-15T06:00:00Z"},
      {"state": "ELEVATED", "duration_minutes": 45, "timestamp": "2026-02-15T09:00:00Z"}
    ],
    "transition_probabilities": {
      "CALM_to_ELEVATED": 0.15,
      "ELEVATED_to_ALERT": 0.25,
      "ALERT_to_CRITICAL": 0.40
    }
  },
  "prediction_model": {
    "model_version": "v2.3",
    "accuracy": 0.82,
    "last_trained": "2026-02-14T00:00:00Z"
  },
  "intervention_effectiveness": {
    "breathing_exercise": {"mean_reduction": 0.15, "confidence": 0.78, "n_samples": 45},
    "caregiver_call": {"mean_reduction": 0.32, "confidence": 0.85, "n_samples": 28}
  },
  "updated_at": "2026-02-15T09:45:00Z"
}
```

**Table 5: Alerts**
```json
{
  "TableName": "dialo-alerts",
  "KeySchema": [
    {"AttributeName": "alert_id", "KeyType": "HASH"}
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "user-timestamp-index",
      "KeySchema": [
        {"AttributeName": "user_id", "KeyType": "HASH"},
        {"AttributeName": "created_at", "KeyType": "RANGE"}
      ]
    }
  ]
}

Sample Record:
{
  "alert_id": "alt_u1v2w3x4",
  "user_id": "usr_a1b2c3d4",
  "alert_type": "escalation",
  "severity": "high",
  "stress_score": 78,
  "triggers": ["trg_raised_voice_001", "trg_hrv_drop_002"],
  "explanation": "Stress escalation detected due to raised voice and elevated heart rate",
  "explanation_localized": {
    "hi": "तनाव बढ़ने का पता चला है क्योंकि आवाज़ ऊंची हो गई है और दिल की धड़कन तेज़ है"
  },
  "recipients": [
    {"user_id": "usr_e5f6g7h8", "type": "caregiver", "acknowledged": true, "ack_time": "2026-02-15T10:02:00Z"},
    {"user_id": "doc_m3n4o5p6", "type": "doctor", "acknowledged": false}
  ],
  "intervention_suggested": "breathing_exercise",
  "created_at": "2026-02-15T10:00:00Z",
  "resolved_at": null
}
```

### Scaling Strategy

**Horizontal Scaling:**
- ECS Fargate tasks: Auto-scale based on CPU (target 70%) and request count
- DynamoDB: On-demand billing automatically scales read/write capacity
- SageMaker endpoints: Multi-instance deployment with auto-scaling

**Vertical Scaling:**
- Fargate task size: Start with 0.5 vCPU, 1 GB RAM; scale to 2 vCPU, 4 GB RAM under load
- RDS (if used): Start with db.t3.medium; scale to db.r5.xlarge

**Geographic Scaling:**
- Multi-region deployment: Primary (Mumbai), Secondary (Hyderabad)
- CloudFront CDN for static assets
- Route 53 latency-based routing

**Data Partitioning:**
- DynamoDB: Partition by user_id (natural sharding)
- S3: Partition by date and user_id (s3://bucket/year/month/day/user_id/)

**Caching Strategy:**
- ElastiCache Redis for:
  - User profiles (TTL: 1 hour)
  - Recent stress scores (TTL: 5 minutes)
  - ML model outputs (TTL: 1 minute)
  - API responses (TTL: 30 seconds)

**Cost Optimization:**
- Reserved instances for baseline load
- Spot instances for batch ML training
- S3 lifecycle policies (Standard → IA → Glacier)
- DynamoDB on-demand for variable workloads

## API Design

### REST API Endpoints

**Base URL**: `https://api.dialo.health/v1`

**Authentication**: Bearer token (JWT) from Cognito

#### 1. Wearable Data Ingestion

```
POST /wearable/data
Content-Type: application/json
Authorization: Bearer <token>

Request Body:
{
  "device_id": "dev_x9y8z7",
  "user_id": "usr_a1b2c3d4",
  "timestamp": 1705315800,
  "stress_score": 75,
  "features": {
    "hr": 92,
    "hrv_sdnn": 28,
    "hrv_rmssd": 22,
    "hrv_pnn50": 5,
    "hrv_lf_hf": 2.8,
    "motion_variance": 0.45,
    "motion_peaks": 12,
    "mfcc": [1.2, -0.5, 0.8, ...]
  },
  "battery_level": 65,
  "location": {
    "lat": 28.6139,
    "lon": 77.2090,
    "accuracy": 50
  }
}

Response (200 OK):
{
  "status": "success",
  "event_id": "evt_y9z0a1b2",
  "processed_at": "2026-02-15T10:00:05Z"
}

Response (400 Bad Request):
{
  "status": "error",
  "error_code": "INVALID_DATA",
  "message": "Missing required field: stress_score"
}
```

#### 2. Get Patient Stress Data

```
GET /patient/{user_id}/stress?start_time={unix_timestamp}&end_time={unix_timestamp}&resolution={1m|5m|1h}
Authorization: Bearer <token>

Response (200 OK):
{
  "user_id": "usr_a1b2c3d4",
  "start_time": 1705315800,
  "end_time": 1705402200,
  "resolution": "5m",
  "data_points": [
    {
      "timestamp": 1705315800,
      "stress_score": 35,
      "state": "CALM"
    },
    {
      "timestamp": 1705316100,
      "stress_score": 42,
      "state": "ELEVATED"
    },
    ...
  ],
  "statistics": {
    "mean": 45.2,
    "std": 12.8,
    "min": 28,
    "max": 78,
    "p50": 43,
    "p95": 68
  }
}
```

#### 3. Get Patient Triggers

```
GET /patient/{user_id}/triggers?sort_by={frequency|severity|recent}
Authorization: Bearer <token>

Response (200 OK):
{
  "user_id": "usr_a1b2c3d4",
  "triggers": [
    {
      "trigger_id": "trg_raised_voice_001",
      "type": "voice_pattern",
      "description": "Raised voice detected",
      "frequency": 23,
      "avg_stress_delta": 18.5,
      "last_seen": "2026-02-10T09:15:00Z",
      "confidence": 0.85
    },
    {
      "trigger_id": "trg_hrv_drop_002",
      "type": "physiological",
      "description": "Sudden HRV decrease",
      "frequency": 18,
      "avg_stress_delta": 22.3,
      "last_seen": "2026-02-12T14:30:00Z",
      "confidence": 0.92
    }
  ],
  "co_occurrence_graph": {
    "edges": [
      {
        "from": "trg_raised_voice_001",
        "to": "trg_hrv_drop_002",
        "weight": 0.7,
        "temporal_lag_seconds": 120
      }
    ]
  }
}
```

#### 4. Create Alert Decision

```
POST /decision/alert
Content-Type: application/json
Authorization: Bearer <token>

Request Body:
{
  "user_id": "usr_a1b2c3d4",
  "stress_score": 78,
  "triggers": ["trg_raised_voice_001", "trg_hrv_drop_002"],
  "context": {
    "time_of_day": "morning",
    "location_type": "home"
  }
}

Response (200 OK):
{
  "alert_id": "alt_u1v2w3x4",
  "decision": "alert_caregivers",
  "severity": "high",
  "explanation": "Stress escalation detected due to raised voice and elevated heart rate",
  "explanation_localized": {
    "hi": "तनाव बढ़ने का पता चला है क्योंकि आवाज़ ऊंची हो गई है और दिल की धड़कन तेज़ है",
    "en": "Stress escalation detected due to raised voice and elevated heart rate"
  },
  "recommended_intervention": "breathing_exercise",
  "recipients": [
    {"user_id": "usr_e5f6g7h8", "type": "caregiver", "notification_sent": true},
    {"user_id": "doc_m3n4o5p6", "type": "doctor", "notification_sent": true}
  ],
  "created_at": "2026-02-15T10:00:00Z"
}
```

#### 5. Get Doctor Report

```
GET /doctor/{doctor_id}/report?user_id={user_id}&period={7d|30d|90d}&format={json|pdf}
Authorization: Bearer <token>

Response (200 OK) - JSON format:
{
  "report_id": "rpt_c3d4e5f6",
  "user_id": "usr_a1b2c3d4",
  "period": "30d",
  "generated_at": "2026-02-15T10:00:00Z",
  "summary": {
    "total_events": 1250,
    "escalation_events": 12,
    "avg_stress_score": 42.5,
    "stress_trend": "stable",
    "crisis_prevented": 3
  },
  "stress_analytics": {
    "daily_averages": [...],
    "time_of_day_pattern": {
      "morning": 38.2,
      "afternoon": 45.8,
      "evening": 48.3,
      "night": 35.1
    }
  },
  "trigger_analysis": {
    "top_triggers": [
      {"trigger_id": "trg_raised_voice_001", "frequency": 23, "impact": "high"},
      {"trigger_id": "trg_hrv_drop_002", "frequency": 18, "impact": "high"}
    ],
    "trigger_heatmap": {
      "monday": {"morning": 2, "afternoon": 5, "evening": 3},
      ...
    }
  },
  "intervention_history": [
    {
      "timestamp": "2026-02-10T09:15:00Z",
      "type": "breathing_exercise",
      "effectiveness": "high",
      "stress_before": 75,
      "stress_after": 52
    }
  ],
  "clinical_notes": [
    {
      "timestamp": "2026-02-05T14:00:00Z",
      "author": "Dr. Sharma",
      "note": "Patient responding well to interventions"
    }
  ]
}

Response (200 OK) - PDF format:
Content-Type: application/pdf
Content-Disposition: attachment; filename="patient_report_usr_a1b2c3d4_30d.pdf"
[Binary PDF data]
```

#### 6. Update Consent

```
PUT /patient/{user_id}/consent
Content-Type: application/json
Authorization: Bearer <token>

Request Body:
{
  "data_collection": true,
  "data_sharing_caregivers": true,
  "data_sharing_doctor": true,
  "emergency_override": true,
  "version": "1.0"
}

Response (200 OK):
{
  "status": "success",
  "user_id": "usr_a1b2c3d4",
  "consent_updated_at": "2026-02-15T10:00:00Z",
  "effective_immediately": true
}
```

#### 7. Emergency "Help Nearby" Signal

```
POST /emergency/help-nearby
Content-Type: application/json
Authorization: Bearer <token>

Request Body:
{
  "user_id": "usr_a1b2c3d4",
  "location": {
    "lat": 28.6139,
    "lon": 77.2090,
    "accuracy": 50
  },
  "timestamp": 1705315800
}

Response (200 OK):
{
  "status": "success",
  "alert_id": "alt_emergency_001",
  "caregivers_notified": 2,
  "estimated_arrival_times": [
    {"user_id": "usr_e5f6g7h8", "eta_minutes": 5},
    {"user_id": "usr_i9j0k1l2", "eta_minutes": 12}
  ]
}
```

### WebSocket API (Real-time Updates)

**Endpoint**: `wss://api.dialo.health/v1/ws`

**Connection**:
```javascript
const ws = new WebSocket('wss://api.dialo.health/v1/ws?token=<jwt_token>');

// Subscribe to user's stress updates
ws.send(JSON.stringify({
  "action": "subscribe",
  "channel": "stress_updates",
  "user_id": "usr_a1b2c3d4"
}));

// Receive real-time updates
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  // data = {
  //   "type": "stress_update",
  //   "user_id": "usr_a1b2c3d4",
  //   "stress_score": 75,
  //   "timestamp": 1705315800
  // }
};
```

## Security & Privacy Model

### Consent Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Consent Management Flow                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Initial Onboarding                                      │
│     ┌──────────────────────────────────────────────────┐   │
│     │ User opens app for first time                    │   │
│     │   ↓                                              │   │
│     │ Display consent form (local language)            │   │
│     │   - Data collection explanation                  │   │
│     │   - Storage and retention policy                 │   │
│     │   - Sharing options (caregivers, doctor)         │   │
│     │   - Emergency override option                    │   │
│     │   - Right to revoke                              │   │
│     │   ↓                                              │   │
│     │ User reviews and accepts/declines                │   │
│     │   ↓                                              │   │
│     │ Record consent in DynamoDB with timestamp        │   │
│     │ Generate audit log entry                         │   │
│     └──────────────────────────────────────────────────┘   │
│                                                             │
│  2. Granular Permission Management                          │
│     ┌──────────────────────────────────────────────────┐   │
│     │ User navigates to Settings → Privacy             │   │
│     │   ↓                                              │   │
│     │ Toggle individual permissions:                   │   │
│     │   [✓] Stress monitoring                          │   │
│     │   [✓] Voice analysis                             │   │
│     │   [✓] Location tracking                          │   │
│     │   [✓] Share with caregivers                      │   │
│     │   [✓] Share with doctor                          │   │
│     │   [✓] Emergency override                         │   │
│     │   ↓                                              │   │
│     │ Changes propagate to cloud within 30 seconds     │   │
│     │ Access control policies updated                  │   │
│     └──────────────────────────────────────────────────┘   │
│                                                             │
│  3. Consent Revocation                                      │
│     ┌──────────────────────────────────────────────────┐   │
│     │ User selects "Revoke All Consent"                │   │
│     │   ↓                                              │   │
│     │ Confirmation dialog with consequences            │   │
│     │   ↓                                              │   │
│     │ Stop data transmission from wearable             │   │
│     │ Revoke caregiver/doctor access                   │   │
│     │ Mark user as "consent_revoked" in database       │   │
│     │ Retain data for legal retention period           │   │
│     │ Provide data export option                       │   │
│     └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Encryption Architecture

**Data at Rest:**
- DynamoDB: AWS KMS customer-managed keys (CMK)
- S3: Server-side encryption (SSE-KMS) with per-user keys
- Mobile: AES-256 encryption using device keystore
- Wearable: AES-128 encryption for buffered data

**Data in Transit:**
- Wearable ↔ Mobile: BLE with AES-128 encryption
- Mobile ↔ Cloud: TLS 1.3 with certificate pinning
- Cloud internal: VPC private subnets, no public internet

**Key Management:**
```
┌─────────────────────────────────────────────────────────────┐
│                  Key Management Hierarchy                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Master Key (AWS KMS CMK)                             │  │
│  │ - Automatic rotation: Annual                         │  │
│  │ - Access: IAM policies only                          │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│                       ▼                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Data Encryption Keys (DEK)                           │  │
│  │ - Generated per user                                 │  │
│  │ - Encrypted by Master Key                            │  │
│  │ - Stored in DynamoDB (encrypted)                     │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│                       ▼                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ User Data                                            │  │
│  │ - Encrypted with user's DEK                          │  │
│  │ - Stored in DynamoDB/S3                              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Data Isolation

**Multi-tenancy Strategy:**
- Logical isolation: All queries filtered by user_id
- Row-level security: DynamoDB partition key = user_id
- API authorization: JWT claims validated against requested user_id
- No cross-user data leakage possible

**Role-Based Access Control (RBAC):**

| Role | Permissions |
|------|-------------|
| Patient | Read own data, Update consent, Trigger emergency |
| Caregiver | Read assigned patient's alerts and trends, Acknowledge alerts |
| Doctor | Read assigned patient's full history, Add clinical notes, Export reports |
| ASHA Worker | Read assigned patients' summaries, Record home visits |
| Admin | User management, System configuration, Audit logs |

### Rural Safe Mode

**Offline Privacy Protection:**
- Data stored locally on device encrypted
- No cloud transmission until user explicitly syncs
- Local alerts only (no caregiver notifications)
- Option to delete local data before syncing

**Explainable AI Logs:**
- Every AI decision logged with reasoning
- Trigger identification rationale stored
- Model confidence scores recorded
- Audit trail for regulatory compliance

## Real-Time Pipeline

### Latency Optimization Strategy

**Target**: Sensor capture → Alert delivery < 2 seconds

**Breakdown:**
```
1. Sensor Capture → Wearable Processing: 50ms
   - Sensor sampling: 10ms
   - Feature extraction: 30ms
   - TinyML inference: 50ms
   Total: 90ms

2. BLE Transmission: 100ms
   - Packet preparation: 10ms
   - BLE transmission: 50ms
   - Mobile reception: 10ms
   - Decryption: 30ms
   Total: 100ms

3. Mobile Processing: 200ms
   - Local ML inference: 100ms
   - Alert evaluation: 50ms
   - UI update: 50ms
   Total: 200ms

4. Cloud Sync (when online): 800ms
   - HTTPS request: 200ms
   - API Gateway: 50ms
   - Lambda/Fargate: 300ms
   - ML inference: 500ms
   - Decision engine: 100ms
   - Alert routing: 150ms
   Total: 1300ms (but async, doesn't block local alert)

5. Push Notification: 500ms
   - Firebase CM: 300ms
   - Device delivery: 200ms
   Total: 500ms

TOTAL (Local Alert): 390ms ✓
TOTAL (Cloud Alert): 1890ms ✓
```

### Edge + Cloud Hybrid Inference

**Strategy**: Fast local inference, accurate cloud inference

```
┌─────────────────────────────────────────────────────────────┐
│              Hybrid Inference Architecture                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Wearable (TinyML)                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Quantized CNN (INT8)                                 │  │
│  │ - Accuracy: 75%                                      │  │
│  │ - Latency: 50ms                                      │  │
│  │ - Purpose: Fast screening                            │  │
│  │ - Output: Stress score (0-100)                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                       │                                     │
│                       ▼                                     │
│  Mobile (TFLite)                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Optimized Model (FP16)                               │  │
│  │ - Accuracy: 82%                                      │  │
│  │ - Latency: 100ms                                     │  │
│  │ - Purpose: Local alert generation                    │  │
│  │ - Output: Stress score + confidence                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                       │                                     │
│                       ▼                                     │
│  Cloud (Full Model)                                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Full Multimodal Model (FP32)                         │  │
│  │ - Accuracy: 88%                                      │  │
│  │ - Latency: 500ms                                     │  │
│  │ - Purpose: Trigger identification, prediction        │  │
│  │ - Output: Stress score + triggers + escalation risk  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Decision Logic:                                            │
│  - If wearable score > 70: Immediate local alert            │
│  - If mobile score > 70 for 5 min: Alert caregivers         │
│  - Cloud refines: Identify triggers, update digital twin    │
│  - Cloud prediction: Forecast escalation, suggest action    │
└─────────────────────────────────────────────────────────────┘
```

### Fallback Logic

**Scenario 1: BLE Disconnected**
- Wearable buffers data (up to 4 hours)
- Wearable provides haptic feedback for high stress
- When reconnected, batch upload buffered data

**Scenario 2: Mobile Offline**
- Mobile uses cached TFLite model for inference
- Local alerts displayed to patient
- Caregiver notifications queued for later delivery
- Data synced when connectivity restored

**Scenario 3: Cloud Unavailable**
- Mobile continues local processing
- Alerts based on local model only
- No trigger identification or prediction
- Graceful degradation message to user

**Scenario 4: Model Inference Failure**
- Fall back to rule-based thresholds
- Alert if stress > 75 for 5 minutes
- Log failure for investigation
- Automatic retry with exponential backoff

## Continuous Learning Loop

### Model Retraining Strategy

**Frequency**: Weekly batch updates

**Pipeline:**
```
1. Data Collection (Continuous)
   - Stream events to S3 (partitioned by date)
   - Aggregate user feedback on alert accuracy
   - Collect intervention outcomes

2. Feature Engineering (Daily)
   - AWS Glue ETL jobs
   - Compute temporal features (trends, patterns)
   - Normalize and scale features
   - Store in Feature Store

3. Model Training (Weekly)
   - SageMaker training job
   - Train on past 90 days of data
   - Separate models per user (personalization)
   - Global model for new users (cold start)

4. Model Evaluation (Weekly)
   - Holdout test set (20% of data)
   - Metrics: AUC-ROC, Precision, Recall, F1
   - Fairness metrics across demographics
   - Latency benchmarks

5. A/B Testing (2 weeks)
   - Deploy new model to 10% of users
   - Compare performance vs. current model
   - Monitor alert accuracy and user feedback
   - Statistical significance testing

6. Model Deployment (After A/B test)
   - Gradual rollout: 10% → 50% → 100%
   - Canary deployment with automatic rollback
   - Update mobile TFLite models via app update
   - Update wearable models via OTA firmware update

7. Monitoring (Continuous)
   - Track model performance metrics
   - Detect model drift (data distribution changes)
   - Alert on performance degradation
   - Trigger retraining if needed
```

### Personalization Layer

**Approach**: Transfer learning from global model

```
Global Model (Pre-trained)
  ↓
User-Specific Fine-Tuning (after 7 days of data)
  - Freeze encoder layers
  - Train only fusion and output layers
  - Use user's historical data
  - Update Digital Twin parameters
  ↓
Personalized Model (per user)
  - Stored in S3: s3://models/user_{user_id}/model_v{version}.pb
  - Loaded for inference
  - Updated weekly
```

### Federated Learning (Future Option)

**Concept**: Train models on-device without centralizing data

**Benefits:**
- Enhanced privacy (data never leaves device)
- Reduced cloud costs
- Personalization without data sharing

**Implementation Plan:**
- Phase 1: Centralized training (current)
- Phase 2: Federated averaging (aggregate gradients)
- Phase 3: Differential privacy (add noise to gradients)

## Future Scalability

### Multi-Tenant Architecture

**Current**: Single deployment per region
**Future**: Multi-tenant SaaS platform

**Isolation Strategy:**
- Tenant ID in all database records
- Separate S3 buckets per tenant
- Tenant-specific encryption keys
- Resource quotas per tenant

### State-Wide Deployment

**Target**: 10 million users across Indian states

**Infrastructure Requirements:**
- Multi-region deployment (Mumbai, Hyderabad, Bangalore)
- DynamoDB global tables for cross-region replication
- CloudFront for low-latency API access
- Regional SageMaker endpoints

**Data Residency:**
- Store data in region closest to user
- Comply with state-level data localization laws
- Cross-region replication for disaster recovery

### ASHA Worker Dashboards

**Features:**
- Mobile-first design (Android app)
- Simplified UI for low digital literacy
- Offline-first with sync when connected
- Voice-based navigation (Hindi, regional languages)
- Patient list with priority indicators
- Home visit scheduling and tracking
- Referral workflow to doctors

### Offline Village Sync Node

**Concept**: Community hub for data synchronization

**Implementation:**
- Raspberry Pi with 4G modem at village health center
- Local WiFi hotspot for patient devices
- Batch sync patient data to cloud
- Cache ML models for distribution
- Emergency alert relay via SMS

**Benefits:**
- Reduces individual data costs
- Enables monitoring in zero-connectivity areas
- Community-based support model
- Scalable to remote regions

---

## Conclusion

This design document provides a comprehensive technical blueprint for the Dialo mental health monitoring system. The architecture prioritizes real-time performance, offline-first operation, privacy protection, and scalability to serve rural Indian populations. The multimodal AI approach combined with behavioral digital twins enables personalized, predictive interventions that can reduce mental health crises and improve quality of life for patients and their families.
