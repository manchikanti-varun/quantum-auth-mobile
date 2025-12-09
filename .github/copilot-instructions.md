# Copilot Instructions — QSafe Mobile App

## Project Overview
**QSafe** is a quantum-safe mobile authenticator built with React Native (Expo managed) + TypeScript. Core features:
- Secure local account storage (encrypted TOTP secrets via `expo-secure-store`)
- QR code scanning for otpauth:// enrollment
- Real-time TOTP generator (RFC 6238, 30-second refresh)
- Push notification handling for MFA challenges
- PQC key generation & challenge signing (Dilithium)
- Biometric authentication support

## Architecture

### Core Tech Stack
- **Framework**: React Native (Expo managed) + Expo Router (file-based routing)
- **Language**: TypeScript (`tsconfig.json` configured)
- **State & Navigation**: React Navigation (bottom tabs + modal)
- **UI Components**: Custom themed components + Expo Vector Icons
- **Security**: `expo-secure-store` (AES-256), `expo-local-authentication` (biometric)
- **Device Communication**: Firebase FCM via `expo-notifications`

### File-Based Routing (`app/` directory)
```
app/
  _layout.tsx      → Root layout (tabs + modal stack)
  modal.tsx        → Modal for account details/add account
  (tabs)/
    _layout.tsx    → Tab layout (index, explore)
    index.tsx      → Home: list accounts + TOTP display
    explore.tsx    → Settings/help tab
```

### Data Flow (Account Enrollment)
```
1. User: "Add Account" → Camera open
2. Scan QR (otpauth://?secret=XXX&issuer=ISSUER&email=EMAIL)
3. Extract fields → Generate Dilithium keypair locally
4. Store encrypted secret in SecureStore with account metadata
5. Register public key with backend
6. Display account in list with real-time TOTP updates
```

### Data Flow (Push Challenge)
```
1. Firebase FCM message arrives → expo-notifications listener
2. Extract challenge_id, nonce, expiry
3. Show "Approve/Reject?" prompt
4. Approve: Sign challenge with Dilithium private key
5. Send signature to backend
6. Show success/failure feedback
```

## Key Conventions & Patterns

### Environment Setup
- **Dev**: `npm start` (Expo start with dev client option)
- **Test on device**: `npm run android` or `npm run ios`
- **Web preview**: `npm run web` (limited support for crypto)
- **Linting**: `npm run lint` (ESLint + Expo rules)

### Component Structure
- **Functional components** with hooks (React 19)
- **Themed components** in `components/themed-*` (use theme context for colors)
- **Custom hooks** in `hooks/`: `useColorScheme()`, `useThemeColor()` for theme management
- **UI module** in `components/ui/`: Collapsible, IconSymbol (platform-specific)

### Secure Storage Pattern
- Store TOTP secrets via `expo-secure-store`:
  ```typescript
  import * as SecureStore from 'expo-secure-store';
  
  // Save (AES-256 encrypted on device)
  await SecureStore.setItemAsync(`account_${issuer}_secret`, encryptedSecret);
  
  // Retrieve
  const secret = await SecureStore.getItemAsync(`account_${issuer}_secret`);
  ```
- **Never** log secrets or keys to console
- Retrieve secrets on app startup + display TOTP in UI

### QR Code Scanning
- Use `expo-barcode-scanner` with `BarCodeScanner.TYPE_QR_CODE`
- Parse otpauth:// URI:
  ```typescript
  // otpauth://totp/issuer:email?secret=BASE32SECRET&issuer=ISSUER
  const url = new URL(data.data); // QR scan result
  const secret = url.searchParams.get('secret');
  const issuer = url.searchParams.get('issuer');
  const email = url.pathname.split(':')[1];
  ```
- Validate secret (base32), issuer, email before storing
- Show confirmation + account name before final save

### PQC Key Management
- **Initialization**: Generate Dilithium keypair on first app launch
  ```typescript
  import dilithium from 'dilithium-crystals-js';
  const keypair = dilithium.generateKeyPair();
  // Store private key in SecureStore (Phase 2+)
  // Send public key to backend during device linking
  ```
- **Challenge Signing**:
  ```typescript
  const signature = dilithium.sign(challenge_bytes, private_key);
  // Send signature to backend for verification
  ```
- Private keys stored in SecureStore only; public keys sent to backend

### TOTP Generation (RFC 6238)
- Use base32-decoded secret + current time
- 30-second window, 6-digit code
- Refresh every second for countdown display
- Example pattern:
  ```typescript
  const generateTOTP = (secret: string): string => {
    const epoch = Math.floor(Date.now() / 1000 / 30);
    const hmac = createHmac('sha1', base32decode(secret));
    hmac.update(Buffer.from(epoch.toString('hex'), 'hex'));
    const digest = hmac.digest();
    const offset = digest[19] & 0x0f;
    const code = (digest.readUInt32BE(offset) & 0x7fffffff) % 1000000;
    return code.toString().padStart(6, '0');
  };
  ```

### Navigation Patterns
- **Tab Navigation**: `index.tsx` (accounts) + `explore.tsx` (settings)
- **Modal for Details**: Modal triggered from account item → show TOTP, approve/reject
- **Deep Linking**: Use `expo-linking` for Firebase notification actions (future)

### Biometric Authentication
- Optional unlock for viewing secrets
- Fallback to PIN if biometric unavailable
- Use `expo-local-authentication`:
  ```typescript
  import * as LocalAuthentication from 'expo-local-authentication';
  
  const biometricAvailable = await LocalAuthentication.hasHardwareAsync();
  const result = await LocalAuthentication.authenticateAsync({
    disableDeviceFallback: false,
  });
  ```

## Critical Files & Patterns
- **`app/_layout.tsx`**: Root navigation + tab setup + modal stack
- **`app/(tabs)/index.tsx`**: Main account list + real-time TOTP display
- **`constants/theme.ts`**: Color scheme + design tokens
- **`hooks/use-color-scheme.ts`**: Theme context + dark/light mode
- **`package.json`**: Real PQC libraries (`dilithium-crystals-js`), Firebase SDK, Expo modules

## Phase 1 Deliverables (Current Focus)
- ✅ Expo managed project scaffold + file-based routing
- ✅ Tab navigation (home, explore)
- ✅ Themed components + color scheme
- ⚠️ QR code scanning (BarCodeScanner setup + otpauth:// parsing)
- ⚠️ SecureStore integration (TOTP secret storage)
- ⚠️ Basic account list UI
- ❌ Real TOTP generation (Phase 2)
- ❌ PQC key generation + enrollment (Phase 2)
- ❌ Push notification handling + challenge signing (Phase 3+)
- ❌ Biometric unlock (Phase 3+)

## Common Tasks
- **Add new screen**: Create in `app/` with `_layout.tsx` for routes
- **Update theme**: Modify `constants/theme.ts` + use `useThemeColor()` hook
- **Add component**: Create in `components/` with themed variants (dark/light)
- **Test QR scanning**: Use `expo-barcode-scanner` with simulator camera mock
- **Secure store operations**: Always wrap in try-catch, handle errors gracefully
- **Add push listener**: Register in root layout using `expo-notifications` + handle challenge data
- **Debug theme issues**: Check `useColorScheme()` returns correct scheme
