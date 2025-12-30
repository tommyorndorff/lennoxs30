# Lennox S30 Component - Technical Architecture

This document provides a comprehensive technical overview of the Lennox S30 Home Assistant custom component architecture, implementation details, and data flow patterns.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Core Components](#core-components)
3. [Entity System](#entity-system)
4. [Device Management](#device-management)
5. [Communication Flow](#communication-flow)
6. [Bluetooth Low Energy Integration](#bluetooth-low-energy-integration)
7. [Configuration System](#configuration-system)
8. [Data Flow and State Management](#data-flow-and-state-management)
9. [Error Handling and Recovery](#error-handling-and-recovery)
10. [Testing Framework](#testing-framework)

## Architecture Overview

The Lennox S30 integration is built as a comprehensive Home Assistant custom component that provides both **cloud** and **local** connectivity to Lennox S30/E30/S40/M30 thermostats. It follows Home Assistant's modern integration patterns and is distributed through HACS (Home Assistant Community Store).

### Key Architectural Principles

- **Event-driven architecture**: Uses callback systems for real-time state synchronization
- **Multi-connection support**: Handles both cloud API and local IP connections simultaneously
- **Zone-based architecture**: Supports multi-zone HVAC systems with individual controls
- **Device-centric design**: Maps physical HVAC equipment to Home Assistant devices
- **Extensible entity system**: Modular entity creation based on available hardware features

## Core Components

### Manager Class (`__init__.py`)

The `Manager` class serves as the central coordinator for the entire integration:

```python
class Manager(object):
    """Manages the connection to cloud or local via API"""
```

**Key Responsibilities:**
- **Connection Management**: Maintains persistent connections to Lennox systems
- **API Coordination**: Interfaces with the `lennoxs30api` library for communication
- **State Synchronization**: Coordinates real-time updates between HA and Lennox systems
- **Polling Control**: Manages normal and fast polling intervals based on system activity
- **Device Discovery**: Discovers and maintains inventory of HVAC equipment
- **Error Recovery**: Handles connection failures, timeouts, and retry logic

**Critical Properties:**
- `api: s30api_async`: Core API client for Lennox communication
- `system_equip_device_map`: Maps systems and equipment to HA device entries
- `connected: bool`: Connection state for availability determination
- `is_metric: bool`: Unit system preference based on HA configuration

### Base Entity System (`base_entity.py`)

**S30BaseEntityMixin**: Foundation mixin for all entities in the integration

```python
class S30BaseEntityMixin:
    """Base Class."""
```

**Core Features:**
- **Availability Management**: Determines entity availability based on connection and cloud status
- **Callback Registration**: Manages update callbacks for real-time state changes
- **Connection State Handling**: Responds to connection state changes
- **Cloud Status Integration**: Integrates with Lennox cloud status for hybrid systems

**Availability Logic:**
```python
@property
def available(self) -> bool:
    """Determines if entity is available."""
    if self._manager.connected is False:
        return False
    if self.base_ignore_cloud_status is False and self._system.cloud_status == "offline":
        return False
    return super().available
```

### Constants and Configuration (`const.py`)

Centralized configuration and constants management:

- **Domain Configuration**: `LENNOX_DOMAIN = "lennoxs30"`
- **Unique ID Suffixes**: Standardized suffixes for entity identification
- **Default Values**: Timeout values, polling intervals, app IDs
- **Entity Categories**: Definitions for different entity types and purposes

## Entity System

The integration creates comprehensive Home Assistant entities organized by functionality:

### Climate Entities (`climate.py`)

**Primary HVAC Control Interface**

```python
class S30Climate(S30BaseEntityMixin, ClimateEntity):
    """Lennox S30 Climate Entity."""
```

**Features:**
- **HVAC Mode Control**: Heat, Cool, Auto, Emergency Heat, Off
- **Temperature Control**: Heating and cooling setpoints with deadband support
- **Fan Control**: Auto, On, Circulate modes
- **Preset Modes**: Away, Schedule Hold, Cancel Hold operations
- **Zone Management**: Individual control per zone in multi-zone systems

**Supported Climate Features:**
```python
SUPPORT_FLAGS = (
    ClimateEntityFeature.PRESET_MODE | 
    ClimateEntityFeature.FAN_MODE | 
    ClimateEntityFeature.TURN_OFF
)
```

### Sensor Entities

#### Core Sensors (`sensor.py`)

**S30TempSensor**: Zone temperature monitoring
```python
class S30TempSensor(S30BaseEntityMixin, SensorEntity):
    """Class for Lennox S30 thermostat temperature."""
```

**S30HumiditySensor**: Zone humidity monitoring
```python
class S30HumiditySensor(S30BaseEntityMixin, SensorEntity):
    """Class for Lennox S30 thermostat humidity."""
```

**S30DiagSensor**: Equipment diagnostic data
```python
class S30DiagSensor(S30BaseEntityMixin, SensorEntity):
    """Diagnostic Data Sensor"""
```

**Additional Sensors:**
- **S30OutdoorTempSensor**: External temperature readings
- **S30InverterPowerSensor**: Power consumption monitoring
- **S30AlertSensor**: System alert notifications
- **S30ActiveAlertsList**: Comprehensive alert listing

#### Specialized Sensors

**WiFi Sensors (`sensor_wifi.py`)**
- Signal strength monitoring
- Connection quality metrics

**Air Quality Sensors (`sensor_iaq.py`)**
- Indoor air quality monitoring
- Integration with BLE sensors

**Weather Sensors (`sensor_wt_env.py`)**
- Environmental monitoring
- Weather condition integration

### Binary Sensors (`binary_sensor.py`)

**Status and Connectivity Monitoring**

**S30InternetStatus**: Internet connectivity monitoring
```python
class S30InternetStatus(S30BaseEntityMixin, BinarySensorEntity):
    """Entity for S30 connected to internet."""
```

**Additional Binary Sensors:**
- **S30HomeStateBinarySensor**: Home/away detection
- **S30RelayServerStatus**: Relay server connectivity
- **S30CloudConnectedStatus**: Cloud service connection
- **Equipment Status Sensors**: Heat pump lockout, auxiliary heat status

### Control Entities

**Switches (`switch.py`)**
- Parameter safety switches
- Feature toggles
- Mode enablement controls

**Selects (`select.py`)**
- HVAC mode selection
- Ventilation mode controls
- Zone mode configuration

**Numbers (`number.py`)**
- Temperature setpoints
- Timing parameters
- Circulation settings

**Buttons (`button.py`)**
- System reset operations
- Parameter update triggers
- Maintenance actions

## Device Management

### Device Architecture (`device.py`)

The integration creates structured device representations for all HVAC equipment:

**Base Device Class**
```python
class Device:
    """Base class for all devices."""
```

**Specialized Device Types:**

**S30ControllerDevice**: Main thermostat controller
- Central hub for system control
- Primary user interface device

**S30IndoorUnit**: Indoor HVAC equipment
- Air handlers
- Furnaces
- Indoor coils

**S30OutdoorUnit**: Outdoor HVAC equipment
- Compressors
- Heat pumps
- Condensing units

**S30AuxiliaryUnit**: Auxiliary equipment
- Backup heating systems
- Supplemental equipment

**S30VentilationUnit**: Ventilation systems
- Fresh air systems
- Air purifiers
- Ventilation controllers

**S40BleDevice**: Bluetooth Low Energy devices
- Air quality sensors
- Environmental monitors

### Device Registration

Devices are automatically created and registered with Home Assistant's device registry:

```python
def create_devices(self, hass: HomeAssistant) -> None:
    """Create devices for the system."""
```

**Device Attributes:**
- **Identifiers**: Unique identification tuples
- **Manufacturer**: "Lennox" branding
- **Model Information**: Equipment model numbers
- **Hardware/Software Versions**: Firmware information
- **Device Relationships**: Parent-child device hierarchies

## Communication Flow

### Initialization Sequence

1. **Manager Initialization**: Create Manager instance with configuration parameters
2. **API Connection**: Establish connection to Lennox cloud or local API
3. **System Discovery**: Retrieve system configuration and equipment inventory
4. **Device Creation**: Generate HA device entries for discovered equipment
5. **Entity Registration**: Create and register entities based on available features
6. **Callback Setup**: Establish real-time update mechanisms

### Real-time Updates

**Callback System**: Event-driven state synchronization
```python
self._zone.registerOnUpdateCallback(self.update_callback, ["temperature", "humidity"])
```

**Update Propagation:**
1. Lennox system state change occurs
2. API library receives update notification
3. Registered callbacks are triggered
4. Entity schedules HA state update
5. Home Assistant reflects new state

### Polling Strategy

**Adaptive Polling**: Different intervals based on system activity

**Normal Polling**: Standard interval during idle periods
- Cloud connections: 10 seconds default
- Local connections: 1 second default

**Fast Polling**: Increased frequency during active changes
- Default: 0.75 seconds
- Triggered by user interactions or system state changes

## Bluetooth Low Energy Integration

### BLE Device Support

**Air Quality Sensors**: Specialized support for Lennox BLE sensors

**Model-Specific Implementations:**
- `ble_device_21p02.py`: 21P02 sensor model support
- `ble_device_22v25.py`: 22V25 sensor model support

**Sensor Capabilities:**
```python
lennox_21p02_sensors = [
    {
        "input_id": 4000,
        "name": "rssi",
        "state_class": SensorStateClass.MEASUREMENT,
        "device_class": SensorDeviceClass.SIGNAL_STRENGTH,
    },
    # Additional sensors...
]
```

**Air Quality Metrics:**
- **CO2 Monitoring**: Short-term and long-term CO2 levels
- **Particulate Matter**: PM2.5 and PM10 measurements
- **VOC Detection**: Volatile organic compound monitoring
- **Signal Quality**: RSSI and connectivity metrics

### BLE Sensor Integration (`sensor_ble.py`)

```python
class S30BleSensor(S30BaseEntityMixin, SensorEntity):
    """Lennox S30 BLE Sensor."""
```

**Features:**
- Dynamic sensor creation based on available BLE devices
- Support for multiple sensor types per device
- Integration with main HVAC system data

## Configuration System

### Config Flow (`config_flow.py`)

**User Interface Configuration**: Modern HA config flow implementation

**Configuration Steps:**
1. **Connection Type Selection**: Cloud vs. Local
2. **Credential Input**: Email/password or IP address
3. **Advanced Options**: Polling intervals, feature toggles
4. **Validation**: Connection testing and error handling

**Configuration Options:**
```python
CONFIG_SCHEMA = vol.Schema({
    DOMAIN: vol.Schema({
        vol.Required(CONF_EMAIL): cv.string,
        vol.Required(CONF_PASSWORD): cv.string,
        vol.Optional(CONF_SCAN_INTERVAL): cv.positive_int,
        # Additional options...
    })
})
```

### Migration Support

**YAML to UI Migration**: Automatic migration from legacy YAML configuration
```python
async def async_setup(hass: HomeAssistant, config: ConfigType):
    """Import config as config entry."""
```

## Data Flow and State Management

### State Synchronization

**Bidirectional Sync**: HA ↔ Lennox System state synchronization

**State Update Flow:**
1. User changes setting in Home Assistant
2. Entity calls appropriate API method
3. Manager processes request through lennoxs30api
4. Lennox system processes change
5. System broadcasts update
6. Integration receives update callback
7. Entity state is refreshed in HA

### Caching and Performance

**Efficient Updates**: Only changed entities are updated
**Callback Filtering**: Specific attribute monitoring to reduce unnecessary updates
**Batched Operations**: Group related API calls for efficiency

## Error Handling and Recovery

### Connection Recovery

**Automatic Reconnection**: Built-in retry logic for connection failures
```python
MAX_ERRORS = 2
RETRY_INTERVAL_SECONDS = 60
```

**Error States:**
- `DS_CONNECTING`: Initial connection attempt
- `DS_CONNECTED`: Normal operation
- `DS_DISCONNECTED`: Connection lost
- `DS_LOGIN_FAILED`: Authentication failure
- `DS_RETRY_WAIT`: Waiting for retry
- `DS_FAILED`: Permanent failure

### Exception Handling

**Graceful Degradation**: Entities become unavailable rather than causing crashes
**Logging**: Comprehensive logging for troubleshooting
**User Feedback**: Clear error messages in HA interface

## Testing Framework

### Test Structure

**Comprehensive Test Coverage**: Unit tests for all major components

**Test Organization:**
- `test_*.py`: Component-specific tests
- `conftest.py`: Shared test fixtures and utilities
- Mock data: JSON test data for API responses

### Test Patterns

**Entity Testing**: Verify entity behavior and state management
```python
@pytest.mark.asyncio
async def test_s30_base_entity_init(hass, manager: Manager):
```

**Manager Testing**: Connection and coordination logic
**Integration Testing**: End-to-end workflow validation
**Mock API Responses**: Realistic test data from actual systems

### Quality Assurance

**Code Quality**: Extensive linting and code analysis
**Coverage Metrics**: High test coverage for critical paths
**Regression Testing**: Prevent issues during updates

---

This technical documentation provides a comprehensive understanding of the Lennox S30 integration's architecture and implementation. For additional information, refer to the component source code and the Home Assistant developer documentation.