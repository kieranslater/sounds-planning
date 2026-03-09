# Presigned URL Flow – Low-Level Design

## 1. Goals

* Only valid users (UUID + subscription) can access audio.
* Prevent scraping or unauthorized downloading.
* Enable **short-lived access** that cannot be reused indefinitely.
* Support both **previews** and **full tracks** seamlessly.
* Work well with **offline downloads** without exposing credentials.

---

## 2. AWS Setup

1. **S3 Buckets**

   * `audio-bucket/full/` → full tracks (Pro only)
   * `audio-bucket/previews/` → 20s preview tracks (Free + Pro)
   * Buckets are **private**; no public reads.

2. **IAM Role / Policy**

   * Lambda / Go API has a role allowing:

   ```json
   {
     "Effect": "Allow",
     "Action": ["s3:GetObject"],
     "Resource": ["arn:aws:s3:::audio-bucket/*"]
   }
   ```

3. **Cloudflare CDN**

   * Points to S3 bucket as origin.
   * Optional token-based or header-based enforcement for extra security.

---

## 3. Backend (Go API) Logic

### 3.1 User Request

```
GET /track/{track_id}/presign?type=preview
Authorization: Bearer <app_token>
X-User-ID: <uuid>
```

* Validate `user_uuid` exists.
* Check **subscription status** if requesting full track.
* Rate limiting / abuse prevention.
* Fetch **S3 object key** from DynamoDB:

  * Preview → `previews/<track_id>.m4a`
  * Full → `full/<track_id>.m4a`

---

### 3.2 Generate Presigned URL

**Go example using AWS SDK v2:**

```
import (
    "time"
    "context"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

func GeneratePresignedURL(bucket, key string, duration time.Duration) (string, error) {
    presignClient := s3.NewPresignClient(s3Client)

    presignedReq, err := presignClient.PresignGetObject(context.TODO(), &s3.GetObjectInput{
        Bucket: &bucket,
        Key:    &key,
    }, s3.WithPresignExpires(duration))
    
    if err != nil {
        return "", err
    }
    return presignedReq.URL, nil
}
```

* `duration` could be **5–10 min** for full tracks, longer for cached previews.
* Example presigned URL returned:

```
https://cdn.yourapp.com/full/abc123.m4a?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...
```

---

### 3.3 Return to Client

```
{
  "track_id": "abc123",
  "presigned_url": "https://cdn.yourapp.com/full/abc123.m4a?...","
  "expires_in": 300
}
```

* App starts playback immediately.
* If URL expires, app requests a new presigned URL.

---

## 4. Client Usage

* Request presigned URL when starting a preview or full track.
* Playback uses **AVPlayer / AVAudioEngine** directly.
* Do not store URL beyond TTL.
* Offline download: fetch while URL is valid, store locally.

---

## 5. Security Enhancements

* Short TTL reduces risk if URL leaks.
* Optional **CDN token validation** or signed cookies.
* Device UUID verification ensures only registered devices can request URLs.
* Optional **Apple App Attest / DeviceCheck** for app-origin validation.
* Previews may have longer TTL since they are cached; full tracks have short TTL.

---

## 6. Preview vs Full Track Handling

| Action                   | Track Type | TTL       | Notes                      |
| ------------------------ | ---------- | --------- | -------------------------- |
| Swipe in catalogue       | Preview    | 10–15 min | Aggressively cached in CDN |
| Tap for full playback    | Full       | 5 min     | Subscription checked       |
| Offline download request | Full       | 5–10 min  | Download stored locally    |

---

## 7. Advantages

* No S3 credentials exposed to client.
* CDN caching reduces S3 egress and cost.
* Can revoke access by changing object key or TTL.
* Works seamlessly with free/pro tiers and offline downloads.

---

## 8. Optional Enhancements

* Rotate object keys to revoke access if needed.
* Log presigned URL requests in DynamoDB for audit/abuse detection.
* Pre-generate presigned URLs for popular previews to reduce API load.
* Integrate with rate-limiting API Gateway for additional security.
