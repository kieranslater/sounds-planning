# Audio App – Design Notes

## Overview

This document outlines ideas and high-level design for an iOS/macOS audio app supporting Free and Pro tiers, streaming, offline downloads, and a mixer feature.

---

## Client Features

### 1. UI / UX

* Swipe to view different tracks.
* Different backlight / theme options.
* Loading bar and clear loop indication.
* Mixer view: mix up to 4 tracks simultaneously with individual volume control.
* Favorites and offline download management.
* Show storage usage and allow easy deletion of tracks.

### 2. Free vs Pro Users

| Feature          | Free                            | Pro                                     |
| ---------------- | ------------------------------- | --------------------------------------- |
| Tracks           | 5–6 MP3 quality tracks embedded | Full catalogue, 256kbps, 8-minute loops |
| Playback         | Local / limited                 | Streamed via CDN                        |
| Mixer            | Limited                         | Full 4-track mixer                      |
| Offline download | Limited / none                  | Full offline download                   |

### 3. Platforms

* iOS
* macOS
* Optional: tvOS

---

## Backend / API

### 1. Golang API

* Provides list of tracks from DynamoDB.
* Manages user access via UUID (no PII).
* Generates presigned URLs for secure streaming/downloads.
* Handles telemetry ingestion (swipes, listens, downloads, mixer usage).

### 2. Database

* DynamoDB tables for tracks metadata, user UUIDs, favorites, downloads.
* Track metadata includes track ID, file key, preview key, Pro/free flag.

### 3. Admin Website

* Allows uploads to S3 bucket.
* Automatically generates 20-second previews on upload (via Lambda).
* Updates DynamoDB with track metadata.
* Releases tracks programmatically to app.

---

## Storage & Streaming

* S3 buckets for full tracks and previews.
* Cloudflare CDN for streaming and caching.
* Pre-cache first 20 seconds of each track for previews.
* Downloads managed selectively: only tracks user taps on.

---

## Security

* Signed URLs for downloads/streams.
* App Attest / DeviceCheck from Apple to prevent abuse.
* Optional rate limiting and device UUID checks.
* CDN-level token/header validation for added security.

---

## Telemetry & Analytics

* Track how many tracks are downloaded.
* Track swipes and how long users listen to each track.
* Mixer usage analytics.
* Anonymous, keyed by UUID only.

---

## Notes / Considerations

* Avoid downloading all 300+ tracks for Pro users: smart selective download.
* Ensure offline downloads are secure.
* Plan for scaling: AWS, Cloudflare, DynamoDB costs and limits.
* Make the app simple and user-friendly.
* Optional: support macOS/tvOS platforms later.
* Keep the backend modular for easy expansion (new features, track types, etc.).
