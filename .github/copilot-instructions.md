# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.2
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

#### Testing Both Success AND Failure Scenarios

**IMPORTANT**: For every "it works" test, implement corresponding "it doesn't work and fails" tests. This ensures proper error handling and validates that your adapter fails gracefully when expected.

```javascript
// Example: Testing successful configuration
it('should configure and start adapter with valid configuration', function () {
    return new Promise(async (resolve, reject) => {
        // ... successful configuration test as shown above
    });
}).timeout(40000);

// Example: Testing failure scenarios
it('should NOT create daily states when daily is disabled', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            
            console.log('🔍 Step 1: Fetching adapter object...');
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.firetv.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            if (!obj) return reject(new Error('Adapter object not found'));
            console.log('✅ Step 1.5: Adapter object loaded');

            console.log('🔍 Step 2: Updating adapter config...');
            Object.assign(obj.native, {
                adbPath: '/usr/bin/adb',
                devices: [{
                    name: 'Test FireTV',
                    ip: '192.168.1.100',
                    enabled: false // Device disabled for this test
                }],
                discoveryEnabled: false,
            });

            await new Promise((res, rej) => {
                harness.objects.setObject(obj._id, obj, (err) => {
                    if (err) return rej(err);
                    console.log('✅ Step 2.5: Adapter object updated');
                    res(undefined);
                });
            });

            console.log('🔍 Step 3: Starting adapter...');
            await harness.startAdapterAndWait();
            console.log('✅ Step 4: Adapter started');

            console.log('⏳ Step 5: Waiting for adapter to process...');
            await new Promise((res) => setTimeout(res, 15000));

            console.log('🔍 Step 6: Fetching state IDs...');
            const stateIds = await harness.dbConnection.getStateIDs('firetv.0.*');

            console.log(`📊 Step 7: Found ${stateIds.length} total states`);

            // Check that disabled device states are not created
            const disabledDeviceStates = stateIds.filter((key) => key.includes('Test FireTV'));
            if (disabledDeviceStates.length === 0) {
                console.log(`✅ Step 8: No states found for disabled device as expected`);
            } else {
                console.log(`❌ Step 8: States present for disabled device (${disabledDeviceStates.length}) (test failed)`);
                return reject(new Error('Expected no states for disabled device but found some'));
            }

            await harness.stopAdapter();
            console.log('🛑 Step 9: Adapter stopped');

            resolve(true);
        } catch (error) {
            reject(error);
        }
    });
}).timeout(40000);

// Example: Testing missing required configuration  
it('should handle missing required configuration properly', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            
            console.log('🔍 Step 1: Fetching adapter object...');
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.firetv.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            if (!obj) return reject(new Error('Adapter object not found'));

            console.log('🔍 Step 2: Removing required configuration...');
            // Remove required configuration to test failure handling
            delete obj.native.adbPath; // This should cause failure or graceful handling

            await new Promise((res, rej) => {
                harness.objects.setObject(obj._id, obj, (err) => {
                    if (err) return rej(err);
                    res(undefined);
                });
            });

            console.log('🔍 Step 3: Starting adapter...');
            await harness.startAdapterAndWait();

            console.log('⏳ Step 4: Waiting for adapter to process...');
            await new Promise((res) => setTimeout(res, 10000));

            console.log('🔍 Step 5: Checking adapter behavior...');
            const stateIds = await harness.dbConnection.getStateIDs('firetv.0.*');

            // Check if adapter handled missing configuration gracefully
            if (stateIds.length === 0) {
                console.log('✅ Adapter properly handled missing configuration - no invalid states created');
                resolve(true);
            } else {
                // If states were created, check if they're in error state
                const connectionState = await new Promise((res, rej) => {
                    harness.states.getState('firetv.0.info.connection', (err, state) => {
                        if (err) return rej(err);
                        res(state);
                    });
                });
                
                if (!connectionState || connectionState.val === false) {
                    console.log('✅ Adapter properly failed with missing configuration');
                    resolve(true);
                } else {
                    console.log('❌ Adapter should have failed or handled missing config gracefully');
                    reject(new Error('Adapter should have handled missing configuration'));
                }
            }

            await harness.stopAdapter();
        } catch (error) {
            console.log('✅ Adapter correctly threw error with missing configuration:', error.message);
            resolve(true);
        }
    });
}).timeout(40000);
```

#### Advanced State Access Patterns

For testing adapters that create multiple states, use bulk state access methods to efficiently verify large numbers of states:

