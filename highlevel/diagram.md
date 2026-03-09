flowchart TD
    %% Client layer
    A[iOS / macOS App] -->|Request catalogue & previews| B[Go API]
    A -->|Stream preview| C[Cloudflare CDN]
    A -->|Stream full track| C

    %% Backend API
    B --> D[DynamoDB: Tracks Table]
    B --> E[DynamoDB: Users Table]
    B --> F[DynamoDB / Telemetry Table]
    B -->|Generate signed URL| C

    %% Storage layer
    C -->|Cache & serve| G[S3 Bucket: Previews / Full Tracks]

    %% Lambda preview generation
    G -->|S3 PUT triggers| H[Lambda: Generate 20s Preview]
    H -->|Store preview| G
    H -->|Update metadata| D

    %% Admin upload flow
    I[Admin Web UI] -->|Request presigned URL| B
    I -->|Upload full track| G

    %% Analytics
    F --> J[Analytics / Dashboard]
    
    %% Notes
    classDef storage fill:#f9f,stroke:#333,stroke-width:1px;
    classDef api fill:#bbf,stroke:#333,stroke-width:1px;
    classDef client fill:#bfb,stroke:#333,stroke-width:1px;
    classDef lambda fill:#ffb,stroke:#333,stroke-width:1px;
    
    class A client;
    class B api;
    class C storage;
    class G storage;
    class H lambda;