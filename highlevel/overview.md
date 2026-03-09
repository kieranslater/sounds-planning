#  Audio App – High Level Design Document

**Purpose:** Enable users to discover, play, and mix high-quality audio tracks for sleep, focus, and ambience. The system supports free and Pro tiers, offline caching, streaming via CDN, and analytics.

---

## 1. Goals / Objectives

- Provide **fast, swipeable access** to hundreds of audio tracks.  
- Support **Pro users** streaming high-resolution audio and **Free users** with embedded tracks.  
- Include **mixer view** to combine multiple tracks with individual volume control.  
- Provide **previews (first 20 seconds)** of tracks in list view, heavily cached on CDN.  
- Minimise device storage usage via selective offline downloads.  
- Track **user interactions and usage telemetry** without storing PII.  
- Scalable architecture with **low cost** using AWS and Cloudflare CDN.  

---

## 2. System Overview

| Layer                 | Components / Technologies | Responsibilities |
|-----------------------|--------------------------|-----------------|
| Client                | iOS/macOS App (SwiftUI)  | Swipe UI, Mixer, Previews, Offline downloads, Display storage usage, Favorites, Playback & seamless loops |
| CDN & Storage         | Cloudflare + S3          | Serve full tracks & previews, cache aggressively, signed URL validation |
| Backend API           | Go (Lambda / ECS)        | Catalog API, presigned URLs, subscription validation, telemetry ingestion, admin interface endpoints |
| Database              | DynamoDB                 | Tracks metadata, user UUIDs, favorites, downloaded tracks |
| Admin Interface       | Next.js / React          | Upload tracks, manage metadata, trigger previews, monitor storage & usage |
| Analytics / Telemetry | PostHog / Lambda / Kinesis | Collect swipes, track listens, downloads, mixer usage |

---

## 3. Client Features

### 3.1 Swipe / Track Catalogue

- Horizontal swipe through tracks.  
- Preview playback of first 20 seconds (cached on CDN).  
- Loading bar / loop indicator for previews and full tracks.  
- Tap for full track or to add to favorites.  

### 3.2 Mixer View

- Mix **up to 4 tracks** simultaneously.  
- Individual volume sliders, mute/unmute.  
- Use AVAudioEngine for low-latency mixing.  
- Can play downloaded tracks offline.  

### 3.3 Offline Management

- Select tracks to download for offline use.  
- Shows **storage usage per track** and total.  
- Option to delete tracks without uninstalling app.  

### 3.4 Free vs Pro

| Feature                   | Free                     | Pro                        |
|----------------------------|-------------------------|----------------------------|
| Track availability         | 5–6 built-in MP3 tracks | Full catalogue (256 kbps, 8 min loops) |
| Previews                   | Available               | Available                  |
| Streaming                  | Limited / local only    | Streaming via CDN          |
| Offline download           | Not available / minimal | Full download for offline  |
| Mixer                      | Limited or single track | Full 4-track mixer         |

---

## 4. Backend / API

### 4.1 Core Responsibilities

- **Catalog endpoints**: Provide track metadata, filter by tags, paging.  
- **Presigned URL generation**: Secure access for previews and full tracks.  
- **Subscription validation**: Apple IAP receipts mapped to user UUIDs.  
- **Telemetry ingestion**: Batched events for analytics.  
- **Admin endpoints**: Track upload, preview generation trigger, metadata update.

### 4.2 Security

- Private S3 buckets for audio.  
- Cloudflare signed URLs with TTL (e.g., 5–10 min).  
- Rate limiting to prevent scraping.  
- Obfuscated file names.  
- Optional Apple DeviceCheck / App Attest integration.  

---

## 5. Storage & Streaming

- **Full tracks:** 8-min loops, 256 kbps, stored in `full/` S3 folder.  
- **Previews:** 20s, lower bitrate, stored in `previews/` folder.  
- **Streaming:** Cloudflare CDN caches previews aggressively; full tracks streamed on-demand.  
- **Offline downloads:** Managed per user, tracked in DynamoDB.  
- **Optional HLS segmentation** for seamless loops and adaptive streaming.  

