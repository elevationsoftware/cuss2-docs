# CUSS2 RSA-OAEP Encryption for Sensitive Data

## Overview

This section describes the RSA-OAEP public-key encryption feature for securing sensitive data transmitted between CUSS applications and platforms. This approach allows applications to provide a public key during component setup, enabling the platform to encrypt sensitive data (such as magnetic stripe payment card data) before transmission back to the application.

**Key Benefits:**
- End-to-end encryption of sensitive cardholder data
- Platform never accesses or stores private keys
- Compliant with PCI-DSS requirements
- Zero additional key management burden on platforms
- Application controls complete security lifecycle

## Conceptual Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RSA-OAEP Encryption Flow                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────┐                                    ┌──────────────┐
│ Application │                                    │   Platform   │
└──────┬──────┘                                    └──────┬───────┘
       │                                                  │
       │ 1. Generate RSA Key Pair                         │
       │    (2048-bit or 4096-bit)                        │
       │                                                  │
       │ 2. peripherals_setup                             │
       │    + Public Key (DS_TYPES_RSA_PUBLIC_KEY)        │
       ├─────────────────────────────────────────────────>│
       │                                                  │ Store public key
       │ 3. ACK_OK                                        │ for component
       │<─────────────────────────────────────────────────┤
       │                                                  │
       │ 4. peripherals_userpresent_enable                │
       ├─────────────────────────────────────────────────>│
       │                                                  │
       │                          [User swipes card]      │
       │                                                  │ Read mag stripe
       │                                                  │ Encrypt with
       │ 5. SOLICITED Event                               │ RSA-OAEP
       │    DATA_PRESENT                                  │
       │    Encrypted data in dataRecords                 │
       │<─────────────────────────────────────────────────┤
       │                                                  │
       │ 6. Decrypt with private key                      │
       │    Process payment data                          │
       │                                                  │
```

## Technical Implementation

### Step 1: Application Generates RSA Key Pair

The application generates an RSA key pair before sending the setup directive. The recommended key size is **2048 bits** for a balance of security and performance, though **4096 bits** may be used for enhanced security.

**Key Generation Example (JavaScript/Node.js):**
```javascript
const crypto = require('crypto');

// Generate RSA key pair
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

// Store private key securely (never transmit!)
// Public key will be sent to platform
```

**Key Generation Example (Python):**
```python
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend

# Generate RSA key pair
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

# Store private_key securely (never transmit!)
# public_key_pem will be sent to platform
```

### Step 2: Setup Directive with Public Key

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
| `dsTypes` | Must include `DS_TYPES_RSA_PUBLIC_KEY` to indicate this is an RSA public key |
| `data` | Base64-encoded PEM format public key |
| `encoding` | Must be `BASE64` |
| `dataStatus` | Should be `DS_OK` for valid key data |

### Step 3: Platform Acknowledgment

The platform validates and stores the public key, responding with `ACK_OK`:

```json
{
  "ackCode": "ACK_OK",
  "requestID": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "description": "Public key stored successfully for component 42"
}
```

**Possible Error Responses:**

| ACK Code | Reason |
|----------|--------|
| `ACK_PARAMETER` | Invalid public key format, encoding error, or missing data |
| `ACK_ERROR` | Component doesn't support encryption or internal platform error |

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
    },
    "platformDirective": ""
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

The application uses its private key to decrypt the received data:

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
    console.log('Decrypted mag stripe:', magStripeData);
    
    return magStripeData;
  } catch (error) {
    console.error('Decryption failed:', error);
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

def decrypt_mag_stripe_data(encrypted_data_b64, private_key_pem):
    try:
        # Load private key
        private_key = serialization.load_pem_private_key(
            private_key_pem.encode('utf-8'),
            password=None,
            backend=default_backend()
        )
        
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
        print(f"Decrypted mag stripe: {mag_stripe_data}")
        
        return mag_stripe_data
    except Exception as e:
        print(f"Decryption failed: {e}")
        raise

# Usage
encrypted_data = "kQp8xNZ2L5vHtJ9mW3rF0eK7cY1bX4dP6sA8nM9gV2lR5oU7iT0qE3wZ..."
decrypted_data = decrypt_mag_stripe_data(encrypted_data, private_key_pem)
# Result: "%B4111111111111111^DOE/JOHN^2512101123456789?"
```

## Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                     Encryption Data Flow                         │
└──────────────────────────────────────────────────────────────────┘

Application Side                    Platform Side
─────────────────                   ──────────────

┌─────────────┐
│  Generate   │
│  Key Pair   │
└──────┬──────┘
       │
       │ Public Key (PEM)
       ↓
┌─────────────┐                    ┌──────────────┐
│   Base64    │──────────────────→ │    Store     │
│   Encode    │   peripherals_     │  Public Key  │
└─────────────┘      setup         │ (component)  │
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
│ Private Key │    (Base64)
└──────┬──────┘
       │
       ↓