```javascript
it('should create and verify multiple states', () => new Promise(async (resolve, reject) => {
    // Configure and start adapter first...
    harness.objects.getObject('system.adapter.firetv.0', async (err, obj) => {
        if (err) {
            console.error('Error getting adapter object:', err);
            reject(err);
            return;
        }

        // Configure adapter as needed
        obj.native.adbPath = '/usr/bin/adb';
        obj.native.devices = [{
            name: 'Test FireTV',
            ip: '192.168.1.100',
            enabled: true
        }];
        harness.objects.setObject(obj._id, obj);

        await harness.startAdapterAndWait();

        // Wait for adapter to create states
        setTimeout(() => {
            // Access bulk states using pattern matching
            harness.dbConnection.getStateIDs('firetv.0.*').then(stateIds => {
                if (stateIds && stateIds.length > 0) {
                    harness.states.getStates(stateIds, (err, allStates) => {
                        if (err) {
                            console.error('❌ Error getting states:', err);
                            reject(err); // Properly fail the test instead of just resolving
                            return;
                        }

                        // Verify states were created and have expected values
                        const expectedStates = ['firetv.0.info.connection'];
                        let foundStates = 0;
                        
                        for (const stateId of expectedStates) {
                            if (allStates[stateId]) {
                                foundStates++;
                                console.log(`✅ Found expected state: ${stateId}`);
                            } else {
                                console.log(`❌ Missing expected state: ${stateId}`);
                            }
                        }

                        if (foundStates === expectedStates.length) {
                            console.log('✅ All expected states were created successfully');
                            resolve();
                        } else {
                            reject(new Error(`Only ${foundStates}/${expectedStates.length} expected states were found`));
                        }
                    });
                } else {
                    reject(new Error('No states found matching pattern firetv.0.*'));
                }
            }).catch(reject);
        }, 20000); // Allow more time for multiple state creation
    });
})).timeout(45000);
```

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
integration-tests:
  needs: [check-and-lint, adapter-tests]
  runs-on: ubuntu-latest
  steps:
    - name: Run integration tests
      run: npx mocha test/integration-*.js --exit
```

#### What NOT to Do
❌ Direct API testing: `axios.get('https://api.example.com')`
❌ Mock adapters: `new MockAdapter()`  
❌ Direct internet calls in tests
❌ Bypassing the harness system

#### What TO Do
✅ Use `@iobroker/testing` framework
✅ Configure via `harness.objects.setObject()`
✅ Start via `harness.startAdapterAndWait()`
✅ Test complete adapter lifecycle
✅ Verify states via `harness.states.getState()`
✅ Allow proper timeouts for async operations

### API Testing with Credentials
For adapters that connect to external APIs requiring authentication, implement comprehensive credential testing:

#### Password Encryption for Integration Tests
When creating integration tests that need encrypted passwords (like those marked as `encryptedNative` in io-package.json):

1. **Read system secret**: Use `harness.objects.getObjectAsync("system.config")` to get `obj.native.secret`
2. **Apply XOR encryption**: Implement the encryption algorithm:
   ```javascript
   async function encryptPassword(harness, password) {
       const systemConfig = await harness.objects.getObjectAsync("system.config");
       if (!systemConfig || !systemConfig.native || !systemConfig.native.secret) {
           throw new Error("Could not retrieve system secret for password encryption");
       }
       
       const secret = systemConfig.native.secret;
       let result = '';
       for (let i = 0; i < password.length; ++i) {
           result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
       }
       return result;
   }
   ```
3. **Store encrypted password**: Set the encrypted result in adapter config, not the plain text
4. **Result**: Adapter will properly decrypt and use credentials, enabling full API connectivity testing

#### Demo Credentials Testing Pattern
- Use provider demo credentials when available (e.g., `demo@api-provider.com` / `demo`)
- Create separate test file (e.g., `test/integration-demo.js`) for credential-based tests
- Add npm script: `"test:integration-demo": "mocha test/integration-demo --exit"`
- Implement clear success/failure criteria with recognizable log messages
- Expected success pattern: Look for specific adapter initialization messages
- Test should fail clearly with actionable error messages for debugging

#### Enhanced Test Failure Handling
```javascript
it("Should connect to API with demo credentials", async () => {
    // ... setup and encryption logic ...
    
    const connectionState = await harness.states.getStateAsync("adapter.0.info.connection");
    
    if (connectionState && connectionState.val === true) {
        console.log("✅ SUCCESS: API connection established");
        return true;
    } else {
        throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
            "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
    }
}).timeout(120000); // Extended timeout for API calls
```

## README Updates

### Required Sections
When updating README.md files, ensure these sections are present and well-documented:

1. **Installation** - Clear npm/ioBroker admin installation steps
2. **Configuration** - Detailed configuration options with examples
3. **Usage** - Practical examples and use cases
4. **Changelog** - Version history and changes (use "## **WORK IN PROGRESS**" section for ongoing changes following AlCalzone release-script standard)
5. **License** - License information (typically MIT for ioBroker adapters)
6. **Support** - Links to issues, discussions, and community support

### Documentation Standards
- Use clear, concise language
- Include code examples for configuration
- Add screenshots for admin interface when applicable
- Maintain multilingual support (at minimum English and German)
- When creating PRs, add entries to README under "## **WORK IN PROGRESS**" section following ioBroker release script standard
- Always reference related issues in commits and PR descriptions (e.g., "solves #xx" or "fixes #xx")

### Mandatory README Updates for PRs
For **every PR or new feature**, always add a user-friendly entry to README.md:

- Add entries under `## **WORK IN PROGRESS**` section before committing
- Use format: `* (author) **TYPE**: Description of user-visible change`
- Types: **NEW** (features), **FIXED** (bugs), **ENHANCED** (improvements), **TESTING** (test additions), **CI/CD** (automation)
- Focus on user impact, not technical implementation details
- Example: `* (DutchmanNL) **FIXED**: Adapter now properly validates login credentials instead of always showing "credentials missing"`

