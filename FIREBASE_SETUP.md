# QSafe Mobile Firebase Configuration

## 1. Firebase Project Setup

### Create Firebase Project
1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click "Create Project"
3. Name: `quantum-auth-qsafe`
4. Enable Google Analytics (optional)

### Add Mobile App to Firebase
1. Click "Add app" in Firebase Console
2. Choose "Android" or "iOS" (or both)
3. Register your app with package name:
   - Android: `com.qsafe.mobile` (or your custom package name)
   - iOS: `com.qsafe.mobile` (or your custom bundle ID)
4. Download configuration files:
   - Android: `google-services.json`
   - iOS: `GoogleService-Info.plist`

## 2. Install Firebase Dependencies

```bash
cd quantum-auth-mobile
expo install firebase
expo install expo-notifications
expo install expo-secure-store
```

## 3. Environment Variables

Create `.env` file in your mobile app root:

```bash
# Firebase Configuration
EXPO_PUBLIC_FIREBASE_API_KEY=your-api-key
EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
EXPO_PUBLIC_FIREBASE_PROJECT_ID=your-project-id
EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=your-sender-id
EXPO_PUBLIC_FIREBASE_APP_ID=your-app-id
EXPO_PUBLIC_FIREBASE_VAPID_KEY=your-vapid-key

# Backend API
EXPO_PUBLIC_API_URL=http://localhost:4000/api
```

## 4. Firebase Services Configuration

### Authentication
1. Go to Firebase Console > Authentication > Sign-in method
2. Enable "Email/Password"
3. Enable "Anonymous" (for testing)

### Firestore Database
1. Go to Firebase Console > Firestore Database
2. Create database in production mode
3. Set up security rules (see below)

### Cloud Messaging (FCM)
1. Go to Firebase Console > Project Settings > Cloud Messaging
2. Copy your Server Key (for backend)
3. Copy your Sender ID (for mobile app)

## 5. Security Rules

Add these Firestore security rules:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users can only access their own data
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    
    // Devices can only be accessed by their owners
    match /devices/{deviceId} {
      allow read, write: if request.auth != null && 
        request.auth.uid == resource.data.userId;
    }
    
    // Auth challenges
    match /authChallenges/{challengeId} {
      allow read: if request.auth != null && 
        (request.auth.uid == resource.data.userId || 
         request.auth.uid == resource.data.deviceId);
      allow create: if request.auth != null;
      allow update: if request.auth != null && 
        request.auth.uid == resource.data.deviceId;
    }
  }
}
```

## 6. Mobile App Integration

### Push Notifications Setup
1. Install required packages:
```bash
expo install expo-notifications
expo install expo-device
```

2. Configure notification handler in your app
3. Request notification permissions
4. Get and store FCM token

### Secure Storage
Use `expo-secure-store` for storing:
- User credentials
- TOTP secrets
- PQC private keys
- JWT tokens

## 7. Testing

### Backend Testing
```bash
cd quantum-auth-backend
npm install
npm run dev
```

### Mobile Testing
```bash
cd quantum-auth-mobile
npm install
expo start
```

## 8. Production Deployment

### Backend
1. Set up production environment variables
2. Deploy to your preferred cloud platform
3. Update mobile app API URLs

### Mobile App
1. Build production APK/IPA
2. Test with production backend
3. Submit to app stores