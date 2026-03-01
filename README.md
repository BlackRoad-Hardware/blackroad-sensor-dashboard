# blackroad-sensor-dashboard

[![License: Proprietary](https://img.shields.io/badge/license-Proprietary-red.svg)](LICENSE)
[![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/)
[![pytest](https://img.shields.io/badge/tested%20with-pytest-brightgreen.svg)](https://pytest.org/)
[![Production Ready](https://img.shields.io/badge/status-production-success.svg)](#)

> **BlackRoad Sensor Dashboard** — Real-time sensor monitoring, anomaly detection, and threshold alerting for the BlackRoad Hardware platform. Built for production deployments at scale.

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Requirements](#requirements)
4. [Installation](#installation)
5. [Quick Start](#quick-start)
6. [API Reference](#api-reference)
   - [SensorDashboard](#sensordashboard)
   - [Sensor Management](#sensor-management)
   - [Reading Management](#reading-management)
   - [Anomaly Detection](#anomaly-detection)
   - [Threshold Alerting](#threshold-alerting)
   - [Dashboard & Export](#dashboard--export)
7. [Data Models](#data-models)
8. [Sensor Types & Default Thresholds](#sensor-types--default-thresholds)
9. [Alert Reference](#alert-reference)
10. [CLI Usage](#cli-usage)
11. [Testing](#testing)
12. [Stripe Integration](#stripe-integration)
13. [npm / JavaScript Clients](#npm--javascript-clients)
14. [Security](#security)
15. [License](#license)

---

## Overview

**blackroad-sensor-dashboard** is the core telemetry engine of the [BlackRoad Hardware](https://blackroadhardware.com) platform. It provides a unified Python library and CLI for ingesting multi-sensor readings, applying per-device calibration, detecting statistical anomalies, firing threshold-breach alerts, and exporting time-series data in JSON or CSV format.

The library uses an embedded SQLite datastore (WAL mode, foreign-key constraints, and indexed queries) so it runs anywhere — edge devices, cloud VMs, or CI pipelines — with zero external service dependencies.

---

## Features

| Capability | Details |
|---|---|
| **Multi-Sensor Support** | Temperature, humidity, pressure, CO₂, light, motion, power |
| **Calibration Offsets** | Per-sensor, persistent offset correction applied to every reading |
| **Z-Score Anomaly Detection** | Configurable sliding-window statistical spike detection |
| **Threshold Alerting** | Min/max breach alerts with INFO / WARNING / CRITICAL severity |
| **Dashboard View** | Aggregated real-time summary across all registered sensors |
| **Batch Ingestion** | Log multiple readings in a single call |
| **Time-Series Export** | JSON or CSV export with configurable look-back window |
| **Alert Lifecycle** | Create, query, and resolve alerts; active/historical views |
| **Offline Detection** | Explicit sensor-offline alert generation |
| **Embedded SQLite** | No external database required; WAL mode for concurrent access |

---

## Requirements

- Python **3.9** or later
- `pytest >= 7.0` *(test suite only)*
- `pytest-cov >= 4.0` *(coverage reporting only)*

No third-party runtime dependencies are required beyond the Python standard library.

---

## Installation

### From source

```bash
git clone https://github.com/BlackRoad-Hardware/blackroad-sensor-dashboard.git
cd blackroad-sensor-dashboard
pip install -r requirements.txt
```

### In a virtual environment (recommended)

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

---

## Quick Start

```bash
# Show live dashboard JSON
python sensor_dashboard.py dashboard

# Show active alerts JSON
python sensor_dashboard.py alerts
```

```python
from sensor_dashboard import SensorDashboard

dash = SensorDashboard()

# Register a sensor
sensor = dash.add_sensor(
    device_id="device-001",
    sensor_type="temperature",
    unit="°C",
    min_value=0.0,
    max_value=50.0,
    calibration_offset=0.5,
    name="Room Temp",
    location="Office A",
)

# Log a reading
reading = dash.log_reading(sensor.id, 22.5)

# Fetch the latest reading
current = dash.get_current(sensor.id)

# Run anomaly detection (Z-score over the last 60 minutes)
alert = dash.detect_anomaly(sensor.id, window=60, z_threshold=2.5)

# Get the full dashboard payload
data = dash.get_dashboard_data()

# Export 24 h of readings as CSV
csv_data = dash.export_timeseries(sensor.id, hours=24, fmt="csv")
```

---

## API Reference

### SensorDashboard

```python
SensorDashboard(db_path: Path = Path("sensor_dashboard.db"))
```

Main entry point. Initialises the SQLite database and all required tables on first use.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `db_path` | `Path` | `sensor_dashboard.db` | Path to the SQLite database file |

---

### Sensor Management

#### `add_sensor`

```python
add_sensor(
    device_id: str,
    sensor_type: str,          # see Sensor Types below
    unit: str,
    min_value: float = None,   # defaults to type-specific lower bound
    max_value: float = None,   # defaults to type-specific upper bound
    calibration_offset: float = 0.0,
    name: str = None,
    location: str = None,
) -> Sensor
```

Registers a new sensor and returns a `Sensor` dataclass.

#### `get_sensor`

```python
get_sensor(sensor_id: str) -> Sensor
```

Retrieves a sensor by its UUID. Raises `ValueError` if not found.

#### `list_sensors`

```python
list_sensors(device_id: str = None) -> List[Sensor]
```

Returns all sensors, optionally filtered by `device_id`.

#### `update_calibration`

```python
update_calibration(sensor_id: str, offset: float) -> Sensor
```

Persists a new calibration offset for the sensor.

---

### Reading Management

#### `log_reading`

```python
log_reading(sensor_id: str, value: float, quality: str = "good") -> Reading
```

Stores a raw reading, applies the calibration offset, and automatically triggers threshold checking. `quality` must be one of `good`, `degraded`, or `error`.

#### `get_current`

```python
get_current(sensor_id: str) -> Optional[Reading]
```

Returns the most recent reading for the sensor, or `None` if no readings exist.

#### `get_history`

```python
get_history(sensor_id: str, hours: int = 24, quality_filter: str = None) -> List[Reading]
```

Returns all readings within the given look-back window, ordered by timestamp ascending.

#### `get_stats`

```python
get_stats(sensor_id: str, hours: int = 24) -> Dict
```

Returns descriptive statistics over the look-back window:

```json
{
  "count": 120,
  "min": 19.8,
  "max": 24.3,
  "avg": 22.1,
  "stddev": 0.9
}
```

#### `batch_log`

```python
batch_log(entries: List[Dict]) -> List[Reading]
```

Logs multiple readings in one call. Each entry must contain `sensor_id` and `value`; `quality` is optional.

```python
dash.batch_log([
    {"sensor_id": s.id, "value": 21.0},
    {"sensor_id": s.id, "value": 21.5, "quality": "degraded"},
])
```

---

### Anomaly Detection

#### `detect_anomaly`

```python
detect_anomaly(sensor_id: str, window: int = 60, z_threshold: float = 2.5) -> Optional[Alert]
```

Computes the Z-score of the latest calibrated reading against all non-error readings within the rolling `window` (minutes). Returns a `WARNING` alert when `z > z_threshold` and a `CRITICAL` alert when `z > z_threshold * 1.5`. Returns `None` if fewer than 5 readings exist in the window or no anomaly is detected.

---

### Threshold Alerting

#### `check_thresholds`

```python
check_thresholds(sensor_id: str) -> Optional[Alert]
```

Compares the latest calibrated reading against the sensor's configured `min_value` / `max_value`. Called automatically by `log_reading`. Returns a `CRITICAL` alert on breach, or `None` if within bounds.

#### `resolve_alert`

```python
resolve_alert(alert_id: str) -> Alert
```

Marks an alert as resolved by setting `resolved_at` to the current UTC timestamp.

#### `get_active_alerts`

```python
get_active_alerts() -> List[Alert]
```

Returns all unresolved alerts, ordered by `triggered_at` descending.

#### `get_all_alerts`

```python
get_all_alerts() -> List[Alert]
```

Returns the complete alert history, ordered by `triggered_at` descending.

#### `mark_sensor_offline`

```python
mark_sensor_offline(sensor_id: str) -> Alert
```

Fires an `OFFLINE / WARNING` alert for the specified sensor.

---

### Dashboard & Export

#### `get_dashboard_data`

```python
get_dashboard_data() -> Dict
```

Returns a full snapshot of the platform:

```json
{
  "generated_at": "2026-03-01T00:00:00+00:00",
  "total_sensors": 4,
  "active_alerts": 1,
  "critical_alerts": 0,
  "sensors": [ ... ],
  "alerts": [ ... ]
}
```

#### `export_timeseries`

```python
export_timeseries(sensor_id: str, hours: int = 24, fmt: str = "json") -> str
```

Exports readings as a JSON string (`fmt="json"`) or CSV string (`fmt="csv"`).

#### `export_alerts`

```python
export_alerts(active_only: bool = True) -> str
```

Exports alerts as a JSON string. Pass `active_only=False` for the full alert history.

---

## Data Models

### `Sensor`

| Field | Type | Description |
|---|---|---|
| `id` | `str` (UUID) | Unique sensor identifier |
| `device_id` | `str` | Parent device identifier |
| `type` | `SensorType` | Sensor category |
| `unit` | `str` | Measurement unit (e.g. `°C`, `%`) |
| `min_value` | `float` | Lower threshold bound |
| `max_value` | `float` | Upper threshold bound |
| `calibration_offset` | `float` | Applied offset (default `0.0`) |
| `name` | `str \| None` | Human-readable label |
| `location` | `str \| None` | Physical location label |

### `Reading`

| Field | Type | Description |
|---|---|---|
| `id` | `str` (UUID) | Unique reading identifier |
| `sensor_id` | `str` | Parent sensor UUID |
| `raw_value` | `float` | Value as received |
| `calibrated_value` | `float` | `raw_value + calibration_offset` |
| `timestamp` | `str` | ISO 8601 UTC timestamp |
| `quality` | `ReadingQuality` | `good` / `degraded` / `error` |

### `Alert`

| Field | Type | Description |
|---|---|---|
| `id` | `str` (UUID) | Unique alert identifier |
| `sensor_id` | `str` | Triggering sensor UUID |
| `type` | `AlertType` | `threshold` / `anomaly` / `offline` |
| `severity` | `AlertSeverity` | `info` / `warning` / `critical` |
| `message` | `str` | Human-readable description |
| `triggered_at` | `str` | ISO 8601 UTC timestamp |
| `resolved_at` | `str \| None` | ISO 8601 UTC timestamp, or `None` if still active |
| `is_active` | `bool` (property) | `True` when `resolved_at is None` |

---

## Sensor Types & Default Thresholds

| Type constant | String key | Default min | Default max |
|---|---|---|---|
| `SensorType.TEMPERATURE` | `"temperature"` | −40.0 | 85.0 |
| `SensorType.HUMIDITY` | `"humidity"` | 0.0 | 100.0 |
| `SensorType.PRESSURE` | `"pressure"` | 870.0 | 1 084.0 |
| `SensorType.CO2` | `"co2"` | 400.0 | 5 000.0 |
| `SensorType.LIGHT` | `"light"` | 0.0 | 100 000.0 |
| `SensorType.MOTION` | `"motion"` | 0.0 | 1.0 |
| `SensorType.POWER` | `"power"` | 0.0 | 10 000.0 |

Pass explicit `min_value` / `max_value` to `add_sensor` to override defaults.

---

## Alert Reference

### Alert Types

| Type | Trigger |
|---|---|
| `threshold` | Calibrated value is outside `[min_value, max_value]` |
| `anomaly` | Z-score exceeds configured threshold in rolling window |
| `offline` | Explicitly signalled via `mark_sensor_offline()` |

### Alert Severities

| Severity | When |
|---|---|
| `info` | Informational — no immediate action required |
| `warning` | Attention recommended; anomaly Z-score in the `(z_threshold, z_threshold × 1.5]` range |
| `critical` | Immediate action required; threshold breach or Z-score > `z_threshold × 1.5` |

---

## CLI Usage

```
python sensor_dashboard.py <command>

Commands:
  dashboard   Print full dashboard JSON to stdout
  alerts      Print active alerts JSON to stdout
```

**Example — pipe to jq:**

```bash
python sensor_dashboard.py dashboard | jq '.sensors[].current_value'
```

---

## Testing

The test suite uses [pytest](https://pytest.org/) and runs against a temporary in-memory database so no persistent state is created.

```bash
# Run all tests
pytest --tb=short -v

# Run with coverage report
pytest --cov=sensor_dashboard --cov-report=term-missing
```

All tests are in `test_sensor_dashboard.py` and cover:

- Sensor registration and custom range configuration
- Reading ingestion with and without calibration offsets
- History retrieval and descriptive statistics
- Threshold alert generation (high and low)
- Z-score anomaly detection (spike and normal)
- Alert resolution lifecycle
- Dashboard data aggregation
- JSON and CSV time-series export
- Batch ingestion
- Offline alert generation
- Sensor calibration updates
- Device-scoped sensor listing

---

## Stripe Integration

**blackroad-sensor-dashboard** is part of the broader BlackRoad Hardware platform, which uses [Stripe](https://stripe.com) for subscription billing and metered API usage.

### Billing model

| Plan | Included sensors | Overage |
|---|---|---|
| Starter | Up to 10 sensors | — |
| Pro | Up to 100 sensors | $0.05 / additional sensor / month |
| Enterprise | Unlimited | Custom |

### Integration points

- **Webhook endpoint** — Your backend receives `invoice.payment_succeeded` and `customer.subscription.updated` events to provision or de-provision sensor quotas.
- **Metered billing** — Each call to `log_reading` can be counted as a usage record and reported to Stripe's metered billing API via `stripe.SubscriptionItem.create_usage_record()`.
- **Customer portal** — Redirect customers to `stripe.billing_portal.Session.create()` for self-serve plan management.

Refer to the [Stripe Python SDK documentation](https://stripe.com/docs/api?lang=python) and the BlackRoad platform integration guide (available in the internal developer portal) for implementation details.

---

## npm / JavaScript Clients

A companion JavaScript/TypeScript client for consuming the BlackRoad Sensor Dashboard HTTP API is published to npm:

```bash
npm install @blackroad/sensor-dashboard-client
```

### Basic usage

```typescript
import { SensorDashboardClient } from "@blackroad/sensor-dashboard-client";

const client = new SensorDashboardClient({ baseUrl: "https://api.blackroadhardware.com" });

const dashboard = await client.getDashboard();
console.log(dashboard.totalSensors, dashboard.activeAlerts);
```

The npm package is independently versioned and maintained in the `blackroad-sensor-dashboard-client` repository. See its README for full API coverage, authentication, and WebSocket streaming support.

---

## Security

- All database access is parameterised; no raw SQL string interpolation is used.
- SQLite foreign-key constraints are enforced (`PRAGMA foreign_keys = ON`).
- WAL journal mode prevents read/write contention.
- The `db_path` parameter lets you place the database on an encrypted volume or a path with restricted OS permissions.
- No credentials, API keys, or secrets are stored by this library.

---

## License

Proprietary — BlackRoad OS, Inc. All rights reserved.

Copyright © 2024–2026 BlackRoad OS, Inc.  
Founder, CEO & Sole Stockholder: Alexa Louise Amundson

Unauthorised copying, distribution, or modification of this software is strictly prohibited. See [LICENSE](LICENSE) for full terms.