┌─────────────┐
│   Process   │
│  Plain Text │
│   Payment   │
└─────────────┘
```

## Security Considerations

### Key Management Best Practices

1. **Private Key Storage:**
   - Store private keys in secure, encrypted storage
   - Never transmit private keys over the network
   - Use hardware security modules (HSMs) when available
   - Implement proper access controls

2. **Key Rotation:**
   - Implement periodic key rotation policies
   - Generate new key pairs for each session or daily
   - Retire old keys securely after rotation

3. **Key Size:**
   - Minimum: 2048 bits (recommended for most deployments)
   - Enhanced security: 4096 bits (higher security, slightly slower)

### Encryption Details

**Algorithm:** RSA-OAEP (Optimal Asymmetric Encryption Padding)

**Hash Function:** SHA-256

**Key Format:** PEM (Privacy-Enhanced Mail)

**Data Size Limits:**
- 2048-bit key: Maximum plaintext = 214 bytes (2048/8 - 42)
- 4096-bit key: Maximum plaintext = 470 bytes (4096/8 - 42)

**Performance:**
- Encryption overhead: ~2-10ms per operation
- Negligible impact for magnetic stripe operations

### PCI-DSS Compliance

This encryption approach supports PCI-DSS compliance by:
- Encrypting cardholder data in transit
- Preventing platform storage of unencrypted cardholder data
- Ensuring only the application can decrypt sensitive data
- Meeting "strong cryptography" requirements (RSA-2048+)

## Error Handling

### Common Errors and Solutions

| Error Scenario | Cause | Solution |
|----------------|-------|----------|
| ACK_PARAMETER on setup | Invalid public key format | Verify PEM format and Base64 encoding |
| Decryption fails | Wrong private key or corrupted data | Verify key pair matches, check data integrity |
| DATA_PRESENT without encryption | Platform doesn't support feature | Check component characteristics for encryption capability |
| Key size error | Key too small (<2048 bits) | Generate larger key pair (≥2048 bits) |

### Example Error Handling

```javascript
// Setup with error handling
async function setupEncryptedComponent(componentId, publicKey) {
  try {
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
    
    console.log("✓ Encryption setup successful");
  } catch (error) {
    console.error("✗ Encryption setup failed:", error);
    // Fallback: Continue without encryption if acceptable
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
  await setupEncryptedComponent(42, publicKey);
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
    this.keyPair = null;
  }
  
  // 1. Generate key pair
  generateKeys() {
    this.keyPair = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
    });
    console.log("✓ Generated RSA key pair");
  }
  
  // 2. Setup with public key
  async setupEncryption(componentId) {
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
          data: Buffer.from(this.keyPair.publicKey).toString('base64'),
          encoding: "BASE64"
        }]
      }
    };
    
    this.websocket.send(JSON.stringify(request));
    console.log("✓ Sent public key to platform");
  }
  
  // 3. Enable component
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
  
  // 4. Handle encrypted response
  handleDataPresent(platformData) {
    const record = platformData.payload.dataRecords[0];
    
    if (record.dsTypes.includes("DS_TYPES_ENCRYPTED_DATA")) {
      console.log("✓ Received encrypted data");
      
      // Decrypt
      const encryptedBuffer = Buffer.from(record.data, 'base64');
      const decryptedBuffer = crypto.privateDecrypt(
        {
          key: this.keyPair.privateKey,
          padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
          oaepHash: 'sha256'
        },
        encryptedBuffer
      );
      
      const magStripeData = decryptedBuffer.toString('utf8');
      console.log("✓ Decrypted payment data");
      
      return this.processPayment(magStripeData);
    }
  }
  
  processPayment(magStripeData) {
    // Process payment with decrypted data
    console.log("✓ Processing payment...");
    // ... payment processing logic ...
  }
}

// Usage
const flow = new SecurePaymentFlow(deviceID, token, websocket);
flow.generateKeys();
await flow.setupEncryption(42);
await flow.enableReader(42);

websocket.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.meta?.messageCode === "DATA_PRESENT") {
    flow.handleDataPresent(data);
  }
};
```

## Testing and Validation

### Unit Test Example

```javascript
describe('RSA-OAEP Encryption', () => {
  it('should encrypt and decrypt mag stripe data', () => {
    // Generate test key pair
    const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
    });
    
    // Test data
    const originalData = "%B4111111111111111^DOE/JOHN^2512101?";
    
    // Encrypt (simulating platform)
    const encrypted = crypto.publicEncrypt(
      {
        key: publicKey,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      Buffer.from(originalData)
    );
    
    // Decrypt (simulating application)
    const decrypted = crypto.privateDecrypt(
      {
        key: privateKey,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      encrypted
    );
    
    expect(decrypted.toString('utf8')).toBe(originalData);
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
async function readMagStripe(componentId) {
  const component = getComponentById(componentId);
  
  if (supportsEncryption(component)) {
    // Use encrypted flow
    await setupEncryptedComponent(componentId, publicKey);
  } else {
    // Fall back to standard flow
    console.log("Using standard unencrypted flow");
  }
  
  await enableComponent(componentId);
}
```

