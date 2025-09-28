# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### FireTV Adapter Specific Context

This adapter provides control and monitoring capabilities for Amazon Fire TV devices and other Android TV-based devices. Key functionality includes:

- **Device Communication**: Uses ADB (Android Debug Bridge) to communicate with Fire TV devices over network
- **Device Discovery**: Automatic discovery of Fire TV devices on the local network using mDNS
- **Remote Control**: Send key commands (play, pause, navigation, etc.) to Fire TV devices
- **State Monitoring**: Monitor power state, currently playing audio/video, and other device states
- **App Management**: Launch and control applications on Fire TV devices

**Key Dependencies**:
- `adbkit`: Node.js client for Android Debug Bridge protocol
- `mdns-discovery`: Network device discovery via multicast DNS
- `bluebird`: Promise library for async operations

**Device Requirements**:
- Fire TV devices must have ADB debugging enabled
- Network connectivity between ioBroker and Fire TV devices
- Proper firewall configuration to allow ADB traffic (port 5555)

**Configuration Parameters**:
- `adbPath`: Path to ADB executable (auto-detected or manually configured)
- `devices`: Array of Fire TV devices with IP addresses and names
- `discoveryEnabled`: Enable/disable automatic device discovery

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_DEVICE_IP = '192.168.1.100'; // Mock Fire TV IP
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test FireTV adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.firetv.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            adbPath: '/usr/bin/adb',
                            devices: [{
                                name: 'Test FireTV',
                                ip: TEST_DEVICE_IP,
                                enabled: true
                            }],
                            discoveryEnabled: false, // Disable discovery in tests
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Get all states created by adapter
                        const stateIds = await harness.dbConnection.getStateIDs('firetv.0.*');
                        
                        console.log(`📊 Found ${stateIds.length} states`);

                        if (stateIds.length > 0) {
                            console.log('✅ Adapter successfully created states');
                            
                            // Show sample of created states
                            const allStates = await new Promise((res, rej) => {
                                harness.states.getStates(stateIds, (err, states) => {
                                    if (err) return rej(err);
                                    res(states);
                                });
                            });

                            // Log some example states for verification
                            const sampleStates = stateIds.slice(0, 5);
                            for (const id of sampleStates) {
                                if (allStates[id]) {
                                    console.log(`📌 ${id}: ${JSON.stringify(allStates[id].val)}`);
                                }
                            }
                        }

                        resolve(true);
                    } catch (error) {
                        console.log('🔍 Step 4: Checking if failure was expected...');
                        
                        // Check if this is an expected failure (e.g., no ADB binary found)
                        if (error.message.includes('adb') || error.message.includes('ADB')) {
                            console.log('✅ Adapter correctly handled missing ADB dependency');
                            resolve(true);
                        } else {
                            console.error('❌ Unexpected error:', error);
                            reject(error);
                        }
                    }
                });
            }).timeout(30000);
        });
    }
});
```

#### FireTV-Specific Testing Considerations

When testing the FireTV adapter:

1. **Mock ADB Dependencies**: Since ADB may not be available in CI environments, handle missing dependencies gracefully
2. **Network Isolation**: Tests should not require actual Fire TV devices on the network
3. **State Validation**: Verify that device states (power, audio playing, etc.) are created correctly
4. **Error Handling**: Test scenarios where devices are unreachable or ADB fails
5. **Discovery Testing**: Verify mDNS discovery works correctly (when enabled)

Example states to verify:
- `firetv.0.[device].power` - Device power state
- `firetv.0.[device].audioPlaying` - Audio playing state
- `firetv.0.[device].androidVersion` - Android version
- `firetv.0.[device].apiLevel` - Android API level

#### Key Integration Testing Rules

1. **NEVER test real devices directly** - Let the adapter handle device communication
2. **ALWAYS use the harness** - `getHarness()` provides the testing environment  
3. **Configure via objects** - Use `harness.objects.setObject()` to set adapter configuration
4. **Start properly** - Use `harness.startAdapterAndWait()` to start the adapter
5. **Check states** - Use `harness.states.getState()` to verify results
6. **Use timeouts** - Allow time for async operations with appropriate timeouts
7. **Test real workflow** - Initialize → Configure → Start → Verify States

#### Workflow Dependencies
Integration tests should run ONLY after lint and adapter tests pass:

```yaml
# In your GitHub workflow
test-integration:
  needs: [check-and-lint, adapter-tests]
  runs-on: ubuntu-latest
  steps:
    - name: Run integration tests
      run: npm run test:integration
```

## Adapter Architecture

### Core Components

#### Main Adapter File (`firetv.js`)
- **Entry Point**: Main adapter logic and state machine
- **Device Management**: Handles multiple Fire TV devices
- **ADB Client**: Manages ADB connections and commands
- **State Updates**: Updates ioBroker states based on device status

#### Key Classes and Methods

```javascript
// FireTV device class structure
class FireTV {
  constructor(adapter, deviceConfig) {
    this.adapter = adapter;
    this.ip = deviceConfig.ip;
    this.name = deviceConfig.name;
    this.client = null; // ADB client instance
  }

  async connect() {
    // Establish ADB connection to device
  }

  async sendKey(keyCode) {
    // Send key command to device
  }

  async updatePowerState() {
    // Check and update device power state
  }

