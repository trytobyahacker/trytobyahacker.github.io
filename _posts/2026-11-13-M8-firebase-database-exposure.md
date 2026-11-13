---
title: "M8: Security Misconfiguration - Firebase Database - Insecure Rules Leading to Data Exposure"
date: 2026-11-13 01:00:00 +0000
categories: [Mobile, OWASP-TOP-10]
tags: [allsafe, android, firebase, owasp, m8, misconfiguration]
description: Exploiting Allsafe's Firebase Database challenge by querying the Realtime Database REST API directly, bypassing the app due to missing security rules.
---

## Objective

Challenge 05 of **Allsafe** focuses on a Firebase Realtime Database with insecure rules.

> **Mission briefing:** In this task, the application is getting data from a realtime database. It's all nice and good, but I have a feeling the developers didn't set the correct rules for production.

## Environment

- Genymotion Phone (570x1230, 220dpi)
- Allsafe installed
- jadx-gui for static analysis

## Step 1: Interact with the app

The "Firebase Data Exposure" challenge shows a **[QUERY DATABASE]** button:

![Firebase Database challenge screen](/assets/img/posts/Screenshot%202026-11-13%20022502.png)

## Step 2: Decompile with jadx

Opening `allsafe.apk` in jadx-gui, I find the `FirebaseDatabase` class, which queries a specific node from the Realtime Database:

```java
com.google.firebase.database.FirebaseDatabase database = DatabaseKt.getDatabase(Firebase.INSTANCE);
DatabaseReference reference = database.getReference().child("secret");
reference.addValueEventListener(new ValueEventListener() {
    @Override
    public void onDataChange(DataSnapshot dataSnapshot) {
        ...
    }
});
```

The app only reads and displays the `secret` node, by design.

![jadx showing the FirebaseDatabase class querying the "secret" node](/assets/img/posts/Screenshot%202026-11-13%20022520.png)

## Step 3: Find the database URL

Checking `strings.xml`, the project's Firebase configuration is listed in plaintext, as it normally is for any Firebase app:

```xml
<string name="firebase_database_url">https://allsafe-8cef0.firebaseio.com</string>
<string name="google_api_key">AIzaSyDjteCQ0-ElkfBxVZIZmBfCSPNEYUYcK1g</string>
<string name="google_app_id">1:983632160629:android:d1d9132ddd988e7127553c</string>
```

![strings.xml showing the Firebase database URL and API key](/assets/img/posts/Screenshot%202026-11-13%20022643.png)

## Step 4: Query the entire database directly, bypassing the app

Firebase Realtime Database exposes a REST API for every node: appending `.json` to any path returns its content. Instead of only requesting the `secret` node the app uses, I query the **database root**:

https://allsafe-8cef0.firebaseio.com/.json


The response returns the full database content, unauthenticated:

```json
{
  "flag": "5077e90341de49d0ed79b8ee53572dab",
  "secret": "A bug is never just a mistake. It represents something bigger. An error of thinking. That makes you who you are."
}
```

![Browser showing the raw JSON response with both the flag and secret nodes](/assets/img/posts/Screenshot%202026-11-13%20023036.png)

This confirms the finding goes beyond the single `secret` node the app was designed to show: **the entire database has public read access**, including a `flag` node that isn't referenced anywhere in the app's code at all. Any node added to this database in the future would be exposed the same way, by default, with zero changes needed on the attacker's side.

> **Note:** Trying to log into `console.firebase.google.com` for this project correctly fails with a permissions error, since that requires being an owner/collaborator on the Firebase project. That's unrelated to this vulnerability: the issue isn't the console, it's the public REST endpoint having no read restrictions at the database root.

## Why this is a problem

- Firebase Realtime Database rules default to **requiring authentication**, but it's common for developers to set them to `".read": true, ".write": true` during development and forget to lock them down before shipping.
- Anyone who extracts the `firebase_database_url` from the APK (a trivial static analysis step) can read, and potentially write, **every node in the database**, not just the ones the app's own code happens to query.
- Since the rule applies at the root, new nodes added later inherit the same exposure automatically, with no extra effort from an attacker.

## OWASP classification

This maps to **M8: Security Misconfiguration** in the OWASP Mobile Top 10 (2024), since the root cause is a misconfigured backend service (Firebase security rules) reachable because the client exposes enough information to locate and query it directly. It also relates to **CWE-284: Improper Access Control**.

## Mitigation

- Set Firebase Realtime Database rules to require authentication by default, and scope read/write access per user (e.g. `auth.uid == $uid`), never leave `".read": true, ".write": true` at the root in production.
- Use Firebase App Check to ensure only legitimate instances of your app can query the database.
- Treat the database URL as something that will inevitably be extracted from the client, security must come from the server-side rules, not from obscurity.
- Regularly audit Firebase rules with the [Firebase Rules Playground](https://firebase.google.com/docs/rules/simulator) before each release.

## References

- OWASP Mobile Top 10 2024 - M8: Security Misconfiguration
- [Firebase Realtime Database Security Rules](https://firebase.google.com/docs/database/security)
- [Allsafe - Intentionally Vulnerable Android Application](https://github.com/t0thkr1s/allsafe)
