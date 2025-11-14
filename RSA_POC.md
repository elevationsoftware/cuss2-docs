# CUSS2 RSA-OAEP Encryption for Sensitive Data

## Overview

This section describes the RSA-OAEP public-key encryption feature for securing sensitive data transmitted between CUSS applications and platforms. This approach allows applications to provide a public key during component setup, enabling the platform to encrypt sensitive data (such as magnetic stripe payment card data) before transmission back to the application.

**Key Benefits:**
- End-to-end encryption of sensitive cardholder data
- Platform never accesses or stores private keys
- Compliant with PCI-DSS requirements
- Zero key persistence—keys cleared when application becomes inactive
- Application controls complete security lifecycle

## Session-Based Key Management

**IMPORTANT:** Public keys are **session-scoped only** and must be provided each time the application becomes ACTIVE:

- **Keys are NOT persisted** across application state changes
- When application transitions to AVAILABLE, STOPPED, or any non-ACTIVE state, the platform **discards all public keys**
- Applications **MUST** call `peripherals_setup` with the public key **every time** they transition to ACTIVE state
- This ensures complete security isolation between passenger sessions

### State Transition Key Management

```
Application State Changes:
┌──────────────────────────────────────────────────────────────┐
│ ACTIVE → AVAILABLE/UNAVAILABLE     Platform Action: DELETE KEYS  │
│ AVAILABLE → ACTIVE     App Action: SEND NEW KEY      │
└──────────────────────────────────────────────────────────────┘
```

## Conceptual Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│             RSA-OAEP Encryption Flow (Per Session)                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────┐                                    ┌──────────────┐
│ Application │                                    │   Platform   │
└──────┬──────┘                                    └──────┬───────┘
       │                                                  │
       │ [App becomes ACTIVE]                             │
       │                                                  │
       │ 1. Generate NEW RSA Key Pair                     │
       │    (2048-bit or 4096-bit)                        │
       │                                                  │
       │ 2. peripherals_setup                             │
       │    + Public Key (DS_TYPES_RSA_PUBLIC_KEY)        │
       ├─────────────────────────────────────────────────>│
       │                                                  │ Store key
       │ 3. ACK_OK                                        │ (session only)
       │<─────────────────────────────────────────────────┤
       │                                                  │
       │ 4. peripherals_userpresent_enable                │
       ├─────────────────────────────────────────────────>│
       │                                                  │
       │                          [User swipes card]      │
       │                                                  │ Read & encrypt
       │ 5. DATA_PRESENT Event                            │ with stored key
       │    (Encrypted data)                              │
       │<─────────────────────────────────────────────────┤
       │                                                  │
       │ 6. Decrypt with private key                      │
       │                                                  │
       │ [Transaction complete - App goes AVAILABLE]      │
       │                                                  │ DELETE all keys
       │                                                  │
```

## Technical Implementation

### Step 1: Application Generates RSA Key Pair

**When:** Every time the application becomes ACTIVE (including after initial activation, transfers, or any return to ACTIVE state)

**Why:** Keys are never persisted, ensuring complete session isolation and security

The application generates an RSA key pair before sending the setup directive. The recommended key size is **2048 bits** for a balance of security and performance, though **4096 bits** may be used for enhanced security.

**Key Generation Example (JavaScript/Node.js):**
```javascript
const crypto = require('crypto');

// Generate FRESH RSA key pair for this session
function generateSessionKeyPair() {
  const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
    modulusLength: 2048,
    publicKeyEncoding: {
      type: 'spki',
      format: 'pem'
    },
    privateKeyEncoding: {
      type: 'pkcs8',
      format: 'pem'
    }
  });

  console.log('✓ Generated new session key pair');
  
  // Store private key securely for THIS SESSION ONLY
  return { publicKey, privateKey };
}
```

**Key Generation Example (Python):**
```python
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend

