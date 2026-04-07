Industrial Equipment Fault Detection System
![Platform](https://img.shields.io/badge/Platform-STM32%20%7C%20Python-blue)
![Sensors](https://img.shields.io/badge/Sensors-Vibration%20%7C%20Temp%20%7C%20Current-green)
![Protocol](https://img.shields.io/badge/Protocol-UART%20%7C%20Modbus%20RTU-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
---
Overview
This project implements a multi-channel industrial equipment fault detection system for rotating machinery — specifically targeting electric motors, pumps, and conveyor drives in a manufacturing environment. The system continuously samples vibration (via ADXL345 accelerometer), winding temperature (via PT100/MAX31865), and motor phase current (via ACS712 Hall-effect sensor), applies statistical fault detection algorithms in real time, and transmits structured fault data over Modbus RTU to a supervisory monitoring station.
The system is designed to detect four primary fault classes: bearing degradation (vibration frequency signature), winding overtemperature, motor overload (overcurrent), and complete sensor failure (open/short circuit detection). Fault events are stored in a ring-buffer log and can be retrieved over serial by a connected HMI or SCADA system.
---
Problem Statement
Unplanned equipment downtime in manufacturing is costly — industry estimates suggest a single hour of downtime on a production line can cost £5,000–£50,000 depending on sector. The majority of motor failures (bearing failures ~41%, stator faults ~37%) are detectable weeks in advance through characteristic signatures in vibration and temperature data.
Maintenance teams typically rely on periodic scheduled checks (time-based maintenance), which either miss developing faults between checks or waste resources on equipment that is still healthy. This system enables condition-based maintenance — only inspecting or replacing equipment when fault indicators cross validated thresholds.
---
Features
Three-channel sensor acquisition: vibration (XYZ), temperature, phase current
Real-time RMS and peak vibration calculation with 512-point rolling window
FFT-based frequency analysis for bearing fault signature detection (BPFO/BPFI)
Temperature rate-of-change monitoring (detects rapid thermal runaway)
Overcurrent detection with configurable trip threshold and time-inverse curve
Sensor open/short circuit detection (built-in self-test on startup)
Four fault classes with individual severity levels (WARNING / CRITICAL)
Fault event ring-buffer (100 events) with millisecond timestamps
Modbus RTU communication (RS-485) to SCADA / HMI
Local LED and buzzer annunciation
---
Tech Stack
Layer	Technology
Microcontroller	STM32F303RE (ARM Cortex-M4 with FPU — required for FFT)
Vibration Sensor	ADXL345 accelerometer (SPI, ±16g, 3200 Hz ODR)
Temperature Sensor	PT100 RTD via MAX31865 SPI amplifier
Current Sensor	ACS712-20A (analogue output, ±20A, 12-bit ADC)
Comms Interface	RS-485 transceiver (MAX485) — Modbus RTU
DSP Library	ARM CMSIS-DSP (arm_rfft_fast_f32)
IDE	STM32CubeIDE
Language	Embedded C (C99)
Monitoring GUI	Python 3 + Modbus (pymodbus) + matplotlib
---
Repository Structure
```
project3-fault-detection-system/
├── README.md
├── code/
│   ├── firmware/
│   │   ├── main.c
│   │   ├── adxl345\_driver.c / .h
│   │   ├── max31865\_driver.c / .h
│   │   ├── acs712\_driver.c / .h
│   │   ├── fault\_engine.c / .h
│   │   ├── modbus\_rtu.c / .h
│   │   ├── ring\_buffer.c / .h
│   │   └── fft\_analysis.c / .h
│   └── monitoring\_gui/
│       ├── monitor.py
│       ├── modbus\_client.py
│       └── requirements.txt
├── diagrams/
│   ├── system\_block\_diagram.png
│   ├── sensor\_wiring\_schematic.png
│   ├── fault\_detection\_flowchart.png
│   └── fft\_bearing\_fault\_plot.png
├── docs/
│   ├── modbus\_register\_map.md
│   ├── fault\_classification.md
│   ├── bearing\_fault\_frequencies.md
│   └── test\_report.md
└── images/
    ├── hardware\_assembled.jpg
    ├── gui\_screenshot.png
    └── oscilloscope\_captures/
```
---
System Architecture
```
┌──────────────────────────────────────────────────────────────────┐
│                    MONITORED EQUIPMENT                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │   ADXL345    │  │  MAX31865    │  │      ACS712-20A        │  │
│  │ Accelerometer│  │ PT100 Amp    │  │  Phase Current Sensor  │  │
│  │  (on motor   │  │ (winding     │  │  (inline on phase L1)  │  │
│  │   housing)   │  │  temp probe) │  │                        │  │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬───────────┘  │
│         │ SPI             │ SPI                   │ Analogue      │
└─────────┼─────────────────┼───────────────────────┼──────────────┘
          │                 │                       │
          ▼                 ▼                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                STM32F303RE — FAULT DETECTION NODE                │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                 │
│  │ ADXL345    │  │ MAX31865   │  │  12-bit    │                 │
│  │ SPI Driver │  │ SPI Driver │  │  ADC Ch1   │                 │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                 │
│        │               │               │                        │
│        └───────────────┼───────────────┘                        │
│                        ▼                                        │
│              ┌──────────────────┐                               │
│              │  FAULT ENGINE    │                               │
│              │  - RMS/Peak calc │                               │
│              │  - FFT analysis  │                               │
│              │  - Threshold chk │                               │
│              │  - Fault logging │                               │
│              └────────┬─────────┘                               │
│                       │                                         │
│              ┌────────▼─────────┐  ┌───────────────────────┐   │
│              │   Modbus RTU     │  │  Local Annunciation   │   │
│              │   TX/RX Handler  │  │  LED + Buzzer outputs │   │
│              └────────┬─────────┘  └───────────────────────┘   │
└───────────────────────┼────────────────────────────────────────┘
                        │ RS-485 (MAX485)
                        │ Modbus RTU @ 19200 baud
                        ▼
          ┌─────────────────────────┐
          │   SCADA / HMI Station   │
          │   Python monitoring GUI │
          │   Fault history viewer  │
          └─────────────────────────┘
```
---
Fault Classification
Fault Class	Sensor	Detection Method	Warning Threshold	Critical Threshold
Bearing Degradation	ADXL345	FFT peak at BPFO/BPFI	RMS > 2.0 g	RMS > 4.5 g
Winding Overtemperature	PT100/MAX31865	Absolute + rate-of-change	> 80°C or > 5°C/min	> 105°C or > 15°C/min
Motor Overload	ACS712	RMS current vs rated	> 110% FLC	> 125% FLC
Sensor Failure	All	Open/short circuit check	—	Out-of-range on self-test
> FLC = Full Load Current (configured per motor nameplate)
> BPFO/BPFI = Ball Pass Frequency Outer/Inner race (bearing fault signatures)
---
Code — Key Modules
ADXL345 High-Speed Acquisition (`adxl345\_driver.c`)
```c
#define ADXL345\_ADDR\_SPI    0x80    // Read bit set
#define ADXL345\_DATAX0      0x32
#define ADXL345\_BW\_RATE     0x2C
#define ADXL345\_POWER\_CTL   0x2D
#define ADXL345\_DATA\_FORMAT 0x31

HAL\_StatusTypeDef ADXL345\_Init(SPI\_HandleTypeDef \*hspi)
{
    uint8\_t tx\[2];

    /\* Set output data rate: 1600 Hz (0x0E) \*/
    tx\[0] = ADXL345\_BW\_RATE;  tx\[1] = 0x0E;
    ADXL345\_WriteReg(hspi, tx);

    /\* Set range: ±16g, full resolution \*/
    tx\[0] = ADXL345\_DATA\_FORMAT;  tx\[1] = 0x0B;
    ADXL345\_WriteReg(hspi, tx);

    /\* Measurement mode on \*/
    tx\[0] = ADXL345\_POWER\_CTL;  tx\[1] = 0x08;
    return ADXL345\_WriteReg(hspi, tx);
}

void ADXL345\_ReadAccelBurst(SPI\_HandleTypeDef \*hspi,
                             int16\_t \*x, int16\_t \*y, int16\_t \*z)
{
    uint8\_t tx = ADXL345\_ADDR\_SPI | ADXL345\_DATAX0 | 0x40; // Multi-byte read
    uint8\_t rx\[7];

    HAL\_GPIO\_WritePin(ACCEL\_CS\_PORT, ACCEL\_CS\_PIN, GPIO\_PIN\_RESET);
    HAL\_SPI\_TransmitReceive(hspi, \&tx, rx, 7, HAL\_MAX\_DELAY);
    HAL\_GPIO\_WritePin(ACCEL\_CS\_PORT, ACCEL\_CS\_PIN, GPIO\_PIN\_SET);

    \*x = (int16\_t)(rx\[2] << 8 | rx\[1]);
    \*y = (int16\_t)(rx\[4] << 8 | rx\[3]);
    \*z = (int16\_t)(rx\[6] << 8 | rx\[5]);
}
```
FFT Vibration Analysis (`fft\_analysis.c`)
```c
#include "arm\_math.h"
#include "fft\_analysis.h"

#define FFT\_SIZE        512
#define SAMPLE\_RATE\_HZ  1600.0f

static float32\_t fft\_input\[FFT\_SIZE];
static float32\_t fft\_output\[FFT\_SIZE];
static arm\_rfft\_fast\_instance\_f32 fft\_instance;

void FFT\_Init(void)
{
    arm\_rfft\_fast\_init\_f32(\&fft\_instance, FFT\_SIZE);
}

/\*\*
 \* @brief  Run FFT on vibration buffer and check for bearing fault frequencies
 \* @param  raw\_buf   512 float samples (acceleration in g)
 \* @param  result    Output: frequency bin magnitudes
 \* @retval BearingFaultFlag — bitmask of detected fault frequencies
 \*/
uint8\_t FFT\_AnalyseVibration(float32\_t \*raw\_buf, float32\_t \*peak\_freq, float32\_t \*peak\_mag)
{
    /\* Copy input and apply Hanning window to reduce spectral leakage \*/
    for (int i = 0; i < FFT\_SIZE; i++) {
        float32\_t w = 0.5f \* (1.0f - arm\_cos\_f32(2.0f \* PI \* i / (FFT\_SIZE - 1)));
        fft\_input\[i] = raw\_buf\[i] \* w;
    }

    /\* Compute real FFT \*/
    arm\_rfft\_fast\_f32(\&fft\_instance, fft\_input, fft\_output, 0);

    /\* Calculate magnitude spectrum \*/
    arm\_cmplx\_mag\_f32(fft\_output, fft\_output, FFT\_SIZE / 2);

    /\* Find peak frequency bin \*/
    float32\_t max\_val;
    uint32\_t max\_idx;
    arm\_max\_f32(fft\_output + 1, FFT\_SIZE / 2 - 1, \&max\_val, \&max\_idx); // skip DC
    max\_idx += 1;

    \*peak\_freq = (float32\_t)max\_idx \* SAMPLE\_RATE\_HZ / FFT\_SIZE;
    \*peak\_mag  = max\_val;

    /\* Check against known bearing fault frequencies
       Example: 6205 bearing, motor at 1450 RPM
       BPFO ≈ 86.9 Hz, BPFI ≈ 111.1 Hz — configured in fault\_config.h \*/
    uint8\_t fault\_flags = 0;
    if (FFT\_CheckFrequencyBand(fft\_output, BEARING\_BPFO\_HZ, 5.0f, BEARING\_FAULT\_MAG\_THRESHOLD))
        fault\_flags |= FAULT\_BEARING\_OUTER;
    if (FFT\_CheckFrequencyBand(fft\_output, BEARING\_BPFI\_HZ, 5.0f, BEARING\_FAULT\_MAG\_THRESHOLD))
        fault\_flags |= FAULT\_BEARING\_INNER;

    return fault\_flags;
}
```
Fault Engine (`fault\_engine.c`)
```c
#include "fault\_engine.h"
#include "ring\_buffer.h"

static FaultEvent\_t fault\_log\[FAULT\_LOG\_SIZE];
static RingBuffer\_t  fault\_ring;
static FaultState\_t  current\_faults;

void FaultEngine\_Init(void)
{
    RingBuffer\_Init(\&fault\_ring, fault\_log, FAULT\_LOG\_SIZE);
    memset(\&current\_faults, 0, sizeof(current\_faults));
}

void FaultEngine\_Update(SensorData\_t \*data)
{
    FaultEvent\_t event;
    uint32\_t now = HAL\_GetTick();

    /\* --- Vibration RMS check --- \*/
    float rms = 0.0f;
    for (int i = 0; i < VIB\_WINDOW; i++) rms += data->vib\_buf\[i] \* data->vib\_buf\[i];
    rms = sqrtf(rms / VIB\_WINDOW);

    if (rms > VIB\_CRITICAL\_G) {
        event = (FaultEvent\_t){ .timestamp = now, .class = FAULT\_VIBRATION,
                                 .severity = SEV\_CRITICAL, .value = rms };
        RingBuffer\_Push(\&fault\_ring, \&event);
        current\_faults.vibration = SEV\_CRITICAL;
    } else if (rms > VIB\_WARNING\_G) {
        current\_faults.vibration = SEV\_WARNING;
    } else {
        current\_faults.vibration = SEV\_OK;
    }

    /\* --- Temperature check (absolute + rate of change) --- \*/
    float temp\_roc = (data->temperature - data->prev\_temperature) /
                     (SAMPLE\_INTERVAL\_S);   // °C/s → scaled to °C/min in threshold

    if (data->temperature > TEMP\_CRITICAL\_C || temp\_roc > TEMP\_ROC\_CRITICAL) {
        event = (FaultEvent\_t){ .timestamp = now, .class = FAULT\_TEMPERATURE,
                                 .severity = SEV\_CRITICAL, .value = data->temperature };
        RingBuffer\_Push(\&fault\_ring, \&event);
        current\_faults.temperature = SEV\_CRITICAL;
    } else if (data->temperature > TEMP\_WARNING\_C || temp\_roc > TEMP\_ROC\_WARNING) {
        current\_faults.temperature = SEV\_WARNING;
    } else {
        current\_faults.temperature = SEV\_OK;
    }

    /\* --- Overcurrent check --- \*/
    if (data->current\_rms > (MOTOR\_FLC\_A \* 1.25f)) {
        event = (FaultEvent\_t){ .timestamp = now, .class = FAULT\_OVERCURRENT,
                                 .severity = SEV\_CRITICAL, .value = data->current\_rms };
        RingBuffer\_Push(\&fault\_ring, \&event);
        current\_faults.current = SEV\_CRITICAL;
    } else if (data->current\_rms > (MOTOR\_FLC\_A \* 1.10f)) {
        current\_faults.current = SEV\_WARNING;
    } else {
        current\_faults.current = SEV\_OK;
    }

    /\* --- Update annunciation outputs --- \*/
    Annunciator\_Update(\&current\_faults);
}
```
Modbus RTU Register Map
Register	Address	Type	Description
Vibration RMS	40001	FLOAT32	Current vibration RMS (g)
Peak Frequency	40003	FLOAT32	Dominant FFT frequency (Hz)
Temperature	40005	FLOAT32	Winding temperature (°C)
Phase Current	40007	FLOAT32	RMS phase current (A)
Fault Status	40009	UINT16	Bitmask of active faults
Fault Count	40010	UINT16	Total faults since startup
Uptime	40011	UINT32	System uptime (seconds)
Fault Log [0..99]	40100+	STRUCT	Fault event ring buffer
Python Monitoring GUI (`monitor.py`)
```python
#!/usr/bin/env python3
"""
Industrial Equipment Fault Detection — Monitoring GUI
Reads live data from STM32 node over Modbus RTU (RS-485)
"""

import time
import struct
from pymodbus.client import ModbusSerialClient
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from collections import deque

# Configuration
SERIAL\_PORT  = "COM3"       # or "/dev/ttyUSB0" on Linux
BAUD\_RATE    = 19200
SLAVE\_ADDR   = 1
POLL\_RATE\_HZ = 2

# Data buffers for live plot (last 60 seconds at 2 Hz)
time\_buf   = deque(maxlen=120)
vib\_buf    = deque(maxlen=120)
temp\_buf   = deque(maxlen=120)
curr\_buf   = deque(maxlen=120)

client = ModbusSerialClient(method='rtu', port=SERIAL\_PORT,
                             baudrate=BAUD\_RATE, timeout=1)

FAULT\_LABELS = {
    0x0001: "VIBRATION WARNING",
    0x0002: "VIBRATION CRITICAL",
    0x0004: "TEMP WARNING",
    0x0008: "TEMP CRITICAL",
    0x0010: "CURRENT WARNING",
    0x0020: "CURRENT CRITICAL",
    0x0040: "SENSOR FAILURE"
}

def read\_float32(registers):
    """Convert two Modbus registers to IEEE 754 float."""
    raw = (registers\[0] << 16) | registers\[1]
    return struct.unpack('>f', struct.pack('>I', raw))\[0]

def poll\_data():
    """Poll all live data registers."""
    result = client.read\_holding\_registers(0, 12, slave=SLAVE\_ADDR)
    if result.isError():
        return None

    r = result.registers
    return {
        'vib\_rms':  read\_float32(r\[0:2]),
        'peak\_freq':read\_float32(r\[2:4]),
        'temp':     read\_float32(r\[4:6]),
        'current':  read\_float32(r\[6:8]),
        'faults':   r\[8],
        'fault\_cnt':r\[9],
        'uptime':   (r\[10] << 16) | r\[11]
    }

def decode\_faults(fault\_mask):
    active = \[label for bit, label in FAULT\_LABELS.items() if fault\_mask \& bit]
    return active if active else \["OK — No faults"]

def update\_plot(frame):
    data = poll\_data()
    if not data: return

    t = len(time\_buf)
    time\_buf.append(t / POLL\_RATE\_HZ)
    vib\_buf.append(data\['vib\_rms'])
    temp\_buf.append(data\['temp'])
    curr\_buf.append(data\['current'])

    ax1.clear(); ax2.clear(); ax3.clear()
    ax1.plot(time\_buf, vib\_buf, 'b-', linewidth=1.5)
    ax1.axhline(y=2.0, color='orange', linestyle='--', label='WARNING 2.0g')
    ax1.axhline(y=4.5, color='red',    linestyle='--', label='CRITICAL 4.5g')
    ax1.set\_ylabel("Vibration RMS (g)"); ax1.legend(fontsize=8)
    ax1.set\_title(f"Fault Status: {' | '.join(decode\_faults(data\['faults']))}")

    ax2.plot(time\_buf, temp\_buf, 'r-', linewidth=1.5)
    ax2.axhline(y=80,  color='orange', linestyle='--')
    ax2.axhline(y=105, color='red',    linestyle='--')
    ax2.set\_ylabel("Temperature (°C)")

    ax3.plot(time\_buf, curr\_buf, 'g-', linewidth=1.5)
    ax3.set\_ylabel("Phase Current (A)")
    ax3.set\_xlabel("Time (s)")

if \_\_name\_\_ == "\_\_main\_\_":
    client.connect()
    fig, (ax1, ax2, ax3) = plt.subplots(3, 1, figsize=(12, 8))
    fig.suptitle("Industrial Equipment Fault Monitor", fontsize=14, fontweight='bold')
    ani = animation.FuncAnimation(fig, update\_plot,
                                   interval=1000 // POLL\_RATE\_HZ, cache\_frame\_data=False)
    plt.tight\_layout()
    plt.show()
    client.close()
```
---
Flowchart Description
```
\[SYSTEM STARTUP]
       │
\[Self-Test: Check all sensors in range]
       ├── ADXL345: verify I0 register = 0xE5
       ├── MAX31865: verify fault register = 0x00
       └── ACS712: verify ADC mid-scale ± 10%
       │
\[FAIL self-test?] ──YES──→ \[Log SENSOR\_FAILURE fault] → \[Halt + alarm]
       │NO
       ▼
\[Start DMA sampling: ADXL345 @ 1600 Hz into 512-sample buffer]
\[Start ADC DMA: ACS712 @ 10 kHz]
\[Start SPI periodic: MAX31865 @ 1 Hz]
       │
\[MAIN ACQUISITION LOOP (triggered by DMA complete callback)]
       │
       ▼
┌─────────────────────────────────────────────────┐
│              FAULT ENGINE UPDATE                 │
│                                                  │
│  \[Calculate vibration RMS from 512-pt window]   │
│            │                                     │
│  \[Run FFT → detect BPFO/BPFI peaks]             │
│            │                                     │
│  \[Calculate temperature rate of change]         │
│            │                                     │
│  \[Calculate current RMS from 100 samples]       │
│            │                                     │
│  \[Check all thresholds]                         │
│     ├── WARNING?  → Set warning flag + log       │
│     └── CRITICAL? → Set critical flag + log      │
│            │                                     │
│  \[Update Modbus registers with latest data]     │
│            │                                     │
│  \[Update LED/buzzer annunciators]               │
│                                                  │
└──────────────────────────────────┬──────────────┘
                                   │
                    \[Await next DMA buffer complete]
                    (non-blocking via interrupt)

\[MODBUS RTU ISR — parallel to above]
  \[Receive query from SCADA]
  \[Service register read/write]
  \[Respond within 1.5ms turnaround]
```
---
Expected Results
Normal operation: All three channels show green status; Modbus registers update every 320 ms (1600 Hz / 512 samples)
Bearing fault simulation (imbalance weight on shaft): FFT shows peak at calculated BPFO frequency; WARNING triggers when peak magnitude exceeds threshold
Temperature fault simulation (heat gun on probe): WARNING at 80°C with rate-of-change monitoring flagging rapid rises
Overcurrent simulation (motor loaded beyond FLC): CRITICAL flag appears on GUI within 2 seconds of sustained overload
Python GUI shows live three-channel stripchart with threshold lines and fault status banner
---
Testing & Validation
Test	Method	Pass Criterion
Sensor self-test	Power cycle; observe startup self-test results	All three channels PASS
Vibration baseline	Motor at no-load; measure RMS	< 0.5g at no-load
Bearing fault detection	Add 5g imbalance weight; spin motor	FFT peak within ±5 Hz of calculated BPFO
Temperature alert	Apply heat gun to PT100; reach 85°C	WARNING triggers ≤ 3s of crossing 80°C
Overcurrent detection	Load motor to 120% FLC	WARNING within 2 update cycles
Modbus comms	Query all registers via Modbus Poll software	All registers return valid values; no CRC errors
Ring buffer	Generate 110 fault events	Only latest 100 retained; oldest overwritten correctly
Python GUI	Run for 30 minutes continuous	No crashes, memory leaks, or missed polls
---
Challenges & Solutions
Challenge 1: FFT spectral leakage masking bearing frequency
With a rectangular window, spectral leakage from the dominant 1× shaft frequency masked the bearing fault sideband frequencies.
Solution: Applied Hanning window before FFT (implemented in CMSIS-DSP), which reduced sideband leakage by ~30 dB and made BPFO peak clearly detectable.
Challenge 2: ACS712 noise floor too high for accurate RMS
At low motor loads, the ACS712's inherent noise (±25 mV → ±125 mA) produced significant error in RMS current calculation.
Solution: Averaged 100 ADC samples per RMS window (oversampling) and applied 1st-order IIR filter. Effective noise floor reduced to ±15 mA.
Challenge 3: Modbus response time conflicting with DMA sampling
Early firmware used polling in the Modbus handler, which blocked DMA sample collection and caused buffer overruns.
Solution: Moved Modbus TX/RX entirely to UART interrupt-driven mode with a 128-byte ring buffer; DMA sampling continues uninterrupted in background.
---
Industrial Relevance
Condition monitoring systems based on this architecture are widely deployed in:
Predictive maintenance programmes: Factories using ISO 10816 vibration severity standards
Motor management relays: ABB, Siemens, and Rockwell all sell dedicated motor protection units using equivalent sensing approaches
IIoT gateway nodes: Edge devices in Industry 4.0 architectures, reporting to cloud SCADA via Modbus-to-MQTT bridges
Mining and heavy industry: Conveyor belt drive monitoring where unplanned downtime costs are extreme
The Modbus RTU interface makes this system immediately compatible with existing SCADA infrastructure (Ignition, WonderWare, FactoryTalk) without hardware changes.
---
Future Improvements
Implement ISO 10816-3 vibration severity classifications directly in firmware
Add machine learning anomaly detection (autoencoder on STM32 using CMSIS-NN or X-CUBE-AI)
Migrate to Modbus TCP over Ethernet (W5500 SPI module) for plant-wide connectivity
Add acoustic emission sensor channel for early-stage bearing crack detection
Implement OTA firmware update over RS-485 for remote field upgrades
Develop web dashboard to replace Python GUI for browser-based monitoring
---
Author
Developed as part of an industrial automation engineering portfolio, demonstrating applied condition monitoring and predictive maintenance system design.
