```mermaid

flowchart TD
    %% Client layer
    App[iOS / macOS App] -->|Request catalogue & previews| API[Go API]
    App -->|Stream preview| CDN[Cloudflare CDN]
    App -->|Stream full track| CDN

    %% Backend API
    API --> Tracks[DynamoDB: Tracks Table]
    API --> Users[DynamoDB: Users Table]
    API --> Telemetry[DynamoDB: Telemetry Table]
    API -->|Generate signed URL| CDN

    %% Storage layer
    CDN -->|Cache & serve| S3[S3 Bucket: Previews / Full Tracks]

    %% Lambda preview generation
    S3 -->|S3 PUT triggers| Lambda[Lambda: Generate 20s Preview]
    Lambda -->|Store preview| S3
    Lambda -->|Update metadata| Tracks

    %% Admin upload flow
    Admin[Admin Web UI] -->|Request presigned URL| API
    Admin -->|Upload full track| S3

    %% Analytics
    Telemetry --> Dashboard[Analytics / Dashboard]
    ```