def generate_session_key_pair():
    """Generate fresh RSA key pair for current session"""
    private_key = rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048,
        backend=default_backend()
    )

    public_key = private_key.public_key()

    # Serialize public key to PEM format
    public_key_pem = public_key.public_key_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )

    print('✓ Generated new session key pair')
    
    # Store private_key for THIS SESSION ONLY
    return private_key, public_key_pem
```

### Step 2: Setup Directive with Public Key

**CRITICAL:** This must be called **every time** the application becomes ACTIVE, before any encrypted operations.

The application sends the public key to the platform using the `peripherals_setup` directive with a `dataRecords` payload containing the public key encoded in Base64.

**Setup Request Structure:**
```json
{
  "meta": {
    "deviceID": "550e8400-e29b-41d4-a716-446655440000",
    "requestID": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "oauthToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "directive": "peripherals_setup",
    "componentID": 42
  },
  "payload": {
    "dataRecords": [
      {
        "dataStatus": "DS_OK",
        "dsTypes": ["DS_TYPES_RSA_PUBLIC_KEY","DS_TYPES_PAYMENT_ISO"],
        "data": "LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF3...",
        "encoding": "BASE64"
      }
    ]
  }
}
```

**Field Descriptions:**

| Field | Description |
|-------|-------------|
| `dsTypes` | Must include `DS_TYPES_RSA_PUBLIC_KEY` to indicate this is an RSA public key. May also include data type for operations (e.g., `DS_TYPES_PAYMENT_ISO`) |
| `data` | Base64-encoded PEM format public key |
| `encoding` | Must be `BASE64` |
| `dataStatus` | Should be `DS_OK` for valid key data |

### Step 3: Platform Acknowledgment

The platform validates and stores the public key **for the current session only**, responding with `ACK_OK`:

```json
{
  "ackCode": "ACK_OK",
  "requestID": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "description": "Public key stored for current session"
}
```

**Platform Behavior:**
- Validates public key format and encoding
- Stores key in memory associated with component and current application session
- **Automatically discards key** when application transitions away from ACTIVE state

**Possible Error Responses:**

| ACK Code | Reason |
|----------|--------|
| `ACK_PARAMETER` | Invalid public key format, encoding error, or missing data |
| `ACK_ERROR` | Component doesn't support encryption or internal platform error |
| `WRONG_APPLICATION_STATE` | Application not in ACTIVE state |

### Step 4: Enable Component for Use

After successful setup, enable the component normally:

```json
{
  "meta": {
    "deviceID": "550e8400-e29b-41d4-a716-446655440000",
    "requestID": "8d0f7680-8536-51ef-b827-f18gc2g01bf8",
    "oauthToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "directive": "peripherals_userpresent_enable",
    "componentID": 42
  },
  "payload": {}
}
```

### Step 5: Receive Encrypted Data

When the platform reads sensitive data (e.g., magnetic stripe), it encrypts the data using RSA-OAEP with SHA-256 before sending the `DATA_PRESENT` event:

**DATA_PRESENT Event with Encrypted Data:**
```json
{
  "meta": {
    "deviceID": "550e8400-e29b-41d4-a716-446655440000",
    "requestID": "8d0f7680-8536-51ef-b827-f18gc2g01bf8",
    "timeStamp": "2025-01-15T14:23:45.123Z",
    "passengerSessionID": "9e1g8791-9647-62fg-c938-g29hd3h12cg9",
    "applicationID": {
      "companyCode": "AA",
      "applicationName": "CheckInApp"
    },
    "componentID": 42,
    "componentState": "READY",
    "messageCode": "DATA_PRESENT",
    "eventClassification": {
      "eventMode": "SOLICITED",
      "eventType": "PRIVATE",
      "eventCategory": "NORMAL"
    }
  },
  "payload": {
    "dataRecords": [
      {
        "dataStatus": "DS_OK",
        "dsTypes": [
          "DS_TYPES_ENCRYPTED_DATA",
          "DS_TYPES_PAYMENT_ISO"
        ],
        "data": "kQp8xNZ2L5vHtJ9mW3rF0eK7cY1bX4dP6sA8nM9gV2lR5oU7iT0qE3wZ...",
        "encoding": "BASE64"
      }
    ]
  }
}
```

**Field Descriptions:**

| Field | Description |
|-------|-------------|
| `dsTypes[0]` | `DS_TYPES_ENCRYPTED_DATA` indicates the data is encrypted |
| `dsTypes[1]` | `DS_TYPES_PAYMENT_ISO` indicates the original data format (before encryption) |
| `data` | Base64-encoded RSA-OAEP encrypted data |
| `encoding` | Always `BASE64` for encrypted data |

### Step 6: Application Decrypts Data

The application uses its private key (from the current session) to decrypt the received data:

**Decryption Example (JavaScript/Node.js):**
```javascript
const crypto = require('crypto');

