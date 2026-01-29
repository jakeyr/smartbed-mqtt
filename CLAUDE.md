# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Smartbed MQTT is a Home Assistant add-on that enables remote control of adjustable smart beds through MQTT integration. The project supports multiple bed manufacturers via different protocols:

- **Cloud-based**: Sleeptracker AI (Tempur/BeautyRest/Serta), ErgoWifi
- **Local WiFi**: ErgoMotion, Logicdata
- **Bluetooth Low Energy (BLE)**: Richmat, Linak, Solace, MotoSleep, Reverie, Leggett & Platt, Okimat, Keeson, Octo

## Development Commands

### Build and Test
```bash
# Build the project
yarn build

# Build for CI (clean + build)
yarn build:ci

# Clean build artifacts
yarn clean

# Run tests (watch mode)
yarn test

# Run tests (CI mode)
yarn test:ci

# Start development server
yarn start
```

### Code Quality
```bash
# Lint code
yarn lint

# Format code
yarn prettier

# Docker development
yarn docker:dev
```

## Architecture Overview

### Core Structure
- **Entry Point**: `src/index.ts` - Main application entry with bed type switching logic
- **Protocol Handlers**: Each bed manufacturer has its own directory (e.g., `Sleeptracker/`, `Richmat/`, `Linak/`)
- **Shared Infrastructure**:
  - `HomeAssistant/` - Home Assistant entity types (Button, Cover, Sensor, Switch, etc.)
  - `MQTT/` - MQTT connection and messaging
  - `Utils/` - Common utilities and helper functions
  - `ESPHome/` - Bluetooth proxy connectivity for BLE devices
  - `Common/` - Shared bed control functionality

### Key Design Patterns

1. **Type-Based Dispatch**: The main application switches on the `type` configuration to instantiate the appropriate bed handler
2. **Protocol Separation**: HTTP/UDP devices vs BLE devices follow different initialization paths
3. **Entity Abstraction**: Home Assistant entities are abstracted into reusable classes with consistent APIs
4. **Configuration-Driven**: All bed configurations are defined in `config.json` with strong typing

### Path Mapping
The project uses TypeScript path mapping defined in `tsconfig.json`:
- `@ha/*` maps to `HomeAssistant/*`
- `@mqtt/*` maps to `MQTT/*`
- `@utils/*` maps to `Utils/*`

### Testing
- Uses Jest with TypeScript support
- Test files follow the pattern `*.test.ts`
- Path mapping is configured for tests via `jest.config.ts`

### Home Assistant Integration
Each bed type creates Home Assistant entities via MQTT discovery:
- **Buttons**: Preset activation, programming, massage controls
- **Covers**: Motor position controls (head/feet/lumbar/etc.)
- **Sensors**: Bed angles, massage status, environmental data
- **Switches**: Lights, snore response, massage toggles
- **Selects**: Massage patterns, intensity levels

The entities are auto-discovered by Home Assistant through MQTT topics following the HA discovery format.

## Configuration
The main configuration is in `config.json` (Home Assistant add-on format) which includes:
- MQTT connection settings (auto-detected from HA)
- Bed type selection
- Device-specific configurations for each supported bed manufacturer
- BLE proxy settings for Bluetooth devices

## Sleeptracker AI Implementation Details

### Environmental Sensors Bug (FIXED)

**Issue**: The `productFeatures` flag is unreliable. Some Sleeptracker beds have BME680 environmental sensors but don't include the `'env_sensors'` flag in their `productFeatures` array. This caused environmental sensors to not appear in Home Assistant even though:
- The sensors physically exist in the bed
- The API returns valid sensor data
- The sensors work in the official app

**Example**: Model `sts60` (SleepTracker Gen2) has a BME680 multi-sensor that provides temperature, humidity, CO2, and VOC data, but `productFeatures` only contains `["api_flan_config_all", "motors"]`.

**Solution** (implemented in `src/Sleeptracker/sleeptracker.ts`):
```typescript
// Check for environmental sensors by looking for BME680 sensor in sensors array
// The productFeatures flag is unreliable - some beds have sensors without the flag
const hasEnvironmentSensors =
  helloData.productFeatures.includes('env_sensors') ||
  helloData.sensors?.some((s) => s.model === 'bme680_multi' || s.type === 'MULTI');
```

This checks both the flag AND the actual sensor hardware presence.

### Environmental Sensor Data

The BME680 multi-sensor provides:
- **Temperature** (`degreesCelsius`) - Celsius, converted by HA
- **Humidity** (`humidityPercentage`) - Percentage
- **CO2** (`co2Ppm`) - Parts per million
- **VOC** (`vocPpb`) - Volatile Organic Compounds in parts per billion

Data is retrieved via the `/latestEnvironmentSensorData` endpoint and updates based on the configured refresh frequency.

### Presence Detection (NOT AVAILABLE)

**Status**: Confirmed NOT available through the Sleeptracker API.

**Investigation findings**:
- The `leftSensor.status` field indicates sensor connectivity ("connected"/"not_available"), not occupancy
- The `sensorStatus` field in sleep sensor data is sensor health status (0 = connected), not presence
- All potential presence-related endpoints return 404:
  - `/presenceSensorData`
  - `/latestPresenceData`
  - `/currentStatus`
  - `/sleepSessionData`
- The field does not change when someone gets in/out of bed
- Historical sleep session data is not available in real-time

**Why the README states this**: The original developer thoroughly investigated the API and correctly determined that real-time occupancy detection is not provided by Sleeptracker's API.

### API Structure

**Authentication**:
- Uses Basic auth with email/password to obtain Bearer token
- Token has expiration time and is cached/refreshed automatically
- Different base URLs for different brands:
  - Tempur: `tsi.sleeptracker.com`
  - BeautyRest/Serta: `motionxlive.com`

**Key Endpoints**:
- `/v1/app/user/session` - Authentication
- `/processor/getByType` - Get devices
- `/processor/{id}/hello` - Get device capabilities and sensor info
- `/processor/getSensorMap` - Get sleep sensor mapping
- `/processor/latestEnvironmentSensorData` - Get environmental data
- `/processor/adjustableBaseControls` - Control bed motors/features

**Hello Data Structure**: Contains critical device information:
- `productFeatures[]` - Feature flags (unreliable for env sensors)
- `sensors[]` - Array of physical sensors attached to the bed
- `motorMeta.capabilities[]` - Motor and massage capabilities per side
- `leftSensor/rightSensor` - Sensor connectivity status

### Debugging Tips

**Environmental Sensors Not Showing**:
1. Check `helloData.sensors` array for a sensor with `model: "bme680_multi"` or `type: "MULTI"`
2. Call `/latestEnvironmentSensorData` endpoint directly to verify data is returned
3. Don't rely solely on `productFeatures.includes('env_sensors')`
4. Check sensor status fields: `status: "ACTIVE"` means sensor is working

**API Testing**:
- Debug scripts available (see README-DEBUGGING.md)
- Use `.env` file for credentials (see `.env.example`)
- Run with: `node --env-file=.env --import tsx debug-sleeptracker-api.ts`

**Common Issues**:
- 504 timeout errors are common - retry the request
- Sensor data updates every 1-15 minutes depending on `sleeptrackerRefreshFrequency`
- Empty environmental data may indicate sensors are calibrating (wait a few minutes)