### Documentation Workflow Standards
- **Mandatory README updates**: Establish requirement to update README.md for every PR/feature
- **Standardized documentation**: Create consistent format and categories for changelog entries
- **Enhanced development workflow**: Integrate documentation requirements into standard development process

### Changelog Management with AlCalzone Release-Script
Follow the [AlCalzone release-script](https://github.com/AlCalzone/release-script) standard for changelog management:

#### Format Requirements
- Always use `## **WORK IN PROGRESS**` as the placeholder for new changes
- Add all PR/commit changes under this section until ready for release
- Never modify version numbers manually - only when merging to main branch
- Maintain this format in README.md or CHANGELOG.md:

```markdown
# Changelog

<!--
  Placeholder for the next version (at the beginning of the line):
  ## **WORK IN PROGRESS**
-->

## **WORK IN PROGRESS**

-   Did some changes
-   Did some more changes

## v0.1.0 (2023-01-01)
Initial release
```

#### Workflow Process
- **During Development**: All changes go under `## **WORK IN PROGRESS**`
- **For Every PR**: Add user-facing changes to the WORK IN PROGRESS section
- **Before Merge**: Version number and date are only added when merging to main
- **Release Process**: The release-script automatically converts the placeholder to the actual version

#### Change Entry Format
Use this consistent format for changelog entries:
- `- (author) **TYPE**: User-friendly description of the change`
- Types: **NEW** (features), **FIXED** (bugs), **ENHANCED** (improvements)
- Focus on user impact, not technical implementation details
- Reference related issues: "fixes #XX" or "solves #XX"

#### Example Entry
```markdown
## **WORK IN PROGRESS**

- (DutchmanNL) **FIXED**: Adapter now properly validates login credentials instead of always showing "credentials missing" (fixes #25)
- (DutchmanNL) **NEW**: Added support for device discovery to simplify initial setup
```

## Dependency Updates

### Package Management
- Always use `npm` for dependency management in ioBroker adapters
- When working on new features in a repository with an existing package-lock.json file, use `npm ci` to install dependencies. Use `npm install` only when adding or updating dependencies.
- Keep dependencies minimal and focused
- Only update dependencies to latest stable versions when necessary or in separate Pull Requests. Avoid updating dependencies when adding features that don't require these updates.
- When you modify `package.json`:
  1. Run `npm install` to update and sync `package-lock.json`.
  2. If `package-lock.json` was updated, commit both `package.json` and `package-lock.json`.

### Dependency Best Practices
- Prefer built-in Node.js modules when possible
- Use `@iobroker/adapter-core` for adapter base functionality
- Avoid deprecated packages
- Document any specific version requirements

## JSON-Config Admin Instructions

### Configuration Schema
When creating admin configuration interfaces:

- Use JSON-Config format for modern ioBroker admin interfaces
- Provide clear labels and help text for all configuration options
- Include input validation and error messages
- Group related settings logically
- Example structure:
  ```json
  {
    "type": "panel",
    "items": {
      "host": {
        "type": "text",
        "label": "Host address",
        "help": "IP address or hostname of the device"
      }
    }
  }
  ```

### Admin Interface Guidelines
- Use consistent naming conventions
- Provide sensible default values
- Include validation for required fields
- Add tooltips for complex configuration options
- Ensure translations are available for all supported languages (minimum English and German)
- Write end-user friendly labels and descriptions, avoiding technical jargon where possible

## Best Practices for Dependencies

### HTTP Client Libraries
- **Preferred:** Use native `fetch` API (Node.js 20+ required for adapters; built-in since Node.js 18)
- **Avoid:** `axios` unless specific features are required (reduces bundle size)

### Example with fetch:
```javascript
try {
  const response = await fetch('https://api.example.com/data');
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  const data = await response.json();
} catch (error) {
  this.log.error(`API request failed: ${error.message}`);
}
```

### Other Dependency Recommendations
- **Logging:** Use adapter built-in logging (`this.log.*`)
- **Scheduling:** Use adapter built-in timers and intervals
- **File operations:** Use Node.js `fs/promises` for async file operations
- **Configuration:** Use adapter config system rather than external config libraries

## Error Handling

### Adapter Error Patterns
- Always catch and log errors appropriately
- Use adapter log levels (error, warn, info, debug)
- Provide meaningful, user-friendly error messages that help users understand what went wrong
- Handle network failures gracefully
- Implement retry mechanisms where appropriate
- Always clean up timers, intervals, and other resources in the `unload()` method

### Example Error Handling:
```javascript
try {
  await this.connectToDevice();
} catch (error) {
  this.log.error(`Failed to connect to device: ${error.message}`);
  this.setState('info.connection', false, true);
  // Implement retry logic if needed
}
```

### Timer and Resource Cleanup:
```javascript
// In your adapter class
private connectionTimer?: NodeJS.Timeout;

async onReady() {
  this.connectionTimer = setInterval(() => {
    this.checkConnection();
  }, 30000);
}

onUnload(callback) {
  try {
    // Clean up timers and intervals
    if (this.connectionTimer) {
      clearInterval(this.connectionTimer);
      this.connectionTimer = undefined;
    }
    // Close connections, clean up resources
    callback();
  } catch (e) {
    callback();
  }
}
```

## Code Style and Standards

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods

## CI/CD and Testing Integration

### GitHub Actions for API Testing
For adapters with external API dependencies, implement separate CI/CD jobs:

```yaml
# Tests API connectivity with demo credentials (runs separately)
demo-api-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false
  
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run demo API tests
      run: npm run test:integration-demo
```

### CI/CD Best Practices
- Run credential tests separately from main test suite
- Use ubuntu-22.04 for consistency
- Don't make credential tests required for deployment
- Provide clear failure messages for API connectivity issues
- Use appropriate timeouts for external API calls (120+ seconds)

### Package.json Script Integration
Add dedicated script for credential testing:
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit"
  }
}
```

### Practical Example: Complete API Testing Implementation
Here's a complete example based on lessons learned from the Discovergy adapter:

#### test/integration-demo.js
```javascript
const path = require("path");
const { tests } = require("@iobroker/testing");

