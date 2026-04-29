https://yaswanthreddy9.github.io/driver/





# The Capacitor & Firebase Push Notification Survival Guide
**Stack:** Capacitor (Web-wrapper), Android, Firebase Cloud Functions (Node 20), Firestore.
**Goal:** Reliable, loud, lock-screen-bypassing push notifications for a Delivery Driver App.

## Overview
Building a driver application using a web-to-native framework (like Capacitor) introduces a massive hurdle: **The Javascript Coma**. When an Android phone locks, it freezes the app's Javascript engine to save battery. Standard push notifications cannot wake the Javascript engine up, resulting in silent, missed orders.

Furthermore, deploying Firebase Cloud Functions to trigger these notifications requires navigating a strict Google Cloud IAM permissions gauntlet.

This document covers the complete pipeline to solve both issues.

---

## Phase 1: The Firebase Deployment Gauntlet
When deploying Firebase Cloud Functions (`firebase deploy --only functions`) on a modern Blaze plan (Node 20), Google utilizes a background "Builder Robot" to compile your code. By default, this robot often lacks the necessary security clearance, resulting in cryptic build failures.

### The Fix: Granting IAM Permissions
You must locate the Compute Engine default service account in your Google Cloud IAM settings. It usually looks like this:
`[PROJECT_NUMBER]-compute@developer.gserviceaccount.com`

*Note: You may need to check "Include Google-provided role grants" in the IAM console to see it, or add it manually via the "Grant Access" button.*

You must assign this specific account **three critical roles**:
1. **Storage Object Viewer:** Allows the robot to read the source code you just uploaded.
2. **Logs Writer:** Allows the robot to write its build-status report. Without this, the build panics and aborts.
3. **Artifact Registry Administrator:** Allows the robot to save the final compiled "image cache" to the cloud.

### The Fix: Node Version Conflicts
If your local machine runs a newer version of Node (e.g., v24) than the Google Cloud environment (v20), the `package-lock.json` file will contain a futuristic blueprint that crashes the Google server.
**The Solution:**
1. Delete the local `package-lock.json` file.
2. Run `npm install --save firebase-functions@latest firebase-admin@latest` to rebuild a clean blueprint.
3. If a previous deployment jammed, wipe it via CLI: `firebase functions:delete [functionName]`.

---

## Phase 2: Bypassing the "Javascript Coma"
Because Android freezes Capacitor's web engine when the screen is off, you cannot rely on Javascript (`audio.play()`) to ring the delivery alarm. You must delegate the alarm directly to the native Android Operating System using the **"Native Siren" Method**.

### Step 1: Embed the Native Audio File
You cannot load the MP3 from your web assets. It must exist inside the native Android system.
1. Download your alarm file (e.g., `siren.mp3`).
2. Open Android Studio.
3. Navigate to `app/src/main/res`.
4. Create a new directory named `raw`.
5. Place `siren.mp3` inside the `raw` folder.

### Step 2: Create a High-Importance Notification Channel (Frontend)
On Android 8.0+, high-priority alerts require a pre-registered "Notification Channel." The app must explicitly tell Android to attach the native siren to this specific channel.

*Add this to your Capacitor app's initialization logic:*

```javascript
async function setupPushNotifications() {
    if (window.Capacitor && window.Capacitor.Plugins.PushNotifications) {
        const PushNotifications = window.Capacitor.Plugins.PushNotifications;

        // 1. Create the Critical Channel
        await PushNotifications.createChannel({
            id: 'order_alerts',         // Must match the backend payload exactly
            name: 'New Order Alerts',
            description: 'Critical alerts for new delivery assignments',
            importance: 5,              // 5 = Highest (Forces heads-up display and audio)
            visibility: 1,              // 1 = Public (Shows on lock screen)
            vibration: true,
            sound: 'siren'              // Matches the MP3 in the 'raw' folder (no extension)
        });

        // 2. Request Permissions & Register Token
        let permStatus = await PushNotifications.requestPermissions();
        if (permStatus.receive === 'granted') {
            await PushNotifications.register();
        }

        // 3. Save FCM Token to Firestore
        PushNotifications.addListener('registration', (token) => {
            db.collection("drivers").doc(driverId).update({ fcmToken: token.value });
        });
    }
}
```
*⚠️ **Crucial Rule:** Notification channels are permanently saved to the Android OS the first time they are created. If you are modifying an existing app, you **must completely uninstall the app from the phone** and reinstall the new APK to register the new channel settings.*

### Step 3: Trigger the Native Siren (Backend)
The Firebase Cloud Function must format the payload specifically for Android to bypass "Doze Mode." It must include a high priority flag, a Time-to-Live (TTL), and reference the exact Channel ID.

*Deploy this in `functions/index.js`:*

```javascript
const functions = require("firebase-functions/v1");
const admin = require("firebase-admin");
admin.initializeApp();

exports.sendOrderNotification = functions.firestore
    .document("orders/{orderId}")
    .onUpdate(async (change, context) => {
        const afterData = change.after.data();
        const beforeData = change.before.data();

        // Trigger only when a driver is newly assigned
        if (!beforeData.driverId && afterData.driverId) {
            const driverDoc = await admin.firestore().collection("drivers").doc(afterData.driverId).get();

            if (driverDoc.exists && driverDoc.data().fcmToken) {
                const message = {
                    token: driverDoc.data().fcmToken,
                    notification: {
                        title: "New Delivery Assigned! 🛵",
                        body: `Order #${afterData.orderNumber || "New"} is ready.`
                    },
                    // The Native Android Wake-Up Engine
                    android: {
                        priority: "high", // Bypasses Doze Mode
                        ttl: 3600000,     // Network retry duration (1 hour)
                        notification: {
                            channelId: "order_alerts", // Matches Capacitor Frontend
                            sound: "siren",            // Triggers the native MP3
                            clickAction: "FCM_PLUGIN_ACTIVITY" // Forces app open on tap
                        }
                    },
                    data: {
                        orderId: context.params.orderId,
                        type: "NEW_ORDER"
                    }
                };

                try {
                    await admin.messaging().send(message);
                    console.log("High-priority alert sent.");
                } catch (error) {
                    console.error("FCM Error:", error);
                }
            }
        }
        return null;
    });
```

## Summary of Execution
By combining Google Cloud IAM fixes, High-Priority FCM payloads, and Native Android `res/raw` audio mapping, a web-based Capacitor application can successfully mimic the lock-screen bypassing behavior of a purely native Java/Kotlin delivery application.
