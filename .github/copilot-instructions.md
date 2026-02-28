# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.5.7
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

---

## 📑 Table of Contents

1. [Project Context](#project-context)
2. [Code Quality & Standards](#code-quality--standards)
   - [Code Style Guidelines](#code-style-guidelines)
   - [ESLint Configuration](#eslint-configuration)
3. [Testing](#testing)
   - [Unit Testing](#unit-testing)
   - [Integration Testing](#integration-testing)
   - [API Testing with Credentials](#api-testing-with-credentials)
4. [Development Best Practices](#development-best-practices)
   - [Dependency Management](#dependency-management)
   - [HTTP Client Libraries](#http-client-libraries)
   - [Error Handling](#error-handling)
5. [Admin UI Configuration](#admin-ui-configuration)
   - [JSON-Config Setup](#json-config-setup)
   - [Translation Management](#translation-management)
6. [Documentation](#documentation)
   - [README Updates](#readme-updates)
   - [Changelog Management](#changelog-management)
7. [CI/CD & GitHub Actions](#cicd--github-actions)
   - [Workflow Configuration](#workflow-configuration)
   - [Testing Integration](#testing-integration)
8. [Adapter Architecture](#adapter-architecture)
9. [Development Guidelines](#development-guidelines)
10. [Common Issues & Troubleshooting](#common-issues--troubleshooting)

---

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

---

## Code Quality & Standards

### Code Style Guidelines

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods

**Timer and Resource Cleanup Example:**
```javascript
private connectionTimer?: NodeJS.Timeout;

async onReady() {
  this.connectionTimer = setInterval(() => this.checkConnection(), 30000);
}

onUnload(callback) {
  try {
    if (this.connectionTimer) {
      clearInterval(this.connectionTimer);
      this.connectionTimer = undefined;
    }
    callback();
  } catch (e) {
    callback();
  }
}
```

### ESLint Configuration

**CRITICAL:** ESLint validation must run FIRST in your CI/CD pipeline, before any other tests. This "lint-first" approach catches code quality issues early.

#### Setup
```bash
npm install --save-dev eslint @iobroker/eslint-config
```

#### Configuration (.eslintrc.json)
```json
{
  "extends": "@iobroker/eslint-config",
  "rules": {
    // Add project-specific rule overrides here if needed
  }
}
```

#### Package.json Scripts
```json
{
  "scripts": {
    "lint": "eslint --max-warnings 0 .",
    "lint:fix": "eslint . --fix"
  }
}
```

#### Best Practices
1. ✅ Run ESLint before committing — fix ALL warnings, not just errors
2. ✅ Use `lint:fix` for auto-fixable issues
3. ✅ Don't disable rules without documentation
4. ✅ Lint all relevant files (main code, tests, build scripts)
5. ✅ Keep `@iobroker/eslint-config` up to date
6. ✅ **ESLint warnings are treated as errors in CI** (`--max-warnings 0`). The `lint` script above already includes this flag — run `npm run lint` to match CI behavior locally

#### Common Issues
- **Unused variables**: Remove or prefix with underscore (`_variable`)
- **Missing semicolons**: Run `npm run lint:fix`
- **Indentation**: Use 4 spaces (ioBroker standard)
- **console.log**: Replace with `adapter.log.debug()` or remove

---

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

#### Key Rules

1. ✅ Use `@iobroker/testing` framework
2. ✅ Configure via `harness.objects.setObject()`
3. ✅ Start via `harness.startAdapterAndWait()`
4. ✅ Verify states via `harness.states.getState()`
5. ✅ Allow proper timeouts for async operations
6. ❌ NEVER test API URLs directly
7. ❌ NEVER bypass the harness system

#### Workflow Dependencies

Integration tests should run ONLY after lint and adapter tests pass:

```yaml
integration-tests:
  needs: [check-and-lint, adapter-tests]
  runs-on: ubuntu-22.04
```

### API Testing with Credentials

For adapters connecting to external APIs requiring authentication:

#### Password Encryption for Integration Tests

```javascript
async function encryptPassword(harness, password) {
    const systemConfig = await harness.objects.getObjectAsync("system.config");
    if (!systemConfig?.native?.secret) {
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

#### Demo Credentials Testing Pattern

- Use provider demo credentials when available (e.g., `demo@api-provider.com` / `demo`)
- Create separate test file: `test/integration-demo.js`
- Add npm script: `"test:integration-demo": "mocha test/integration-demo --exit"`
- Implement clear success/failure criteria

**Example Implementation:**
```javascript
it("Should connect to API with demo credentials", async () => {
    const encryptedPassword = await encryptPassword(harness, "demo_password");
    
    await harness.changeAdapterConfig("your-adapter", {
        native: {
            username: "demo@provider.com",
            password: encryptedPassword,
        }
    });

    await harness.startAdapter();
    await new Promise(resolve => setTimeout(resolve, 60000));
    
    const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");
    
    if (connectionState?.val === true) {
        console.log("✅ SUCCESS: API connection established");
        return true;
    } else {
        throw new Error("API Test Failed: Expected API connection. Check logs for API errors.");
    }
}).timeout(120000);
```

---

## Development Best Practices

### Dependency Management

- Always use `npm` for dependency management
- Use `npm ci` for installing existing dependencies (respects package-lock.json)
- Use `npm install` only when adding or updating dependencies
- Keep dependencies minimal and focused
- Only update dependencies in separate Pull Requests

**When modifying package.json:**
1. Run `npm install` to sync package-lock.json
2. Commit both package.json and package-lock.json together

**Best Practices:**
- Prefer built-in Node.js modules when possible
- Use `@iobroker/adapter-core` for adapter base functionality
- Avoid deprecated packages
- Document specific version requirements

### HTTP Client Libraries

- **Preferred:** Use native `fetch` API (Node.js 20+ required)
- **Avoid:** `axios` unless specific features are required

**Example with fetch:**
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

**Other Recommendations:**
- **Logging:** Use adapter built-in logging (`this.log.*`)
- **Scheduling:** Use adapter built-in timers and intervals
- **File operations:** Use Node.js `fs/promises`
- **Configuration:** Use adapter config system

### Error Handling

- Always catch and log errors appropriately
- Use adapter log levels (error, warn, info, debug)
- Provide meaningful, user-friendly error messages
- Handle network failures gracefully
- Implement retry mechanisms where appropriate
- Always clean up timers, intervals, and resources in `unload()` method

**Example:**
```javascript
try {
  await this.connectToDevice();
} catch (error) {
  this.log.error(`Failed to connect to device: ${error.message}`);
  this.setState('info.connection', false, true);
  // Implement retry logic if needed
}
```

---

## Admin UI Configuration

### JSON-Config Setup

Use JSON-Config format for modern ioBroker admin interfaces.

**Example Structure:**
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

**Guidelines:**
- ✅ Use consistent naming conventions
- ✅ Provide sensible default values
- ✅ Include validation for required fields
- ✅ Add tooltips for complex options
- ✅ Ensure translations for all supported languages (minimum English and German)
- ✅ Write end-user friendly labels, avoid technical jargon

### Translation Management

**CRITICAL:** Translation files must stay synchronized with `admin/jsonConfig.json`. Orphaned keys or missing translations cause UI issues and PR review delays.

#### Overview
- **Location:** `admin/i18n/{lang}/translations.json` for 11 languages (de, en, es, fr, it, nl, pl, pt, ru, uk, zh-cn)
- **Source of truth:** `admin/jsonConfig.json` - all `label` and `help` properties must have translations
- **Command:** `npm run translate` - auto-generates translations but does NOT remove orphaned keys
- **Formatting:** English uses tabs, other languages use 4 spaces

#### Critical Rules
1. ✅ Keys must match exactly with jsonConfig.json
2. ✅ No orphaned keys in translation files
3. ✅ All translations must be in native language (no English fallbacks)
4. ✅ Keys must be sorted alphabetically

#### Workflow for Translation Updates

**When modifying admin/jsonConfig.json:**

1. Make your changes to labels/help texts
2. Run automatic translation: `npm run translate`
3. Create validation script (`scripts/validate-translations.js`):

```javascript
const fs = require('fs');
const path = require('path');
const jsonConfig = JSON.parse(fs.readFileSync('admin/jsonConfig.json', 'utf8'));

function extractTexts(obj, texts = new Set()) {
    if (typeof obj === 'object' && obj !== null) {
        if (obj.label) texts.add(obj.label);
        if (obj.help) texts.add(obj.help);
        for (const key in obj) {
            extractTexts(obj[key], texts);
        }
    }
    return texts;
}

const requiredTexts = extractTexts(jsonConfig);
const languages = ['de', 'en', 'es', 'fr', 'it', 'nl', 'pl', 'pt', 'ru', 'uk', 'zh-cn'];
let hasErrors = false;

languages.forEach(lang => {
    const translationPath = path.join('admin', 'i18n', lang, 'translations.json');
    const translations = JSON.parse(fs.readFileSync(translationPath, 'utf8'));
    const translationKeys = new Set(Object.keys(translations));
    
    const missing = Array.from(requiredTexts).filter(text => !translationKeys.has(text));
    const orphaned = Array.from(translationKeys).filter(key => !requiredTexts.has(key));
    
    console.log(`\n=== ${lang} ===`);
    if (missing.length > 0) {
        console.error('❌ Missing keys:', missing);
        hasErrors = true;
    }
    if (orphaned.length > 0) {
        console.error('❌ Orphaned keys (REMOVE THESE):', orphaned);
        hasErrors = true;
    }
    if (missing.length === 0 && orphaned.length === 0) {
        console.log('✅ All keys match!');
    }
});

process.exit(hasErrors ? 1 : 0);
```

4. Run validation: `node scripts/validate-translations.js`
5. Remove orphaned keys manually from all translation files
6. Add missing translations in native languages
7. Run: `npm run lint && npm run test`

#### Add Validation to package.json

```json
{
  "scripts": {
    "translate": "translate-adapter",
    "validate:translations": "node scripts/validate-translations.js",
    "pretest": "npm run lint && npm run validate:translations"
  }
}
```

---

## Documentation

### README Updates

#### Required Sections
1. **Installation** - Clear npm/ioBroker admin installation steps
2. **Configuration** - Detailed configuration options with examples
3. **Usage** - Practical examples and use cases
4. **Changelog** - Version history (use "## **WORK IN PROGRESS**" for ongoing changes)
5. **License** - License information (typically MIT for ioBroker adapters)
6. **Support** - Links to issues, discussions, community support

#### Documentation Standards
- Use clear, concise language
- Include code examples for configuration
- Add screenshots for admin interface when applicable
- Maintain multilingual support (minimum English and German)
- Always reference issues in commits and PRs (e.g., "fixes #xx")

#### Mandatory README Updates for PRs

For **every PR or new feature**, always add a user-friendly entry to README.md:

- Add entries under `## **WORK IN PROGRESS**` section
- Use format: `* (author) **TYPE**: Description of user-visible change`
- Types: **NEW** (features), **FIXED** (bugs), **ENHANCED** (improvements), **TESTING** (test additions), **CI/CD** (automation)
- Focus on user impact, not technical details

**Example:**
```markdown
## **WORK IN PROGRESS**

* (DutchmanNL) **FIXED**: Adapter now properly validates login credentials (fixes #25)
* (DutchmanNL) **NEW**: Added device discovery to simplify initial setup
```

### Changelog Management

Follow the [AlCalzone release-script](https://github.com/AlCalzone/release-script) standard.

#### Format Requirements

```markdown
# Changelog

<!--
  Placeholder for the next version (at the beginning of the line):
  ## **WORK IN PROGRESS**
-->

## **WORK IN PROGRESS**

- (author) **NEW**: Added new feature X
- (author) **FIXED**: Fixed bug Y (fixes #25)

## v0.1.0 (2023-01-01)
Initial release
```

#### Workflow Process
- **During Development:** All changes go under `## **WORK IN PROGRESS**`
- **For Every PR:** Add user-facing changes to WORK IN PROGRESS section
- **Before Merge:** Version number and date added when merging to main
- **Release Process:** Release-script automatically converts placeholder to actual version

#### Change Entry Format
- Format: `- (author) **TYPE**: User-friendly description`
- Types: **NEW**, **FIXED**, **ENHANCED**
- Focus on user impact, not technical implementation
- Reference issues: "fixes #XX" or "solves #XX"

---

## CI/CD & GitHub Actions

### Workflow Configuration

#### GitHub Actions Best Practices

**Must use ioBroker official testing actions:**
- `ioBroker/testing-action-check@v1` for lint and package validation
- `ioBroker/testing-action-adapter@v1` for adapter tests
- `ioBroker/testing-action-deploy@v1` for automated releases with Trusted Publishing (OIDC)

**Configuration:**
- **Node.js versions:** Test on 20.x, 22.x, 24.x
- **Platform:** Use ubuntu-22.04
- **Automated releases:** Deploy to npm on version tags (requires NPM Trusted Publishing)
- **Monitoring:** Include Sentry release tracking for error monitoring

#### Critical: Lint-First Validation Workflow

**ALWAYS run ESLint checks BEFORE other tests.** Benefits:
- Catches code quality issues immediately
- Prevents wasting CI resources on tests that would fail due to linting errors
- Provides faster feedback to developers
- Enforces consistent code quality

**Workflow Dependency Configuration:**
```yaml
jobs:
  check-and-lint:
    # Runs ESLint and package validation
    # Uses: ioBroker/testing-action-check@v1
    
  adapter-tests:
    needs: [check-and-lint]  # Wait for linting to pass
    # Run adapter unit tests
    
  integration-tests:
    needs: [check-and-lint, adapter-tests]  # Wait for both
    # Run integration tests
```

**Key Points:**
- The `check-and-lint` job has NO dependencies - runs first
- ALL other test jobs MUST list `check-and-lint` in their `needs` array
- If linting fails, no other tests run, saving time
- Fix all ESLint errors before proceeding

### Testing Integration

#### API Testing in CI/CD

For adapters with external API dependencies:

```yaml
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

#### Testing Best Practices
- Run credential tests separately from main test suite
- Don't make credential tests required for deployment
- Provide clear failure messages for API issues
- Use appropriate timeouts for external calls (120+ seconds)

#### Package.json Integration
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit"
  }
}
```

---

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