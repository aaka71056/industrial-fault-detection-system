# Fault Detection Logic

## Objective
To monitor industrial equipment health by analysing vibration, temperature, and current signals and identifying abnormal operating conditions.

## Inputs
- Vibration sensor data
- Temperature sensor data
- Current sensor data

## Processing
1. Read all sensor values
2. Apply filtering and normalization
3. Compare values against predefined thresholds
4. Perform FFT analysis on vibration data
5. Detect abnormal conditions
6. Classify fault type and severity
7. Send results via Modbus RTU

## Fault Classes
- Normal Condition
- Overtemperature Fault
- Overcurrent Fault
- Excessive Vibration Fault
- Bearing Fault Signature Detected

## Outputs
- Fault status
- Severity level
- Sensor values
- Modbus register updates