function decryptMagStripeData(encryptedDataB64, privateKeyPem) {
  try {
    // Decode from Base64
    const encryptedBuffer = Buffer.from(encryptedDataB64, 'base64');
    
    // Decrypt using RSA-OAEP with SHA-256
    const decryptedBuffer = crypto.privateDecrypt(
      {
        key: privateKeyPem,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      encryptedBuffer
    );
    
    // Convert to string
    const magStripeData = decryptedBuffer.toString('utf8');
    console.log('✓ Decrypted mag stripe data');
    
    return magStripeData;
  } catch (error) {
    console.error('✗ Decryption failed:', error);
    throw error;
  }
}

// Usage
const encryptedData = "kQp8xNZ2L5vHtJ9mW3rF0eK7cY1bX4dP6sA8nM9gV2lR5oU7iT0qE3wZ...";
const decryptedData = decryptMagStripeData(encryptedData, privateKey);
// Result: "%B4111111111111111^DOE/JOHN^2512101123456789?"
```

**Decryption Example (Python):**
```python
import base64
from cryptography.hazmat.primitives import serialization, hashes
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.backends import default_backend

def decrypt_mag_stripe_data(encrypted_data_b64, private_key):
    try:
        # Decode from Base64
        encrypted_data = base64.b64decode(encrypted_data_b64)
        
        # Decrypt using RSA-OAEP with SHA-256
        decrypted_data = private_key.decrypt(
            encrypted_data,
            padding.OAEP(
                mgf=padding.MGF1(algorithm=hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )
        
        # Convert to string
        mag_stripe_data = decrypted_data.decode('utf-8')
        print(f"✓ Decrypted mag stripe data")
        
        return mag_stripe_data
    except Exception as e:
        print(f"✗ Decryption failed: {e}")
        raise

# Usage
encrypted_data = "kQp8xNZ2L5vHtJ9mW3rF0eK7cY1bX4dP6sA8nM9gV2lR5oU7iT0qE3wZ..."
decrypted_data = decrypt_mag_stripe_data(encrypted_data, private_key)
# Result: "%B4111111111111111^DOE/JOHN^2512101123456789?"
```

## Session Lifecycle and Key Management

### Complete Session Flow

```
┌────────────────────────────────────────────────────────────────┐
│                    Session Lifecycle                           │
└────────────────────────────────────────────────────────────────┘

Application                                Platform
───────────                                ────────

[State: AVAILABLE]                         [No keys stored]
       │
       │ State change to ACTIVE
       ├──────────────────────────────────>
       │                                   
[State: ACTIVE]                            
       │
       │ Generate NEW key pair
       │
       │ peripherals_setup + public key
       ├──────────────────────────────────> Store key (session)
       │
       │ ACK_OK
       │<──────────────────────────────────
       │
       │ [Use encrypted operations]
       │ <────────────────────────────────> [Encrypt with key]
       │
       │ Transaction complete
       │ State change to AVAILABLE
       ├──────────────────────────────────>
       │                                    DELETE all keys
[State: AVAILABLE]                         [No keys stored]
       │
       │ [Next passenger]
       │ State change to ACTIVE
       ├──────────────────────────────────>
       │
[State: ACTIVE]
       │
       │ Generate NEW key pair
       │ peripherals_setup + NEW public key
       ├──────────────────────────────────> Store NEW key
       │                                    (new session)
```

### Application State Handler Example

```javascript
class SessionKeyManager {
  constructor() {
    this.currentKeyPair = null;
    this.currentState = 'AVAILABLE';
  }
  
  async handleStateChange(newState) {
    console.log(`State transition: ${this.currentState} → ${newState}`);
    
    if (newState === 'ACTIVE') {
      // Generate fresh keys for new session
      console.log('✓ Application becoming ACTIVE');
      this.currentKeyPair = this.generateSessionKeyPair();
      
      // Setup ALL components that need encryption
      await this.setupAllEncryptedComponents();
      
      console.log('✓ Ready for encrypted operations');
      
    } else if (this.currentState === 'ACTIVE' && newState !== 'ACTIVE') {
      // Leaving ACTIVE state - clear keys
      console.log('✓ Application leaving ACTIVE state');
      this.currentKeyPair = null; // Discard keys
      console.log('✓ Session keys cleared');
    }
    
    this.currentState = newState;
  }
  
  generateSessionKeyPair() {
    const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
    });
    
    console.log('✓ Generated new session key pair');
    return { publicKey, privateKey };
  }
  
  async setupAllEncryptedComponents() {
    // Setup each component that needs encryption
    const encryptedComponents = [42, 43]; // MSR readers
    
    for (const componentId of encryptedComponents) {
      await this.setupEncryptedComponent(componentId);
    }
  }
  
  async setupEncryptedComponent(componentId) {
    const request = {
      meta: {
        deviceID: this.deviceID,
        requestID: crypto.randomUUID(),
        oauthToken: this.oauthToken,
        directive: "peripherals_setup",
        componentID: componentId
      },
      payload: {
        dataRecords: [{
          dataStatus: "DS_OK",
          dsTypes: ["DS_TYPES_RSA_PUBLIC_KEY"],
          data: Buffer.from(this.currentKeyPair.publicKey).toString('base64'),
          encoding: "BASE64"
        }]
      }
    };
    
    this.websocket.send(JSON.stringify(request));
    console.log(`✓ Sent public key for component ${componentId}`);
  }
}

// Usage
const keyManager = new SessionKeyManager();

// When platform activates application
await keyManager.handleStateChange('ACTIVE');

// When transaction complete
await keyManager.handleStateChange('AVAILABLE');
```

## Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│              Per-Session Encryption Data Flow                    │
└──────────────────────────────────────────────────────────────────┘

Application Side                    Platform Side
─────────────────                   ──────────────

[ACTIVE State]
       │
┌──────┴──────┐
│  Generate   │
│  NEW Keys   │
│ (per session)│
└──────┬──────┘
       │
       │ Public Key (PEM)
       ↓
┌─────────────┐                    ┌──────────────┐
│   Base64    │──────────────────→ │    Store     │
│   Encode    │   peripherals_     │  (session    │
└─────────────┘      setup         │   memory)    │
                                   └──────┬───────┘
                                          │
                         [Card Swipe]     │
                                          ↓
                                    ┌──────────────┐
                                    │  Read Mag    │
                                    │   Stripe     │
                                    └──────┬───────┘
                                           │
                                           ↓
                                    ┌──────────────┐
                                    │   Encrypt    │
                                    │  RSA-OAEP    │
                                    │   SHA-256    │
                                    └──────┬───────┘
                                           │
┌─────────────┐                           │
│   Decrypt   │ ←─────────────────────────┘
│  with       │    DATA_PRESENT
│Session Key  │    (Base64)
└──────┬──────┘
       │
       ↓
┌─────────────┐
│   Process   │
│  Payment    │
└─────────────┘

[AVAILABLE State]
       │
       ↓
┌─────────────┐                    ┌──────────────┐
│   Discard   │                    │    DELETE    │
│   Private   │                    │   Public     │
│    Key      │                    │    Key       │
└─────────────┘                    └──────────────┘
```

## Security Considerations

### Session-Based Key Security

1. **No Key Persistence:**
   - Platform NEVER stores keys to disk or persistent storage
   - Keys exist only in memory for current application session
   - Automatic cleanup on state transition ensures zero key leakage

2. **Fresh Keys Per Session:**
   - New key pair generated each time application becomes ACTIVE
   - No key reuse across passenger sessions
   - Each transaction isolated with unique encryption

3. **Private Key Protection:**
   - Application stores private keys only for current session
   - Clear private key from memory when transitioning to non-ACTIVE state
   - Never log, transmit, or persist private keys

### Key Management Best Practices

1. **Private Key Storage (Session Only):**
   - Store in secure memory during ACTIVE state only
   - Overwrite/zero memory when clearing keys
   - Never write private keys to disk or logs
   - Use hardware security modules (HSMs) when available

2. **Key Size:**
   - Minimum: 2048 bits (recommended for most deployments)
   - Enhanced security: 4096 bits (higher security, slightly slower)

3. **State Transition Handling:**
   ```javascript
   // CRITICAL: Clear keys on state change
   onStateChange(newState) {
     if (newState !== 'ACTIVE') {
       // Overwrite sensitive memory
       if (this.privateKey) {
         this.privateKey = null;
       }
       this.currentKeyPair = null;
     }
   }
   ```

### Encryption Details

**Algorithm:** RSA-OAEP (Optimal Asymmetric Encryption Padding)

**Hash Function:** SHA-256

**Key Format:** PEM (Privacy-Enhanced Mail)

**Data Size Limits:**
- 2048-bit key: Maximum plaintext = 214 bytes (2048/8 - 42)
- 4096-bit key: Maximum plaintext = 470 bytes (4096/8 - 42)

**Performance:**
- Key generation: ~100-300ms (2048-bit)
- Encryption: ~2-10ms per operation
- Negligible impact for magnetic stripe operations

### PCI-DSS Compliance

This session-based encryption approach supports PCI-DSS compliance by:
- Encrypting cardholder data in transit
- Preventing platform storage of unencrypted cardholder data
- Ensuring only the application can decrypt sensitive data
- Meeting "strong cryptography" requirements (RSA-2048+)
- Zero key persistence eliminates long-term key storage risks
- Session isolation prevents cross-contamination

## Error Handling

### Common Errors and Solutions

| Error Scenario | Cause | Solution |
|----------------|-------|----------|
| ACK_PARAMETER on setup | Invalid public key format | Verify PEM format and Base64 encoding |
| Decryption fails | Wrong private key or corrupted data | Verify using correct session key pair |
| WRONG_APPLICATION_STATE | Setup called when not ACTIVE | Ensure application is ACTIVE before setup |
| DATA_PRESENT without encryption | No public key configured | Call peripherals_setup after becoming ACTIVE |
| Key size error | Key too small (<2048 bits) | Generate larger key pair (≥2048 bits) |

### State-Aware Error Handling

```javascript
async function setupEncryptedComponent(componentId, publicKey, currentState) {
  try {
    // Verify state before attempting setup
    if (currentState !== 'ACTIVE') {
      throw new Error(
        `Cannot setup encryption in ${currentState} state. ` +
        `Application must be ACTIVE.`
      );
    }
    
    const setupRequest = {
      meta: {
        deviceID: this.deviceID,
        requestID: crypto.randomUUID(),
        oauthToken: this.oauthToken,
        directive: "peripherals_setup",
        componentID: componentId
      },
      payload: {
        dataRecords: [{
          dataStatus: "DS_OK",
          dsTypes: ["DS_TYPES_RSA_PUBLIC_KEY"],
          data: Buffer.from(publicKey).toString('base64'),
          encoding: "BASE64"
        }]
      }
    };
    
    this.websocket.send(JSON.stringify(setupRequest));
    
    // Wait for ACK
    const ack = await this.waitForAck(setupRequest.meta.requestID);
    
    if (ack.ackCode !== "ACK_OK") {
      throw new Error(`Setup failed: ${ack.description}`);
    }
    
    console.log("✓ Encryption setup successful for session");
  } catch (error) {
    console.error("✗ Encryption setup failed:", error);
    throw error;
  }
}
```

## Component Capability Detection

Before attempting to use encryption, applications should verify the component supports this feature:

```javascript
// Check component characteristics for encryption support
function supportsEncryption(componentCharacteristics) {
  if (!componentCharacteristics.dsTypesList) {
    return false;
  }
  
  return componentCharacteristics.dsTypesList.includes(
    "DS_TYPES_RSA_PUBLIC_KEY"
  ) && componentCharacteristics.dsTypesList.includes(
    "DS_TYPES_ENCRYPTED_DATA"
  );
}

// Usage
const component = getComponentById(42);
if (supportsEncryption(component.componentCharacteristics[0])) {
  console.log("✓ Component supports RSA encryption");
  // Will setup when application becomes ACTIVE
} else {
  console.log("✗ Component does not support encryption");
  // Use standard unencrypted flow
}
```

## Complete Workflow Example

```javascript
class SecurePaymentFlow {
  constructor(deviceID, oauthToken, websocket) {
    this.deviceID = deviceID;
    this.oauthToken = oauthToken;
    this.websocket = websocket;
    this.sessionKeyPair = null;
    this.currentState = 'AVAILABLE';
  }
  
  // Handle application state changes
  async onApplicationStateChange(newState) {
    console.log(`State: ${this.currentState} → ${newState}`);
    
    if (newState === 'ACTIVE') {
      await this.initializeActiveSession();
    } else if (this.currentState === 'ACTIVE') {
      this.cleanupSession();
    }
    
    this.currentState = newState;
  }
  
  // Initialize new ACTIVE session
  async initializeActiveSession() {
    console.log('═══ New Session Started ═══');
    
    // 1. Generate fresh keys
    this.sessionKeyPair = this.generateKeys();
    
    // 2. Setup all encrypted components
    await this.setupEncryption(42); // MSR component
    
    // 3. Ready for operations
    console.log('✓ Session initialized with encryption');
  }
  
  // Clean up when leaving ACTIVE
  cleanupSession() {
    console.log('═══ Session Ended ═══');
    
    // Clear sensitive key material
    this.sessionKeyPair = null;
    
    console.log('✓ Session keys cleared');
  }
  
  // Generate key pair for current session
  generateKeys() {
    const keyPair = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
    });
    
    console.log("✓ Generated session RSA key pair");
    return keyPair;
  }
  
  // Setup encryption for component
  async setupEncryption(componentId) {
    if (this.currentState !== 'ACTIVE') {
      throw new Error('Cannot setup encryption: not in ACTIVE state');
    }
    
    const request = {
      meta: {
        deviceID: this.deviceID,
        requestID: crypto.randomUUID(),
        oauthToken: this.oauthToken,
        directive: "peripherals_setup",
        componentID: componentId
      },
      payload: {
        dataRecords: [{
          dataStatus: "DS_OK",
          dsTypes: ["DS_TYPES_RSA_PUBLIC_KEY"],
          data: Buffer.from(this.sessionKeyPair.publicKey).toString('base64'),
          encoding: "BASE64"
        }]
      }
    };
    
    this.websocket.send(JSON.stringify(request));
    console.log(`✓ Configured encryption for component ${componentId}`);
  }
  
  // Enable reader for card swipe
  async enableReader(componentId) {
    const request = {
      meta: {
        deviceID: this.deviceID,
        requestID: crypto.randomUUID(),
        oauthToken: this.oauthToken,
        directive: "peripherals_userpresent_enable",
        componentID: componentId
      },
      payload: {}
    };
    
    this.websocket.send(JSON.stringify(request));
    console.log("✓ Enabled mag stripe reader");
  }
  
  // Handle encrypted data from platform
  handleDataPresent(platformData) {
    const record = platformData.payload.dataRecords[0];
    
    if (!record.dsTypes.includes("DS_TYPES_ENCRYPTED_DATA")) {
      console.warn("⚠ Received unencrypted data");
      return null;
    }
    
    console.log("✓ Received encrypted payment data");
    
    // Decrypt with current session key
    const encryptedBuffer = Buffer.from(record.data, 'base64');
    const decryptedBuffer = crypto.privateDecrypt(
      {
        key: this.sessionKeyPair.privateKey,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      encryptedBuffer
    );
    
    const magStripeData = decryptedBuffer.toString('utf8');
    console.log("✓ Decrypted payment data");
    
    return this.processPayment(magStripeData);
  }
  
  processPayment(magStripeData) {
    console.log("✓ Processing payment...");
    // ... payment processing logic ...
    return { success: true };
  }
}

// Application lifecycle usage
const flow = new SecurePaymentFlow(deviceID, token, websocket);

// Platform activates application
await flow.onApplicationStateChange('ACTIVE');
// Keys generated, encryption setup automatically

// Begin transaction
await flow.enableReader(42);

// Handle incoming data
websocket.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.meta?.messageCode === "DATA_PRESENT") {
    flow.handleDataPresent(data);
  }
};

// Transaction complete - app goes AVAILABLE
await flow.onApplicationStateChange('AVAILABLE');
// Keys automatically cleared

// Next passenger - app becomes ACTIVE again
await flow.onApplicationStateChange('ACTIVE');
// NEW keys generated, NEW encryption setup
```

## Testing and Validation

### Unit Test Example

```javascript
describe('Session-Based RSA-OAEP Encryption', () => {
  it('should use different keys for each session', () => {
    // Session 1
    const keyPair1 = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
    });
    
    // Session 2 (new passenger)
    const keyPair2 = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
    });
    
    // Keys must be different
    expect(keyPair1.publicKey).not.toBe(keyPair2.publicKey);
    expect(keyPair1.privateKey).not.toBe(keyPair2.privateKey);
    
    // Each session's key can only decrypt its own data
    const testData = "%B4111111111111111^DOE/JOHN^2512101?";
    
    const encrypted1 = crypto.publicEncrypt(
      {
        key: keyPair1.publicKey,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      Buffer.from(testData)
    );
    
    // Session 2 key cannot decrypt Session 1 data
    expect(() => {
      crypto.privateDecrypt(
        {
          key: keyPair2.privateKey,
          padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
          oaepHash: 'sha256'
        },
        encrypted1
      );
    }).toThrow();
    
    // Session 1 key can decrypt Session 1 data
    const decrypted1 = crypto.privateDecrypt(
      {
        key: keyPair1.privateKey,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      encrypted1
    );
    
    expect(decrypted1.toString('utf8')).toBe(testData);
  });
});
```

## Backward Compatibility

This encryption feature is **optional and backward compatible**:

- **New applications** can use encryption with compatible platforms
- **Existing applications** continue to work without encryption
- **Platforms** that don't support encryption return unencrypted data
- **No breaking changes** to existing CUSS2 interfaces

### Graceful Degradation

```javascript
async function initializeSession(state) {
  if (state === 'ACTIVE') {
    const component = getComponentById(42);
    
    if (supportsEncryption(component)) {
      // Use encrypted flow with session keys
      const keyPair = generateSessionKeyPair();
      await setupEncryptedComponent(42, keyPair.publicKey);
      console.log("✓ Using encrypted session");
    } else {
      // Fall back to standard flow
      console.log("ℹ Using standard unencrypted flow");
    }
    
    await enableComponent(42);
  }
}
```

## Summary: Key Lifecycle Rules

1. **Generate keys** when application becomes ACTIVE
2. **Send public key** via `peripherals_setup` for each encrypted component
3. **Use encryption** during ACTIVE session
4. **Clear keys** when transitioning away from ACTIVE
5. **Repeat** for next session with NEW keys

This session-based approach ensures maximum security with zero key management burden on the platform.