  async updateAudioPlaying() {
    // Check if audio/video is playing
  }
}
```

#### Configuration Handling
The adapter normalizes and validates configuration:

```javascript
function normalizeConfig() {
    // Auto-detect ADB path if not provided
    if (!soef.existFile(adapter.config.adbPath)) {
        adapter.config.adbPath = checkPATH(); // Search system PATH
    }
    
    // Validate device configurations
    adapter.config.devices = adapter.config.devices.filter(device => 
        device.ip && device.name
    );
}
```

### State Management

#### Device States Structure
Each Fire TV device creates the following state hierarchy:
```
firetv.0.[deviceName].power          // boolean - device power state
firetv.0.[deviceName].audioPlaying   // boolean - audio playing state  
firetv.0.[deviceName].androidVersion // string - Android version
firetv.0.[deviceName].apiLevel       // number - Android API level
firetv.0.[deviceName].sendKey        // string - command state for sending keys
firetv.0.[deviceName].launchApp      // string - command state for launching apps
```

#### State Definitions
When creating states, follow ioBroker conventions:

```javascript
const usedStateNames = {
    power: {
        n: 'power',
        type: 'boolean',
        role: 'switch.power',
        read: true,
        write: true
    },
    audioPlaying: {
        n: 'audioPlaying', 
        type: 'boolean',
        role: 'indicator',
        read: true,
        write: false
    },
    sendKey: {
        n: 'sendKey',
        type: 'string', 
        role: 'text',
        read: true,
        write: true
    }
};
```

## Code Patterns & Best Practices

### Error Handling
```javascript
// Always handle ADB errors gracefully
try {
    await this.client.shell(this.client.id, command);
} catch (error) {
    this.adapter.log.error(`ADB command failed: ${error.message}`);
    // Update connection state to indicate failure
    this.adapter.setState(`${this.name}.connected`, false, true);
}
```

### Async Operations
```javascript
// Use proper async/await patterns
async function updateDeviceStates() {
    const devices = adapter.config.devices || [];
    
    for (const deviceConfig of devices) {
        try {
            const device = new FireTV(adapter, deviceConfig);
            await device.connect();
            await device.updateAllStates();
        } catch (error) {
            adapter.log.warn(`Failed to update ${deviceConfig.name}: ${error.message}`);
        }
    }
}
```

### Resource Cleanup
```javascript
// Always clean up ADB connections
FireTV.prototype.close = function () {
    if (this.client && this.client.id) {
        this.client.disconnect(this.client.id).catch(err => {
            adapter.log.debug(`Disconnect error: ${err.message}`);
        });
        
        this.client.kill(err => {
            if (err) adapter.log.error(`Failed to kill ADB server: ${err.message}`);
        });
    }
};
```

### Configuration Validation
```javascript
// Validate configuration before use
function validateConfig() {
    if (!adapter.config.devices || !Array.isArray(adapter.config.devices)) {
        adapter.log.error('No devices configured');
        return false;
    }
    
    const validDevices = adapter.config.devices.filter(device => {
        if (!device.ip) {
            adapter.log.warn(`Device ${device.name} missing IP address`);
            return false;
        }
        return true;
    });
    
    adapter.config.devices = validDevices;
    return validDevices.length > 0;
}
```

## Development Guidelines

### Fire TV Specific Considerations

1. **ADB Path Detection**: The adapter should automatically detect ADB installation or provide clear error messages
2. **Network Discovery**: Use mDNS to discover Fire TV devices automatically when possible
3. **Key Codes**: Use standard Android key codes for remote control functionality
4. **State Polling**: Implement efficient polling for device states without overwhelming the device
5. **Connection Management**: Handle network connectivity issues and device reboots gracefully

### Code Style
- Follow existing code patterns in the repository
- Use meaningful variable and function names
- Add appropriate logging for debugging
- Handle edge cases and error conditions
- Document complex ADB interactions

### Security Considerations
- Validate all user inputs (IP addresses, device names)
- Handle ADB security prompts appropriately  
- Provide clear documentation about enabling ADB debugging
- Consider network security implications of ADB over network

### Performance Guidelines
- Minimize ADB command frequency to avoid device overload
- Use connection pooling for multiple devices
- Implement proper backoff strategies for failed connections
- Cache device information where appropriate

## Common Issues & Troubleshooting

### ADB Connectivity
- Ensure Fire TV has "ADB debugging" enabled in Developer Options
- Verify network connectivity between ioBroker and Fire TV
- Check firewall settings for ADB port (5555)
- Handle ADB authentication dialogs

### Device Discovery
- mDNS discovery may not work on all networks
- Provide manual device configuration as fallback
- Handle network segmentation issues

### State Synchronization
- Device states may not update immediately
- Implement proper state caching and refresh intervals
- Handle device sleep/wake state changes

## Testing Checklist

### Unit Tests
- [ ] Configuration parsing and validation
- [ ] ADB command generation
- [ ] State creation and updates
- [ ] Error handling scenarios
- [ ] Device discovery logic

### Integration Tests  
- [ ] Adapter startup with valid configuration
- [ ] Device connection and communication
- [ ] State creation and updates
- [ ] Error recovery scenarios
- [ ] Resource cleanup on shutdown

### Manual Testing
- [ ] Device discovery works correctly
- [ ] Remote control commands function properly
- [ ] State updates reflect device status accurately
- [ ] Configuration changes apply correctly
- [ ] Logs provide helpful debugging information

This file helps GitHub Copilot understand the context and patterns specific to ioBroker Fire TV adapter development, enabling more accurate and helpful code suggestions.