// Helper function to encrypt password using ioBroker's encryption method
async function encryptPassword(harness, password) {
    const systemConfig = await harness.objects.getObjectAsync("system.config");
    
    if (!systemConfig || !systemConfig.native || !systemConfig.native.secret) {
        throw new Error("Could not retrieve system secret for password encryption");
    }
    
    const secret = systemConfig.native.secret;
    let result = '';
    for (let i = 0; i < password.length; ++i) {
        result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
    }
    
    return result;
}

// Run integration tests with demo credentials
tests.integration(path.join(__dirname, ".."), {
    defineAdditionalTests({ suite }) {
        suite("API Testing with Demo Credentials", (getHarness) => {
            let harness;
            
            before(() => {
                harness = getHarness();
            });

            it("Should connect to API and initialize with demo credentials", async () => {
                console.log("Setting up demo credentials...");
                
                if (harness.isAdapterRunning()) {
                    await harness.stopAdapter();
                }
                
                const encryptedPassword = await encryptPassword(harness, "demo_password");
                
                await harness.changeAdapterConfig("your-adapter", {
                    native: {
                        username: "demo@provider.com",
                        password: encryptedPassword,
                        // other config options
                    }
                });

                console.log("Starting adapter with demo credentials...");
                await harness.startAdapter();
                
                // Wait for API calls and initialization
                await new Promise(resolve => setTimeout(resolve, 60000));
                
                const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");
                
                if (connectionState && connectionState.val === true) {
                    console.log("✅ SUCCESS: API connection established");
                    return true;
                } else {
                    throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
                        "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
                }
            }).timeout(120000);
        });
    }
});
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