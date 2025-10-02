# Authentication

## Table of Contents

- [Overview](#overview)
- [Core Concepts](#core-concepts)
- [Authentication Methods](#authentication-methods)
- [Session Management](#session-management)
- [Multi-Factor Authentication (MFA)](#multi-factor-authentication-mfa)
- [Single Sign-On (SSO)](#single-sign-on-sso)
- [Modern Authentication Protocols](#modern-authentication-protocols)
- [Best Practices](#best-practices)
- [Common Vulnerabilities](#common-vulnerabilities)

## Overview

Authentication is the process of verifying the identity of a user, device, or system. It answers the question: "Who are you?" and ensures that entities are who they claim to be before granting access to resources.

```mermaid
graph LR
    A[User Claims Identity] --> B[Authentication System]
    B --> C{Verify Credentials}
    C -->|Valid| D[Grant Access Token]
    C -->|Invalid| E[Deny Access]
    D --> F[Access Protected Resources]
    
    style C fill:#FFD700
    style D fill:#90EE90
    style E fill:#FFB6C6
```

## Core Concepts

### Authentication vs Authorization

```mermaid
flowchart TD
    Start[User Request] --> Auth[Authentication]
    Auth --> AuthQ{Who are you?}
    AuthQ -->|Verified| Authz[Authorization]
    AuthQ -->|Failed| Deny1[Access Denied]
    Authz --> AuthzQ{What can you do?}
    AuthzQ -->|Permitted| Access[Grant Access]
    AuthzQ -->|Not Permitted| Deny2[Access Denied]
    
    style Auth fill:#E1F5FE
    style Authz fill:#FFF3E0
```

### Authentication Factors

```mermaid
mindmap
  root((Authentication<br/>Factors))
    Something You Know
      Password
      PIN
      Security Questions
    Something You Have
      Hardware Token
      Mobile Device
      Smart Card
    Something You Are
      Fingerprint
      Face Recognition
      Iris Scan
    Somewhere You Are
      GPS Location
      IP Address
      Network
    Something You Do
      Typing Pattern
      Behavior Analysis
      Gait Recognition
```

## Authentication Methods

### 1. Password-Based Authentication

The most common but increasingly vulnerable method.

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client
    participant S as Server
    participant DB as Database
    
    U->>C: Enter username & password
    C->>S: POST /login {username, password}
    S->>DB: Query user by username
    DB-->>S: Return hashed password + salt
    S->>S: Hash input password + salt
    S->>S: Compare hashes
    alt Match
        S->>S: Generate session token
        S-->>C: Return token + user data
        C-->>U: Login successful
    else No Match
        S-->>C: 401 Unauthorized
        C-->>U: Invalid credentials
    end
```

**Implementation Example:**

```javascript
// Password hashing (registration)
const bcrypt = require('bcrypt');

async function registerUser(username, password) {
  const saltRounds = 12;
  const hashedPassword = await bcrypt.hash(password, saltRounds);
  
  // Store hashedPassword in database
  await db.users.create({
    username,
    password: hashedPassword
  });
}

// Password verification (login)
async function authenticateUser(username, password) {
  const user = await db.users.findOne({ username });
  if (!user) return null;
  
  const isValid = await bcrypt.compare(password, user.password);
  return isValid ? user : null;
}
```

**Best Practices:**
- Never store passwords in plain text
- Use strong hashing algorithms (bcrypt, Argon2, scrypt)
- Implement password complexity requirements
- Use rate limiting to prevent brute force attacks

### 2. Token-Based Authentication

Modern approach using stateless tokens (JWT, OAuth tokens).

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Server
    participant R as Resource Server
    
    C->>A: POST /login (credentials)
    A->>A: Validate credentials
    A->>A: Generate JWT
    A-->>C: Return JWT
    
    Note over C: Store JWT securely
    
    C->>R: GET /api/data<br/>Authorization: Bearer JWT
    R->>R: Verify JWT signature
    R->>R: Check expiration
    R->>R: Extract user claims
    R-->>C: Return protected data
```

**JWT Structure:**

```mermaid
graph LR
    A[Header] -->|.| B[Payload]
    B -->|.| C[Signature]
    
    A1[Algorithm<br/>Type] -.-> A
    B1[User Data<br/>Claims<br/>Expiration] -.-> B
    C1[HMAC/RSA<br/>Signature] -.-> C
    
    style A fill:#E3F2FD
    style B fill:#FFF3E0
    style C fill:#F3E5F5
```

**Implementation Example:**

```javascript
const jwt = require('jsonwebtoken');
const SECRET_KEY = process.env.JWT_SECRET;

// Generate token
function generateToken(user) {
  const payload = {
    userId: user.id,
    username: user.username,
    role: user.role
  };
  
  return jwt.sign(payload, SECRET_KEY, {
    expiresIn: '1h',
    issuer: 'your-app',
    audience: 'your-app-users'
  });
}

// Verify token middleware
function verifyToken(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, SECRET_KEY);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

### 3. Biometric Authentication

Uses unique biological characteristics.

```mermaid
flowchart TD
    A[Biometric Capture] --> B{Type}
    B -->|Fingerprint| C[Fingerprint Scanner]
    B -->|Face| D[Face Recognition]
    B -->|Iris| E[Iris Scanner]
    B -->|Voice| F[Voice Analysis]
    
    C --> G[Extract Features]
    D --> G
    E --> G
    F --> G
    
    G --> H[Compare with Stored Template]
    H --> I{Match Score > Threshold?}
    I -->|Yes| J[Authenticate]
    I -->|No| K[Reject]
    
    style J fill:#90EE90
    style K fill:#FFB6C6
```

### 4. Certificate-Based Authentication

Uses digital certificates and PKI.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant CA as Certificate Authority
    
    Note over C,CA: Certificate Issuance (One-time)
    C->>CA: Request certificate
    CA->>CA: Verify identity
    CA-->>C: Issue signed certificate
    
    Note over C,S: Authentication Flow
    C->>S: Initiate connection
    S->>C: Request certificate
    C->>S: Send client certificate
    S->>S: Verify certificate signature
    S->>CA: Check revocation status
    CA-->>S: Certificate valid
    S-->>C: Connection established
```

## Session Management

### Session vs Token Comparison

```mermaid
graph TB
    subgraph "Session-Based"
        A1[Client] --> B1[Login]
        B1 --> C1[Server Creates Session]
        C1 --> D1[Session ID in Cookie]
        D1 --> E1[Store Session in Server/DB]
        E1 --> F1[Validate on Each Request]
    end
    
    subgraph "Token-Based"
        A2[Client] --> B2[Login]
        B2 --> C2[Server Creates JWT]
        C2 --> D2[Token Sent to Client]
        D2 --> E2[Client Stores Token]
        E2 --> F2[Validate Token Locally]
    end
    
    style C1 fill:#FFE0B2
    style C2 fill:#C8E6C9
```

### Session Implementation

```javascript
// Express session example
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,        // HTTPS only
    httpOnly: true,      // Not accessible via JavaScript
    maxAge: 3600000,     // 1 hour
    sameSite: 'strict'   // CSRF protection
  }
}));

// Login route
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body.username, req.body.password);
  
  if (user) {
    req.session.userId = user.id;
    req.session.role = user.role;
    res.json({ success: true });
  } else {
    res.status(401).json({ error: 'Invalid credentials' });
  }
});
```

### Session Security

```mermaid
graph TD
    A[Session Security] --> B[Secure Cookie Flags]
    A --> C[Session Timeout]
    A --> D[Session Regeneration]
    A --> E[Session Invalidation]
    
    B --> B1[httpOnly: true]
    B --> B2[secure: true]
    B --> B3[sameSite: strict]
    
    C --> C1[Idle Timeout]
    C --> C2[Absolute Timeout]
    
    D --> D1[After Login]
    D --> D2[Privilege Escalation]
    
    E --> E1[On Logout]
    E --> E2[On Security Event]
    
    style A fill:#FFD700
```

## Multi-Factor Authentication (MFA)

### MFA Flow

```mermaid
sequenceDiagram
    participant U as User
    participant A as Auth Server
    participant T as TOTP/SMS Service
    
    U->>A: Submit username & password
    A->>A: Verify credentials
    alt Credentials Valid
        A->>T: Request 2FA code
        T->>U: Send OTP (SMS/App)
        U->>A: Submit OTP
        A->>A: Verify OTP
        alt OTP Valid
            A-->>U: Grant access + token
        else OTP Invalid
            A-->>U: 401 - Invalid OTP
        end
    else Credentials Invalid
        A-->>U: 401 - Invalid credentials
    end
```

### TOTP Implementation

```javascript
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');

// Generate secret for user
function generateMFASecret(username) {
  const secret = speakeasy.generateSecret({
    name: `YourApp (${username})`,
    length: 32
  });
  
  return {
    secret: secret.base32,
    qrCode: secret.otpauth_url
  };
}

// Generate QR code
async function generateQRCode(otpauth_url) {
  return await QRCode.toDataURL(otpauth_url);
}

// Verify TOTP token
function verifyTOTP(token, secret) {
  return speakeasy.totp.verify({
    secret: secret,
    encoding: 'base32',
    token: token,
    window: 2  // Allow 2 time steps before/after
  });
}

// MFA verification middleware
async function verifyMFA(req, res, next) {
  const { token } = req.body;
  const user = await db.users.findById(req.user.id);
  
  if (!user.mfaEnabled) {
    return next();
  }
  
  if (verifyTOTP(token, user.mfaSecret)) {
    next();
  } else {
    res.status(401).json({ error: 'Invalid MFA token' });
  }
}
```

### MFA Methods Comparison

```mermaid
graph TB
    subgraph "MFA Methods"
        A[SMS OTP]
        B[TOTP Apps]
        C[Hardware Tokens]
        D[Biometric]
        E[Backup Codes]
    end
    
    subgraph "Security Level"
        A -.-> L1[Low]
        B -.-> L2[Medium-High]
        C -.-> L3[High]
        D -.-> L2
        E -.-> L1
    end
    
    subgraph "User Convenience"
        A -.-> U1[High]
        B -.-> U2[Medium]
        C -.-> U3[Low]
        D -.-> U1
        E -.-> U2
    end
    
    style L3 fill:#90EE90
    style L1 fill:#FFB6C6
```

## Single Sign-On (SSO)

### SSO Architecture

```mermaid
graph TB
    U[User] --> IDP[Identity Provider]
    IDP --> SP1[Service Provider 1]
    IDP --> SP2[Service Provider 2]
    IDP --> SP3[Service Provider 3]
    
    IDP -->|SAML/OAuth| SP1
    IDP -->|SAML/OAuth| SP2
    IDP -->|SAML/OAuth| SP3
    
    style IDP fill:#FFD700
    style SP1 fill:#E3F2FD
    style SP2 fill:#E3F2FD
    style SP3 fill:#E3F2FD
```

### SSO Flow (SAML)

```mermaid
sequenceDiagram
    participant U as User
    participant SP as Service Provider
    participant IDP as Identity Provider
    
    U->>SP: Access protected resource
    SP->>SP: No valid session
    SP->>U: Redirect to IDP (SAML Request)
    U->>IDP: Forward SAML Request
    
    alt Not Authenticated
        IDP->>U: Show login form
        U->>IDP: Submit credentials
    end
    
    IDP->>IDP: Authenticate user
    IDP->>IDP: Create SAML Assertion
    IDP->>U: Redirect to SP (SAML Response)
    U->>SP: Forward SAML Response
    SP->>SP: Validate SAML Assertion
    SP->>SP: Create session
    SP-->>U: Grant access to resource
```

## Modern Authentication Protocols

### OAuth 2.0 Flow

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client App
    participant AS as Authorization Server
    participant RS as Resource Server
    
    U->>C: Click "Login with OAuth"
    C->>AS: Authorization Request
    AS->>U: Show consent screen
    U->>AS: Grant permission
    AS->>C: Authorization Code
    C->>AS: Exchange code for token
    AS-->>C: Access Token + Refresh Token
    C->>RS: Request resource (with token)
    RS->>RS: Validate token
    RS-->>C: Protected resource
```

### OpenID Connect (OIDC)

```mermaid
graph LR
    A[OAuth 2.0] --> B[+ Identity Layer]
    B --> C[OpenID Connect]
    
    C --> D[ID Token]
    C --> E[UserInfo Endpoint]
    C --> F[Standard Claims]
    
    D --> G[JWT with user info]
    E --> H[Additional user data]
    F --> I[email, name, picture, etc.]
    
    style C fill:#FFD700
```

**OIDC Implementation:**

```javascript
const { Issuer } = require('openid-client');

// Discover OIDC provider
async function setupOIDC() {
  const issuer = await Issuer.discover('https://accounts.google.com');
  
  const client = new issuer.Client({
    client_id: process.env.CLIENT_ID,
    client_secret: process.env.CLIENT_SECRET,
    redirect_uris: ['http://localhost:3000/callback'],
    response_types: ['code']
  });
  
  return client;
}

// Generate authorization URL
function getAuthorizationUrl(client) {
  return client.authorizationUrl({
    scope: 'openid email profile',
    state: generateRandomState()
  });
}

// Handle callback
async function handleCallback(client, params) {
  const tokenSet = await client.callback(
    'http://localhost:3000/callback',
    params
  );
  
  const claims = tokenSet.claims();
  
  return {
    accessToken: tokenSet.access_token,
    idToken: tokenSet.id_token,
    user: {
      id: claims.sub,
      email: claims.email,
      name: claims.name
    }
  };
}
```

### Passwordless Authentication

```mermaid
flowchart TD
    A[User Enters Email] --> B[System Sends Magic Link]
    B --> C[User Clicks Link]
    C --> D{Verify Token}
    D -->|Valid & Not Expired| E[Authenticate User]
    D -->|Invalid/Expired| F[Request New Link]
    E --> G[Create Session]
    
    A2[Alternative: WebAuthn] --> B2[User Registers Device]
    B2 --> C2[Browser Generates Key Pair]
    C2 --> D2[Store Public Key]
    D2 --> E2[User Authenticates]
    E2 --> F2[Sign Challenge with Private Key]
    F2 --> G2[Verify Signature]
    
    style E fill:#90EE90
    style G2 fill:#90EE90
```

**Magic Link Implementation:**

```javascript
const crypto = require('crypto');

// Generate magic link
async function generateMagicLink(email) {
  const token = crypto.randomBytes(32).toString('hex');
  const expiresAt = new Date(Date.now() + 15 * 60 * 1000); // 15 minutes
  
  await db.magicTokens.create({
    email,
    token,
    expiresAt,
    used: false
  });
  
  const magicLink = `https://yourapp.com/auth/verify?token=${token}`;
  await sendEmail(email, magicLink);
  
  return { success: true };
}

// Verify magic link
async function verifyMagicLink(token) {
  const record = await db.magicTokens.findOne({ token });
  
  if (!record || record.used || record.expiresAt < new Date()) {
    return null;
  }
  
  await db.magicTokens.update({ token }, { used: true });
  
  return record.email;
}
```

## Best Practices

### Authentication Security Checklist

```mermaid
mindmap
  root((Authentication<br/>Security))
    Password Policy
      12+ characters
      Complexity rules
      No common passwords
      Regular rotation
    Credential Storage
      Hash passwords
      Use bcrypt/Argon2
      Salt per user
      Never log passwords
    Session Security
      Secure cookies
      Session timeout
      Regenerate on login
      HTTPS only
    Rate Limiting
      Login attempts
      API calls
      Token generation
      Account lockout
    MFA
      Enforce for admins
      Optional for users
      Multiple methods
      Backup codes
    Monitoring
      Failed attempts
      Unusual locations
      Concurrent sessions
      Privilege changes
```

### Implementation Guidelines

1. **Always use HTTPS** - No authentication over plain HTTP
2. **Implement rate limiting** - Prevent brute force attacks
3. **Use secure password hashing** - bcrypt, Argon2, or scrypt
4. **Enable MFA** - Especially for privileged accounts
5. **Implement proper session management** - Timeouts and regeneration
6. **Use secure tokens** - JWT with proper validation
7. **Monitor authentication events** - Log failures and anomalies
8. **Regular security audits** - Test authentication mechanisms

## Common Vulnerabilities

### OWASP Authentication Risks

```mermaid
graph TD
    A[Authentication Vulnerabilities] --> B[Broken Authentication]
    A --> C[Session Management Flaws]
    A --> D[Credential Stuffing]
    A --> E[Brute Force Attacks]
    
    B --> B1[Weak passwords]
    B --> B2[Default credentials]
    B --> B3[Exposed session IDs]
    
    C --> C1[No session timeout]
    C --> C2[Session fixation]
    C --> C3[Predictable session IDs]
    
    D --> D1[Leaked password databases]
    D --> D2[No rate limiting]
    
    E --> E1[No account lockout]
    E --> E2[Weak password policy]
    
    style A fill:#FFB6C6
```

### Protection Strategies

```javascript
// Rate limiting example
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many login attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    // Log security event
    logger.warn('Rate limit exceeded', {
      ip: req.ip,
      username: req.body.username
    });
    res.status(429).json({ error: 'Too many attempts' });
  }
});

app.post('/login', loginLimiter, async (req, res) => {
  // Login logic
});

// Account lockout after failed attempts
async function handleFailedLogin(username) {
  const user = await db.users.findOne({ username });
  if (!user) return;
  
  user.failedAttempts = (user.failedAttempts || 0) + 1;
  user.lastFailedAttempt = new Date();
  
  if (user.failedAttempts >= 5) {
    user.lockedUntil = new Date(Date.now() + 30 * 60 * 1000); // 30 min
    await sendSecurityAlert(user.email, 'Account locked');
  }
  
  await user.save();
}
```

### Security Monitoring

```javascript
// Authentication event logging
function logAuthEvent(event, user, req) {
  logger.info('Authentication event', {
    event,
    userId: user?.id,
    username: user?.username,
    ip: req.ip,
    userAgent: req.get('user-agent'),
    timestamp: new Date(),
    success: event.includes('success')
  });
}

// Detect anomalies
async function detectAnomalies(user, req) {
  const recentLogins = await db.authLogs.find({
    userId: user.id,
    timestamp: { $gt: new Date(Date.now() - 24 * 60 * 60 * 1000) }
  });
  
  // Check for unusual location
  const currentLocation = getLocationFromIP(req.ip);
  const usualLocations = recentLogins.map(l => l.location);
  
  if (!usualLocations.includes(currentLocation)) {
    await sendSecurityAlert(user.email, 'Login from new location');
  }
}
```

---

## Related Documentation

- [Authorization](./authorization.md) - Access control after authentication
- [Encryption](./encryption.md) - Protecting credentials in transit and at rest
- [Network Security](./network_security.md) - Securing authentication channels
- [Monitoring & Auditing](./monitoring_auditing.md) - Tracking authentication events
- [Best Practices](./best_practises.md) - Overall security guidelines

---

**Remember**: Authentication is the first line of defense. Implement it carefully, monitor it continuously, and update it regularly to address emerging threats.