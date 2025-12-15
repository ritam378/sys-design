# Design a Video Streaming Platform

## Table of Contents
- [Problem Statement](#problem-statement)
- [Requirements](#requirements)
- [Capacity Estimation](#capacity-estimation)
- [High-Level Design](#high-level-design)
- [Video Upload and Processing](#video-upload-and-processing)
- [Video Transcoding](#video-transcoding)
- [Adaptive Bitrate Streaming](#adaptive-bitrate-streaming)
- [CDN Distribution](#cdn-distribution)
- [Storage Architecture](#storage-architecture)
- [Video Playback](#video-playback)
- [Analytics and Monitoring](#analytics-and-monitoring)
- [Optimizations](#optimizations)
- [Real-World Examples](#real-world-examples)
- [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **video streaming platform** like YouTube or Netflix that can:
- Upload and store videos
- Transcode videos to multiple formats/resolutions
- Stream videos with adaptive bitrate
- Handle millions of concurrent viewers
- Provide low-latency playback worldwide

**Similar to**: YouTube, Netflix, Twitch, TikTok

---

## Requirements

### Functional Requirements

1. **Upload videos**: Users can upload videos (max 10GB)
2. **Watch videos**: Stream videos with adaptive quality
3. **Search videos**: Search by title, description, tags
4. **Recommendations**: Suggest related videos
5. **Comments and likes**: Social features
6. **Analytics**: View counts, watch time, engagement

### Non-Functional Requirements

1. **Scalability**: 1 billion users, 500M daily active
2. **Availability**: 99.9% uptime
3. **Low Latency**: <2s to start playback
4. **Global Distribution**: CDN coverage
5. **Storage**: Petabytes of video content
6. **Bandwidth**: Handle millions of concurrent streams
7. **Cost Efficiency**: Optimize storage and CDN costs

---

## Capacity Estimation

### Storage

```
Assumptions:
- 500 million videos in library
- Average video: 300MB (original)
- After transcoding: 5 versions × 100MB = 500MB per video
- Total storage: 500M × 500MB = 250 PB

With replication (3x): 750 PB
With metadata/thumbnails: 800 PB
```

### Bandwidth

```
Daily active users: 500M
Average watch time: 30 minutes/day
Average bitrate: 2 Mbps

Daily bandwidth:
500M users × 30 min × 2 Mbps = 500M × 0.5 hours × 2 Mbps
= 500M × 3,600 MB = 1,800,000 TB/day = 1.8 EB/day

Peak concurrent users: 50M
Peak bandwidth: 50M × 2 Mbps = 100 Tbps
```

### Uploads

```
Daily uploads: 500,000 videos
Upload size average: 1GB
Daily upload data: 500 TB
Processing time: 10-30 minutes per video
```

---

## High-Level Design

```
┌──────────────────────────────────────────────────────────────┐
│                         Users                                 │
│                    (Billions worldwide)                       │
└────────────┬─────────────────────────────────────────────────┘
             │
             ↓
┌──────────────────────────────────────────────────────────────┐
│                      CDN (CloudFront/Akamai)                  │
│                  Cache video segments globally                │
└────────────┬─────────────────────────────────────────────────┘
             │
             ↓
┌──────────────────────────────────────────────────────────────┐
│                    Load Balancer                              │
└────────────┬─────────────────────────────────────────────────┘
             │
    ┌────────┴────────┐
    ↓                 ↓
┌─────────┐     ┌──────────────┐
│  API    │     │   Streaming  │
│ Gateway │     │   Service    │
└────┬────┘     └──────┬───────┘
     │                 │
     ↓                 ↓
┌─────────────────────────────────────────────────────────────┐
│                    Microservices                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Upload  │  │  Search  │  │Analytics │  │   Rec.   │   │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        ↓             ↓             ↓             ↓
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   S3     │  │PostgreSQL│  │  Redis   │  │Elastic-  │   │
│  │ (Videos) │  │(Metadata)│  │ (Cache)  │  │ search   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
        │
        ↓
┌─────────────────────────────────────────────────────────────┐
│              Video Processing Pipeline                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Upload  │→ │Transcode │→ │Thumbnail │→ │   CDN    │   │
│  │  Queue   │  │  (FFmpeg)│  │Generation│  │  Upload  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Video Upload and Processing

### Upload Flow

```python
from flask import Flask, request, jsonify
import boto3
import uuid
from datetime import datetime

app = Flask(__name__)
s3_client = boto3.client('s3')
sqs_client = boto3.client('sqs')

class VideoUploadService:
    def __init__(self):
        self.s3_bucket = 'raw-videos-bucket'
        self.processing_queue = 'video-processing-queue'

    def initiate_upload(self, user_id, filename, file_size):
        """
        Step 1: Generate presigned URL for direct S3 upload
        Avoids uploading through application servers
        """
        video_id = str(uuid.uuid4())
        s3_key = f"uploads/{user_id}/{video_id}/{filename}"

        # Generate presigned URL (valid for 1 hour)
        presigned_url = s3_client.generate_presigned_url(
            'put_object',
            Params={
                'Bucket': self.s3_bucket,
                'Key': s3_key,
                'ContentType': 'video/mp4'
            },
            ExpiresIn=3600
        )

        # Create video metadata
        metadata = {
            'video_id': video_id,
            'user_id': user_id,
            'filename': filename,
            'file_size': file_size,
            'status': 'uploading',
            's3_key': s3_key,
            'created_at': datetime.utcnow().isoformat()
        }

        # Save to database
        self.save_metadata(metadata)

        return {
            'video_id': video_id,
            'upload_url': presigned_url,
            's3_key': s3_key
        }

    def confirm_upload(self, video_id):
        """
        Step 2: Client confirms upload completion
        Trigger processing pipeline
        """
        # Update status
        self.update_status(video_id, 'uploaded')

        # Enqueue for processing
        message = {
            'video_id': video_id,
            'action': 'transcode',
            'timestamp': datetime.utcnow().isoformat()
        }

        sqs_client.send_message(
            QueueUrl=self.processing_queue,
            MessageBody=json.dumps(message)
        )

        return {'status': 'processing_queued'}

@app.route('/api/videos/upload/init', methods=['POST'])
def init_upload():
    """
    POST /api/videos/upload/init
    {
      "filename": "my_video.mp4",
      "file_size": 1073741824
    }
    """
    data = request.json
    service = VideoUploadService()
    result = service.initiate_upload(
        user_id=request.user_id,
        filename=data['filename'],
        file_size=data['file_size']
    )
    return jsonify(result), 200

@app.route('/api/videos/<video_id>/upload/complete', methods=['POST'])
def complete_upload(video_id):
    """
    POST /api/videos/{video_id}/upload/complete
    """
    service = VideoUploadService()
    result = service.confirm_upload(video_id)
    return jsonify(result), 200
```

### Multipart Upload (Large Files)

```python
class MultipartUploadService:
    def __init__(self):
        self.chunk_size = 10 * 1024 * 1024  # 10MB chunks

    def initiate_multipart_upload(self, video_id, filename):
        """Start multipart upload for large files"""
        response = s3_client.create_multipart_upload(
            Bucket='raw-videos-bucket',
            Key=f'uploads/{video_id}/{filename}',
            ContentType='video/mp4'
        )

        return {
            'upload_id': response['UploadId'],
            'chunk_size': self.chunk_size
        }

    def get_upload_url_for_part(self, video_id, upload_id, part_number):
        """Generate presigned URL for each chunk"""
        url = s3_client.generate_presigned_url(
            'upload_part',
            Params={
                'Bucket': 'raw-videos-bucket',
                'Key': f'uploads/{video_id}/video.mp4',
                'UploadId': upload_id,
                'PartNumber': part_number
            },
            ExpiresIn=3600
        )
        return url

    def complete_multipart_upload(self, video_id, upload_id, parts):
        """Complete multipart upload"""
        s3_client.complete_multipart_upload(
            Bucket='raw-videos-bucket',
            Key=f'uploads/{video_id}/video.mp4',
            UploadId=upload_id,
            MultipartUpload={'Parts': parts}
        )
```

---

## Video Transcoding

### Transcoding Pipeline

```python
import subprocess
import json

class VideoTranscoder:
    def __init__(self):
        self.output_formats = [
            {'name': '360p', 'width': 640, 'height': 360, 'bitrate': '800k'},
            {'name': '480p', 'width': 854, 'height': 480, 'bitrate': '1400k'},
            {'name': '720p', 'width': 1280, 'height': 720, 'bitrate': '2800k'},
            {'name': '1080p', 'width': 1920, 'height': 1080, 'bitrate': '5000k'},
            {'name': '4K', 'width': 3840, 'height': 2160, 'bitrate': '20000k'}
        ]

    def transcode_video(self, input_path, video_id):
        """
        Transcode video to multiple resolutions
        Use FFmpeg for encoding
        """
        results = []

        for fmt in self.output_formats:
            output_path = f"/tmp/{video_id}_{fmt['name']}.mp4"

            # FFmpeg command
            command = [
                'ffmpeg',
                '-i', input_path,
                '-vf', f"scale={fmt['width']}:{fmt['height']}",
                '-c:v', 'libx264',  # H.264 codec
                '-b:v', fmt['bitrate'],
                '-c:a', 'aac',
                '-b:a', '128k',
                '-movflags', '+faststart',  # Enable streaming
                '-y',  # Overwrite output
                output_path
            ]

            # Execute FFmpeg
            subprocess.run(command, check=True)

            # Upload to S3
            s3_key = f"videos/{video_id}/{fmt['name']}/video.mp4"
            s3_client.upload_file(
                output_path,
                'processed-videos-bucket',
                s3_key
            )

            results.append({
                'resolution': fmt['name'],
                's3_key': s3_key,
                'bitrate': fmt['bitrate']
            })

        return results

    def create_hls_segments(self, input_path, video_id, resolution):
        """
        Create HLS segments for adaptive streaming
        Output: .m3u8 playlist + .ts segments
        """
        output_dir = f"/tmp/{video_id}_{resolution}"
        os.makedirs(output_dir, exist_ok=True)

        command = [
            'ffmpeg',
            '-i', input_path,
            '-codec:', 'copy',
            '-start_number', '0',
            '-hls_time', '10',  # 10-second segments
            '-hls_list_size', '0',
            '-f', 'hls',
            f'{output_dir}/playlist.m3u8'
        ]

        subprocess.run(command, check=True)

        # Upload segments to S3
        self.upload_hls_to_s3(output_dir, video_id, resolution)

    def extract_thumbnail(self, input_path, video_id):
        """Extract thumbnail from video"""
        output_path = f"/tmp/{video_id}_thumb.jpg"

        command = [
            'ffmpeg',
            '-i', input_path,
            '-ss', '00:00:05',  # 5 seconds in
            '-vframes', '1',
            '-vf', 'scale=1280:720',
            output_path
        ]

        subprocess.run(command, check=True)

        # Upload thumbnail
        s3_client.upload_file(
            output_path,
            'thumbnails-bucket',
            f'{video_id}/thumbnail.jpg'
        )

        return f'{video_id}/thumbnail.jpg'

# Worker to process transcoding queue
class TranscodingWorker:
    def __init__(self):
        self.transcoder = VideoTranscoder()

    def process_message(self, message):
        """Process video from queue"""
        video_id = message['video_id']

        # Download from S3
        input_path = self.download_from_s3(video_id)

        # Transcode to multiple resolutions
        transcoded_videos = self.transcoder.transcode_video(
            input_path,
            video_id
        )

        # Create HLS segments
        for video in transcoded_videos:
            self.transcoder.create_hls_segments(
                video['s3_key'],
                video_id,
                video['resolution']
            )

        # Extract thumbnail
        thumbnail = self.transcoder.extract_thumbnail(input_path, video_id)

        # Update database
        self.update_video_metadata(video_id, {
            'status': 'ready',
            'formats': transcoded_videos,
            'thumbnail': thumbnail,
            'processed_at': datetime.utcnow().isoformat()
        })

        # Notify CDN to cache
        self.invalidate_cdn_cache(video_id)
```

---

## Adaptive Bitrate Streaming

### HLS (HTTP Live Streaming)

```
Master Playlist (master.m3u8):
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=854x480
480p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/playlist.m3u8

Resolution Playlist (720p/playlist.m3u8):
#EXTM3U
#EXT-X-TARGETDURATION:10
#EXTINF:10.0,
segment0.ts
#EXTINF:10.0,
segment1.ts
#EXTINF:10.0,
segment2.ts
...
```

### DASH (Dynamic Adaptive Streaming over HTTP)

```xml
<!-- manifest.mpd -->
<MPD xmlns="urn:mpeg:dash:schema:mpd:2011">
  <Period>
    <AdaptationSet mimeType="video/mp4">
      <Representation id="360p" bandwidth="800000" width="640" height="360">
        <BaseURL>360p/</BaseURL>
        <SegmentTemplate media="segment_$Number$.m4s"
                         initialization="init.mp4"
                         duration="10" />
      </Representation>
      <Representation id="720p" bandwidth="2800000" width="1280" height="720">
        <BaseURL>720p/</BaseURL>
        <SegmentTemplate media="segment_$Number$.m4s"
                         initialization="init.mp4"
                         duration="10" />
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

### Client-Side ABR Logic

```javascript
class AdaptiveBitratePlayer {
    constructor(videoElement) {
        this.video = videoElement;
        this.qualities = [];
        this.currentQuality = null;
        this.bandwidth = 0;
        this.bufferLength = 0;
    }

    selectQuality() {
        // Measure current bandwidth
        this.bandwidth = this.measureBandwidth();

        // Get buffer length
        this.bufferLength = this.video.buffered.length > 0
            ? this.video.buffered.end(0) - this.video.currentTime
            : 0;

        // Select appropriate quality
        let selectedQuality = this.qualities[0]; // Start with lowest

        for (let quality of this.qualities) {
            // Can we sustain this quality?
            if (this.bandwidth * 0.8 > quality.bitrate) {
                selectedQuality = quality;
            }
        }

        // If buffer is low, drop quality
        if (this.bufferLength < 5) {
            selectedQuality = this.getSafeQuality(selectedQuality);
        }

        // Switch if different from current
        if (selectedQuality !== this.currentQuality) {
            this.switchQuality(selectedQuality);
        }
    }

    measureBandwidth() {
        // Calculate from recent segment downloads
        const recentDownloads = this.getRecentDownloads();
        const totalBytes = recentDownloads.reduce((sum, d) => sum + d.bytes, 0);
        const totalTime = recentDownloads.reduce((sum, d) => sum + d.time, 0);

        return (totalBytes * 8) / totalTime; // bits per second
    }
}
```

---

## CDN Distribution

### Multi-Tier Caching

```
┌─────────────┐
│    Users    │
└──────┬──────┘
       │
       ↓
┌─────────────────┐
│   Edge CDN      │  ← Closest to users (Cloudflare, Akamai)
│ (Pop in city)   │     Cache: 1-24 hours
└──────┬──────────┘
       │ (cache miss)
       ↓
┌─────────────────┐
│  Regional CDN   │  ← Regional data centers
│                 │     Cache: 7-30 days
└──────┬──────────┘
       │ (cache miss)
       ↓
┌─────────────────┐
│  Origin (S3)    │  ← Source of truth
│                 │     Permanent storage
└─────────────────┘
```

### CDN Configuration

```python
import boto3

class CDNManager:
    def __init__(self):
        self.cloudfront = boto3.client('cloudfront')

    def create_distribution(self, video_id):
        """Create CloudFront distribution for video"""
        response = self.cloudfront.create_distribution(
            DistributionConfig={
                'CallerReference': video_id,
                'Origins': {
                    'Quantity': 1,
                    'Items': [{
                        'Id': f's3-{video_id}',
                        'DomainName': 'videos.s3.amazonaws.com',
                        'S3OriginConfig': {
                            'OriginAccessIdentity': ''
                        }
                    }]
                },
                'DefaultCacheBehavior': {
                    'TargetOriginId': f's3-{video_id}',
                    'ViewerProtocolPolicy': 'redirect-to-https',
                    'MinTTL': 86400,  # 1 day
                    'DefaultTTL': 604800,  # 7 days
                    'MaxTTL': 31536000,  # 1 year
                    'Compress': True
                },
                'Enabled': True
            }
        )

        return response['Distribution']['DomainName']

    def invalidate_cache(self, distribution_id, paths):
        """Invalidate CDN cache for updated content"""
        self.cloudfront.create_invalidation(
            DistributionId=distribution_id,
            InvalidationBatch={
                'Paths': {
                    'Quantity': len(paths),
                    'Items': paths
                },
                'CallerReference': str(uuid.uuid4())
            }
        )
```

---

## Storage Architecture

### Tiered Storage

```python
class VideoStorageManager:
    """
    Hot storage: Recently uploaded, popular videos (SSD)
    Warm storage: Moderate access (HDD)
    Cold storage: Rarely accessed (Glacier)
    """

    def archive_old_videos(self):
        """Move old videos to cheaper storage"""
        # Find videos not accessed in 90 days
        old_videos = self.db.query("""
            SELECT video_id, s3_key
            FROM videos
            WHERE last_accessed < NOW() - INTERVAL '90 days'
            AND storage_class = 'STANDARD'
        """)

        for video in old_videos:
            # Move to Glacier
            s3_client.copy_object(
                CopySource={'Bucket': 'videos', 'Key': video['s3_key']},
                Bucket='videos',
                Key=video['s3_key'],
                StorageClass='GLACIER'
            )

            # Update database
            self.db.update('videos', video['video_id'], {
                'storage_class': 'GLACIER'
            })
```

---

## Video Playback

### Streaming API

```python
@app.route('/api/videos/<video_id>/stream')
def stream_video(video_id):
    """
    Return HLS manifest URL
    CDN will serve actual video segments
    """
    video = db.get_video(video_id)

    # Increment view count (async)
    analytics_queue.publish({
        'event': 'video_view',
        'video_id': video_id,
        'user_id': request.user_id,
        'timestamp': datetime.utcnow().isoformat()
    })

    # Return CDN URL
    cdn_url = f"https://cdn.example.com/videos/{video_id}/master.m3u8"

    return jsonify({
        'video_id': video_id,
        'stream_url': cdn_url,
        'thumbnail': video['thumbnail'],
        'duration': video['duration'],
        'available_qualities': ['360p', '480p', '720p', '1080p']
    })
```

---

## Analytics and Monitoring

```python
class VideoAnalytics:
    def __init__(self, kafka_producer):
        self.producer = kafka_producer

    def track_view(self, video_id, user_id, duration_watched):
        """Track video view"""
        event = {
            'event_type': 'video_view',
            'video_id': video_id,
            'user_id': user_id,
            'duration_watched': duration_watched,
            'timestamp': datetime.utcnow().isoformat()
        }
        self.producer.send('video-analytics', event)

    def track_quality_switch(self, video_id, from_quality, to_quality):
        """Track quality changes for ABR optimization"""
        event = {
            'event_type': 'quality_switch',
            'video_id': video_id,
            'from_quality': from_quality,
            'to_quality': to_quality,
            'timestamp': datetime.utcnow().isoformat()
        }
        self.producer.send('video-analytics', event)

    def track_buffering(self, video_id, buffer_duration):
        """Track buffering events"""
        event = {
            'event_type': 'buffering',
            'video_id': video_id,
            'buffer_duration': buffer_duration,
            'timestamp': datetime.utcnow().isoformat()
        }
        self.producer.send('video-analytics', event)
```

---

## Optimizations

### 1. Predictive Prefetching

```python
def prefetch_next_segment(current_segment, bandwidth):
    """Prefetch next segments based on bandwidth"""
    if bandwidth > 5000000:  # 5 Mbps
        # High bandwidth - prefetch 3 segments ahead
        return [current_segment + 1, current_segment + 2, current_segment + 3]
    else:
        # Low bandwidth - only 1 segment ahead
        return [current_segment + 1]
```

### 2. Thumbnail Sprites

```python
def generate_thumbnail_sprite(video_id):
    """
    Create single image with multiple thumbnails
    Reduces HTTP requests for seeking preview
    """
    command = [
        'ffmpeg',
        '-i', f'video_{video_id}.mp4',
        '-vf', 'fps=1/10,scale=160:90,tile=10x10',
        f'sprite_{video_id}.jpg'
    ]
    subprocess.run(command)
```

---

## Real-World Examples

### YouTube
- **Upload**: Multipart upload, immediate processing
- **Transcoding**: Multiple resolutions (144p to 8K)
- **ABR**: DASH protocol
- **CDN**: Google's global CDN network

### Netflix
- **Encoding**: Per-title encoding optimization
- **ABR**: Custom algorithm based on network conditions
- **CDN**: Open Connect (own CDN infrastructure)
- **Caching**: Pre-populate popular content in ISP caches

---

## Interview Tips

### Key Points

1. **Upload**: Presigned URLs, multipart for large files
2. **Transcoding**: FFmpeg, multiple resolutions, HLS/DASH
3. **ABR**: Bandwidth measurement, quality switching
4. **CDN**: Multi-tier caching, edge locations
5. **Storage**: Tiered (hot/warm/cold), lifecycle policies
6. **Scalability**: Queue-based processing, horizontal scaling

### Common Questions

**Q: How do you handle millions of concurrent uploads?**
- Direct S3 upload with presigned URLs
- Queue-based asynchronous processing
- Auto-scaling transcoding workers

**Q: How do you reduce CDN costs?**
- Tiered storage (move old videos to cheaper storage)
- Compression
- Cache popular content longer
- Regional CDNs instead of global for less popular content

**Q: How do you ensure low latency playback globally?**
- CDN with edge locations
- ABR to adapt to network conditions
- Prefetching next segments
- HLS/DASH for efficient streaming

This video streaming design showcases handling large-scale media processing, global content delivery, and adaptive streaming - critical for modern video platforms.
