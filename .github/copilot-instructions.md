# GitHub Copilot Instructions for Lennox S30 Home Assistant Custom Component

## Project Overview
This is a Home Assistant custom component for Lennox S30 thermostats. It provides integration with Home Assistant to control and monitor Lennox S30 HVAC systems through various entity types.

## Code Structure and Conventions

### Entity Types
- **Climate**: Main thermostat control (`climate.py`)
- **Sensors**: Temperature, humidity, diagnostics, alerts (`sensor_*.py`)
- **Binary Sensors**: Status indicators, connectivity (`binary_sensor*.py`)
- **Switches**: Feature toggles, modes (`switch_*.py`, `*_switch.py`)
- **Selects**: Dropdown options for modes (`select_*.py`, `*_select.py`)
- **Numbers**: Numeric inputs for setpoints (`number_*.py`)
- **Buttons**: Action triggers (`button*.py`)

### Key Files
- `__init__.py`: Integration setup and coordinator
- `const.py`: Constants, entity types, and configuration
- `config_flow.py`: Configuration UI flow
- `base_entity.py`: Base class for all entities
- `device.py`: Device representation
- `manager.py`: Core system management

### Coding Standards
- Follow Home Assistant development guidelines
- Use `async`/`await` for all I/O operations
- Inherit from appropriate HA base classes (`CoordinatorEntity`, `ClimateEntity`, etc.)
- Implement proper `unique_id`, `name`, and `device_info` properties
- Use `@property` decorators for entity attributes
- Handle exceptions gracefully with proper logging

### Home Assistant Patterns
- Use `DataUpdateCoordinator` for data fetching
- Implement `async_setup_entry` and `async_unload_entry`
- Use `hass.data[DOMAIN]` for storing integration data
- Follow entity naming conventions: `{device_name} {entity_description}`
- Use device registry for device management
- Implement proper availability checking

### Testing Patterns
- Test files follow `test_{module_name}.py` convention
- Use pytest fixtures from `conftest.py`
- Mock external API calls and hardware interactions
- Test entity state changes and attribute updates
- Verify proper setup and teardown

### Configuration
- Support both UI and YAML configuration where appropriate
- Validate user inputs in config flow
- Provide helpful error messages
- Support discovery when possible

### Error Handling
- Use Home Assistant's logging system
- Catch and handle API timeouts gracefully
- Provide meaningful error messages to users
- Implement retry logic for transient failures

### Entity Attributes
- Expose relevant device information as attributes
- Follow Home Assistant attribute naming conventions
- Include diagnostic information when helpful
- Maintain consistency across similar entity types

## BLE (Bluetooth Low Energy) Integration
- `ble_device_*.py` files handle Bluetooth communication
- Separate handling for different device models (21p02, 22v25)
- Include proper device discovery and connection management

## When Suggesting Code
- Always consider Home Assistant's entity lifecycle
- Include proper imports from `homeassistant.components`
- Use type hints consistently
- Consider backwards compatibility with older HA versions
- Follow the existing code style and patterns in the repository