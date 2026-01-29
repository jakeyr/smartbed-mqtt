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