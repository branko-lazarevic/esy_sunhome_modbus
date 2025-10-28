# ESY Sunhome HM6 - Modbus Integration

A work-in-progress ESPHome configuration for monitoring the ESY Sunhome HM6 via Modbus RS485.

For those who are looking to purchase/install a sunhome battery, get $50 off using my code: AU1587 when you register and set up the ESY sunhome app. 

If, like me, you're switching to Amber for wholesale rates, use my code QVLA4DT4 to get $120 off.

## Hardware Requirements

- [**M5 Stack Atom S3 Lite**](https://shop.m5stack.com/products/atoms3-lite-esp32s3-dev-kit) (or similar ESP32-S3)
- [**M5Stack Isolated RS485 Unit**](https://shop.m5stack.com/products/isolated-rs485-unit) (or similar RS485 to UART converter)
- **120Ω resistance** 120Ω Termination Resistor
- **Inverter**: ESY Sunhome HM6 Inverter

### Wiring

```
ESP32-S3        RS485 Module        Inverter
GPIO2 (TX)  --> TX/DI           
GPIO1 (RX)  <-- RX/RO           
5V          --> VCC                 
GND         --> GND                 
                A/+ (RS485)     --> A/+ (PIN 5)
                B/- (RS485)     --> B/- (PIN 4)
```

## Register Map

All registers discovered and documented below. Addresses shown are 0-based (as used in code).

### Input Registers (Function Code 4)

These registers are read-only and contain real-time measurements.

| Register | Address | Name | Type | Unit | Scale | Description |
|----------|---------|------|------|------|-------|-------------|
| **BATTERY** |
| R29 | 28 | Battery Mode | UINT16 | - | 1 | 0=Unknown, 1=Charging, 2=Topping, 3=Unknown, 4=Full, 5=Discharging |
| R32 | 31 | Battery Power | INT16 | W | 1 | Positive=Charging, Negative=Discharging |
| R33 | 32 | Battery SOC | UINT16 | % | 1 | State of Charge (0-100%) |
| R34 | 33 | Battery Voltage? | UINT16 | V | 0.1 | Candidate - needs verification |
| R35 | 34 | Battery Current? | INT16 | A | 0.01 | Candidate - needs verification |
| **SOLAR / PV** |
| R23 | 22 | PV1 Power | UINT16 | W | 1 | Solar Panel String 1 Power |
| R26 | 25 | PV2 Power | UINT16 | W | 1 | Solar Panel String 2 Power |
| R24 | 23 | PV1 Voltage? | UINT16 | V | 0.1 | Candidate - needs verification |
| R27 | 26 | PV2 Voltage? | UINT16 | V | 0.1 | Candidate - needs verification |
| **GRID** |
| R40 | 39 | Grid Frequency | UINT16 | Hz | 0.01 | AC Grid Frequency (e.g., 5000 = 50.00 Hz) |
| R43 | 42 | Grid AC Voltage | UINT16 | V | 0.1 | AC Grid Voltage (e.g., 2300 = 230.0V) |
| R50 | 49 | Grid Power | INT16 | W | 1 | Positive=Import, Negative=Export |
| R51 | 50 | Grid Current? | INT16 | A | 0.01 | Candidate - needs verification |
| R55 | 54 | Grid Frequency Alt | UINT16 | Hz | 0.01 | Alternate frequency reading |
| **LOAD** |
| R91 | 90 | Load Power | UINT16 | W | 1 | Total household load power |
| R92 | 91 | Load Current? | UINT16 | A | 0.01 | Candidate - needs verification |
| **INVERTER STATUS** |
| R58 | 57 | External PV AC Voltage | UINT16 | V | 0.1 | External AC voltage reading |
| R76 | 75 | Internal PV AC Voltage | UINT16 | V | 0.1 | Internal AC voltage reading |
| R77 | 76 | Inverter Temp? | INT16 | °C | 0.1 | Candidate - needs verification |
| R78 | 77 | Battery Temp? | INT16 | °C | 0.1 | Candidate - needs verification |

### Holding Registers (Function Code 3)

These registers may be read/write and typically contain configuration or累积 values.

| Register | Address | Name | Type | Unit | Scale | Description |
|----------|---------|------|------|------|-------|-------------|
| R31 | 30 | Daily Power Generation | UINT16 | kWh | 0.001 | Daily solar generation (resets daily) |
| R32-33 | 31-32 | Total Energy Generated? | UINT32 | kWh | 0.1 | Lifetime generation (32-bit) - candidate |

## Sensors Provided

### Real-Time Power Sensors
- **PV1 Power** - Solar string 1 output (W)
- **PV2 Power** - Solar string 2 output (W)
- **Total PV Power** - Combined solar output (W)
- **Battery Power** - Battery charge/discharge (W, +/-)
- **Grid Power** - Grid import/export (W, +/-)
- **Load Power** - House consumption (W)

### Voltage & Frequency
- **Grid AC Voltage** - Main voltage (V)
- **Grid Frequency** - Grid frequency (Hz)
- **External PV AC Voltage** - External voltage reading (V)
- **Internal PV AC Voltage** - Internal voltage reading (V)

### Battery Status
- **Battery SOC** - State of Charge (%)
- **Battery Mode** - Text: Charging/Discharging/Full/etc.

### Calculated Sensors
- **Net Power Flow** - Overall grid import/export (W)
- **Battery Power Flow** - Simplified battery power indicator (W)
- **Self Consumption** - Percentage of solar used directly (%)

### Energy Integration Sensors
All accumulate energy over time for Home Assistant Energy Dashboard:

- **PV1 Energy** - Energy from string 1 (kWh)
- **PV2 Energy** - Energy from string 2 (kWh)
- **Total PV Energy** - Total solar generation (kWh)
- **Load Energy** - Total consumption (kWh)
- **Grid Import Energy** - Energy bought from grid (kWh)
- **Grid Export Energy** - Energy sold to grid (kWh)
- **Battery Charge Energy** - Energy stored (kWh)
- **Battery Discharge Energy** - Energy used from battery (kWh)

### Daily Counter
- **Daily Power Generation** - Today's solar generation (kWh, from inverter)

## Understanding Power Flow

### Positive/Negative Sign Convention

| Sensor | Positive (+) | Negative (-) |
|--------|-------------|-------------|
| **Battery Power** | Charging | Discharging |
| **Grid Power** | Importing (buying) | Exporting (selling) |

### Example Scenarios

**Scenario 1: Sunny Day, Low Load**
- PV Power: 4000W
- Load Power: 500W
- Battery Power: +2000W (charging)
- Grid Power: -1500W (exporting)
- Self Consumption: 50%

**Scenario 2: Evening, Battery Discharging**
- PV Power: 0W
- Load Power: 800W
- Battery Power: -800W (discharging)
- Grid Power: 0W
- Self Consumption: 0% (no solar)

**Scenario 3: Morning, Battery Full**
- PV Power: 2000W
- Load Power: 300W
- Battery Power: 0W (full)
- Grid Power: -1700W (exporting)
- Self Consumption: 15%

## Configuration

### Modbus Settings
- **Baud Rate**: 9600
- **Parity**: None
- **Stop Bits**: 1
- **Slave Address**: 0x01 (1)
- **Function Codes**: 
  - FC3 (Read Holding Registers) - Energy counters
  - FC4 (Read Input Registers) - Real-time values

### Update Interval
Default: 10 seconds (adjustable via dropdown: 5/10/15/30/60 seconds)

The update interval can be changed dynamically through Home Assistant without rebooting the ESP32.

## Home Assistant Integration

### Automatic Discovery
All sensors automatically appear in Home Assistant via the API.

## Troubleshooting

### No Data / All Sensors Show "Unknown"

1. **Check Wiring**: Verify RS485 A/B connections (try swapping if no data)
2. **Check Slave Address**: Ensure inverter Modbus address matches (default 0x01)
3. **Check Logs**: Look in ESPHome logs for Modbus errors

## Advanced: Adding Custom Registers

To add a new register:

```yaml
- platform: modbus_controller
  modbus_controller_id: sunhome
  id: my_new_sensor
  name: "My New Sensor"
  register_type: read        # or 'holding'
  address: 42                # 0-based address
  unit_of_measurement: "W"
  device_class: power
  state_class: measurement
  value_type: U_WORD         # or S_WORD, U_DWORD, etc.
  filters:
    - multiply: 0.1          # if scaling needed
  accuracy_decimals: 1
```

## Credits

Thanks to Wes Pope for the initial findings: [@wes314](https://github.com/wes314/ESY-Sunhome-MODBUS/tree/main) 

## Screenshot

![Example Screenshot](/screenshot.png)
