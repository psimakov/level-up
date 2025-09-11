<cheat-sheet>

# Cheat Sheet: Design YouTube

## 1. Problem Statement & Scope

Design a video-sharing platform allowing users to upload and stream video content. The focus is on the core mechanics of uploading and watching videos at a massive scale.

### Functional Requirements

*   **Core:**
    *   Users can upload videos.
    *   Users can watch (stream) videos.
*   **Out of Scope:**
    *   Video metadata (view counts, comments, recommendations).
    *   Search, channels, subscriptions.

### Non-Functional Requirements

*   **High Availability:** Prioritize availability over strong consistency (eventual consistency is acceptable for uploads).
*   **Large File Support:** Handle uploads and streaming for very large videos (e.g., up to 256 GB).
*   **Low Latency Streaming:** Ensure fast video start-up (< 500ms) even in low-bandwidth environments.
*   **Scalability:** Support ~1 million uploads and ~100 million video views per day.
*   **Resumable Uploads:** Uploads should be able to resume after an interruption.

## 2. Core Concepts: Video Streaming 101

*   **Codec (Encoder/Decoder):** An algorithm (e.g., H.264, AV1) that compresses video data to reduce file size while preserving quality.
*   **Container:** A file format (e.g., MP4, MOV) that holds the compressed video, audio, and metadata.
*   **Bitrate:** The amount of data used to represent one second of video (in kbps or mbps). Higher quality video has a higher bitrate.
*   **Manifest File:** A text file (like an index or playlist) that lists all available versions (different bitrates/resolutions) of a video and provides URLs to the small video segments for each version.

## 3. High-Level Design & Solution Walkthrough

The system is split into two main asynchronous pipelines: **Upload** and **Streaming**.

### A. Video Upload Pipeline

This pipeline handles getting a large video file from a user into the system and preparing it for streaming. Direct uploads to the application server are not feasible due to file size limits.

1.  **Initiate Upload:**
    *   The client sends a request to the **Video Service** with video metadata (title, description, file size).
    *   The Video Service creates a metadata record in a database (e.g., Cassandra, DynamoDB) with a status of `pending`.
2.  **Direct Upload to Blob Storage:**
    *   The Video Service requests pre-signed URLs from a blob store (like AWS S3) for a **multipart upload**.
    *   These URLs are returned to the client. The client bypasses the application server and uploads the video in large chunks (~5-10MB each) directly to S3. This is crucial for handling large files and enabling resumability.
3.  **Asynchronous Post-Processing (The DAG):**
    *   Once all chunks are uploaded, S3 stitches them into the original video file and triggers an **S3 Notification** (e.g., to an SQS queue).
    *   This notification kicks off a Directed Acyclic Graph (DAG) of processing jobs orchestrated by a system like Temporal or AWS Step Functions:
        *   **Validation:** Check the video format, scan for malware.
        *   **Segmentation (Chunking):** The original video is split into small, 2-10 second segments optimized for streaming.
        *   **Transcoding (Parallel):** Each segment is transcoded in parallel into multiple formats and resolutions (e.g., 1080p, 720p, 480p). This creates different bitrate versions to support adaptive streaming.
        *   **Manifest Generation:** A manifest file is created, containing URLs (pointing to S3/CDN) for all the transcoded segments.
        *   **Store Artifacts:** All segments and the manifest file are stored back in S3.
4.  **Finalize Upload:**
    *   The final step of the DAG updates the video metadata record in the database, changing the status to `uploaded` and adding the URL to the manifest file.

### B. Video Streaming (Playback) Pipeline

This pipeline handles delivering the processed video to viewers efficiently.

1.  **Client Request:** A user clicks play. The client requests the video metadata from the **Video Service**.
2.  **Fetch Manifest:** The metadata contains the URL for the **manifest file**. The client fetches this manifest file, preferably from a **Content Delivery Network (CDN)** to reduce latency. The CDN caches popular manifest files and video segments geographically close to users.
3.  **Adaptive Bitrate Streaming:**
    *   The client's video player inspects the manifest file to see all available quality levels.
    *   It detects the user's current network bandwidth.
    *   It starts by requesting the first few video segments for the appropriate quality level (e.g., 480p for a slow connection, 1080p for a fast one) from the CDN.
    *   The player continuously monitors network conditions and can seamlessly switch to requesting higher or lower quality segments to prevent buffering, providing an adaptive streaming experience.

## 4. Deep Dive: Common Problems & Pitfalls

### Resumable Uploads

*   **Problem:** A large video upload fails midway due to a network issue.
*   **Solution:** The multipart upload process inherently supports this. The client can query the Video Service to see which chunks have already been successfully uploaded (tracked via S3 event notifications updating the video metadata) and resume uploading only the missing ones.

### Scaling Reads & Hotspots

*   **Problem:** A viral video gets millions of views, creating a massive load on the backend.
*   **Solution:**
    *   **CDN:** The CDN is the first and most important line of defense. It absorbs the vast majority of requests for popular video segments and manifest files, shielding S3 and the backend services.
    *   **Metadata Caching:** The Video Metadata DB can still become a bottleneck for popular videos. A distributed cache (like Redis or Memcached) should be used to store metadata for frequently accessed videos, reducing database load. The cache would be partitioned by `videoId`.

### Common Candidate Mistakes

*   **Ignoring Large Files:** Proposing a design where the video is uploaded through the application server in a single POST request. This is a major red flag.
*   **Not Understanding Asynchronicity:** Failing to recognize that video processing (transcoding, etc.) must be done asynchronously in the background after the initial upload is complete.
*   **Lack of Intuition on Streaming:** Not realizing that videos must be served in small chunks (segments) to enable quick start times and adaptive streaming. Simply providing a download link to the full S3 file is an incomplete solution.
*   **Underestimating the Role of a CDN:** Designing the system without a CDN, which is essential for scaling a global video delivery platform.

## 5. Interviewer's Guide: Leveling Expectations

*   **Mid-Level (E4):**
    *   Should understand the need for blob storage (S3) and asynchronous processing.
    *   May need hints to arrive at multipart uploads and segment-based streaming. The core idea of "chunking" on both ends is key.
*   **Senior (E5):**
    *   Should proactively design for large files using multipart uploads.
    *   Must understand and explain the need for transcoding and adaptive bitrate streaming.
    *   Should proactively introduce a CDN and caching layers to address scaling.
*   **Staff+ (E6+):**
    *   Expected to have a deep, intuitive understanding of the entire end-to-end flow.
    *   Should drive the conversation, discussing trade-offs, specific technologies (e.g., different codecs, orchestration tools like Temporal), and failure modes (e.g., what if a transcoding job fails?).
    *   Treats the interviewer as a peer in a design discussion.

</cheat-sheet>