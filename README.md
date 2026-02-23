# blackroad-sensor-dashboard

> Real-time sensor data dashboard and alerting — part of the BlackRoad Hardware platform.

## Features

- **Multi-Sensor Support** — Temperature, humidity, pressure, CO₂, light, motion, power
- **Calibration Offsets** — Per-sensor calibration adjustments
- **Z-Score Anomaly Detection** — Statistical spike detection over configurable windows
- **Threshold Alerting** — Min/max threshold breach alerts with severity levels
- **Dashboard View** — Aggregated real-time summary of all sensors
- **Export** — JSON or CSV time-series export

## Quick Start

```bash
pip install -r requirements.txt
python sensor_dashboard.py dashboard
python sensor_dashboard.py alerts
```

## Usage

```python
from sensor_dashboard import SensorDashboard

dash = SensorDashboard()

sensor = dash.add_sensor("device-001", "temperature", "°C",
                          min_value=0.0, max_value=50.0,
                          calibration_offset=0.5, name="Room Temp")

dash.log_reading(sensor.id, 22.5)
current = dash.get_current(sensor.id)

# Detect anomaly (z-score)
alert = dash.detect_anomaly(sensor.id, window=60, z_threshold=2.5)

# Get dashboard data
data = dash.get_dashboard_data()

# Export as CSV
csv_data = dash.export_timeseries(sensor.id, hours=24, fmt="csv")
```

## Alert Types

| Type | Description |
|------|-------------|
| `threshold` | Value outside min/max bounds |
| `anomaly` | Statistical outlier (Z-score) |
| `offline` | Sensor not reporting |

## Testing

```bash
pytest --tb=short -v
```

## License

Proprietary — BlackRoad OS, Inc. All rights reserved.