---

## 6. Preview Generation Lambda

- Triggered by S3 `PUT` of new full track.  
- Uses FFmpeg to create **first 20 seconds** preview.  
- Stores preview in `previews/` S3 folder.  
- Updates DynamoDB `Tracks` table with preview metadata.  
- Ensures CDN can pre-cache the preview for instant playback.  

---

## 7. Database Design (DynamoDB)

### Tracks Table

| Field         | Type           | Notes                  |
|---------------|----------------|-----------------------|
| track_id      | string (PK)    | UUID                  |
| title         | string         | Track name            |
| duration      | int            | Seconds               |
| file_key      | string         | S3 object key         |
| preview_key   | string         | S3 object key preview |
| pro_only      | bool           | True for Pro-only     |
| checksum      | string         | Optional integrity    |
| upload_ts     | timestamp      | Track versioning      |

### Users Table

| Field            | Type           | Notes                  |
|------------------|----------------|-----------------------|
| user_uuid        | string (PK)    | Device-generated UUID |
| subscription     | bool           | Pro status            |
| downloads[]      | list           | Offline tracks        |
| favourites[]     | list           | Liked tracks          |

### Telemetry / Event Table

- user_uuid, track_id, event_type, timestamp, duration, mixer usage  
- Aggregated nightly or batched to reduce cost.

---

## 8. Admin / Upload Flow

1. Admin uploads audio track.  
2. API returns presigned POST URL.  
3. Track uploaded to S3.  
4. Lambda generates 20s preview automatically.  
5. DynamoDB updated with metadata.  
6. CDN caches preview segment for fast app swipe view.  

---

## 9. Security / Access Control

- Private S3 bucket, Cloudflare in front.  
- Signed URLs for both preview & full tracks.  
- Rate limiting and token checks at CDN edge.  
- Obfuscate file structure (`UUID.m4a` instead of descriptive filenames).  
- Device UUID + optional Apple App Attest for anti-abuse.  

---

## 10. Telemetry & Analytics

- Track:
  - Number of tracks played
  - Swipes / track preview views
  - Mixer usage (volume levels, active tracks)
  - Downloads / offline usage
- Anonymised by UUID only (no personal data).  
- Analytics stored in PostHog, or Kinesis → S3 → Lambda for aggregation.  

---

## 11. Scaling / Cost Considerations

- Serverless Go API (Lambda) → auto-scales, low cost.  
- Cloudflare CDN caches previews and full tracks → minimal S3 egress.  
- DynamoDB minimal cost for metadata and small user data.  
- Pre-cache previews → reduces repeated S3 reads for high-traffic “list view”.  
- Optional HLS segmentation → lower startup time and smoother looping.  

---

## 12. Next Steps (Breakdown for deeper design)

1. **Frontend**
   - Swipe view & preview playback.
   - Mixer design with AVAudioEngine.
   - Offline downloads & storage UI.
   - Favorites & subscription handling.

2. **Backend / API**
   - Catalog endpoint & pagination.
   - Presigned URL generation & access control.
   - Telemetry ingestion.
   - Subscription receipt validation.
   - Admin endpoints for uploads & metadata.

3. **S3 + CDN**
   - Bucket structure & permissions.
   - Cloudflare caching rules & token auth.
   - Pre-cache preview segments.

4. **Lambda Functions**
   - Preview generation from full tracks.
   - Optional normalization / transcoding.

5. **Database**
   - DynamoDB table design & access patterns.
   - User UUID management & offline tracking.

6. **Analytics**
   - Event schema & aggregation pipeline.
   - Dashboard / reporting options.

7. **Security**
   - Signed URLs, obfuscated filenames.
   - Device attestation / API auth.
   - Rate limiting & abuse mitigation.
