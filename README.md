
# Video Editing / Compositing Headless API

## Overview

A headless API service that accepts scene-by-scene video descriptions and produces edited marketing videos with voice-over, music, and visual effects. There is an AI component (i.e. LLMs analyzing input assets and making fine-grained decisions about which assets to use, which ffmpeg commands to run to edit them) and then the media handling and final stitching together component.

The overall goal is to have a stable, high quality automated video-editor that can turn a detailed scenario into a finished production-ready .mp4 that can be uploaded straight to tiktok / youtube shorts.

## API Endpoints

### POST /api/v1/video/compose

Creates a video composition job.

**Authentication:** Bearer token in Authorization header (stored in .env)

**Request Body:** JSON schema:

```json
{
  "metadata": {
    "webhookUrl": "https://some-domain.com/webhook/video-complete",
    "jobId": "optional-client-provided-job-id",
    "priority": "normal|high"
  },
  
  "videoConfig": {
    "output": {
      "aspectRatio": "9:16", // for now only vertical social videos
      "targetDuration": 30, // duration in seconds
      "format": "mp4",
      "resolution": "1080x1920",
      "fps": 30
    },
    
    "voiceConfig": {
      "service": "elevenLabs|openaiTTS|googleTTS",
      "voiceId": "voice-specific-id",
      "params": { //these can be different depending on TTS service used, treat as arbitrary JSON, validate on a per-TTS service requirements basis
        "speed": 1.0,
        "pitch": 1.0,
        "stability": 0.75,
        "similarityBoost": 0.75
      }
    },
    
    "audioConfig": {
      "backgroundMusic": {
        "url": "https://s3.url/music.mp3",
        "volume": 0.3,
        "fadeIn": 2,
        "fadeOut": 2,
        "loop": true
      },
      "masteringProfile": "youtube|tiktok"
    },
    
    "brandingConfig": {
      "logo": {
        "url": "https://s3.url/logo.png",
        "position": "bottom-right",
        "size": "small|medium|large",
        "opacity": 0.9,
        "alwaysVisible": true, //whether it should always be there throughout the final video or only on some shots to be decided by the LLM driving the editor
        "padding": 20
      }
    },
    
    "bookendConfig": {
      "intro": {
        "url": "https://s3.url/intro.mp4", //optional video to be prepended
        "duration": null
      },
      "outro": {
        "url": "https://s3.url/outro.mp4", //optional video to be added to the end
        "duration": null
      }
    }
  },
  
  "assets": {
    "heroPortrait": "https://s3.url/portrait.jpg", // photo to be used for talking head generation
    "productImages": [
      "https://s3.url/product1.jpg",
      "https://s3.url/product2.jpg"
    ],
    "productVideos": [
      "https://s3.url/product-demo1.mp4",
      "https://s3.url/product-demo2.mp4"
    ],
    "stockFootageKeywords": ["technology", "lifestyle"]
  },
  
  "script": {
    "fullVoiceOver": "Here's the complete voice over text...",
    "scenes": [
      {
        "sceneId": "scene-001",
        "type": "talkingHead|talkingHeadProduct|productShowcase|bRoll|transition",
        "duration": {
          "target": 5,
          "min": 3,
          "max": 7
        },
        "voiceOver": {
          "text": "Welcome to our product review...",
          "emotion": "excited|neutral|serious|friendly",
          "emphasis": ["product", "amazing"]
        },
        "visuals": { //only needed for broll shots
          "background": "blur|product|solid|gradient",
          "productAssets": ["auto|specific-asset-urls"],
          "generativePrompt": "modern office environment with tech gadgets",
          "stockFootage": {
            "keywords": ["technology", "innovation"],
            "duration": 5
          }
        },
        "subtitles": {
          "enabled": true,
          "style": "minimal|bold|animated",
          "position": "bottom|middle",
          "customStyle": {
            "fontFamily": "Arial",
            "fontSize": 48,
            "color": "#FFFFFF",
            "backgroundColor": "#000000",
            "backgroundOpacity": 0.7
          }
        },
        "transitions": {
          "in": "fade|cut|slide|zoom",
          "out": "fade|cut|slide|zoom",
          "duration": 0.5
        }
      }
    ]
  }
}
```

**Response (202 Accepted):**

```json
{
  "status": "accepted",
  "jobId": "job-uuid-123",
  "estimatedCompletionTime": "2024-01-15T10:30:00Z",
  "message": "Job queued successfully"
}

```

**Error Responses:**

-   400: Invalid request body
-   401: Invalid API key
-   422: Validation error with specific field errors
-   429: Rate limit exceeded
-   500: Internal server error

### GET /api/v1/video/status/{jobId} (Optional)

Check job status.

**Response:**

```json
{
  "jobId": "job-uuid-123",
  "status": "queued|processing|completed|failed",
  "progress": 75,
  "currentStep": "generating_voiceover",
  "error": "error message if failed"
}

```

## Webhook Callback

When processing completes, the service calls the webhook URL specified in the request using the Bearer API token stored in the .env

In case of status: failed, we need a description of what happened.
In case of success, the response should be as specified below:

**Webhook Request:**

```json
{
  "jobId": "job-uuid-123",
  "status": "completed|failed",
  "outputs": {
    "fullVideo": {
      "url": "https://s3.url/final/video.mp4",
      "duration": 30.5,
      "size": 15728640,
      "resolution": "1080x1920"
    },
    "scenes": [
      {
        "sceneId": "scene-001",
        "url": "https://s3.url/scenes/scene-001.mp4",
        "duration": 5.2
      }
    ],
    "audio": {
      "voiceOnly": "https://s3.url/audio/voice.mp3",
      "voiceWithMusic": "https://s3.url/audio/mixed.mp3"
    }
  },
  "error": "error details if failed",
  "completedAt": "2024-01-15T10:45:00Z"
}

```

## Processing Behavior

### Scene Types

-   **talkingHead**: Generate animated talking head using provided portrait and voice-over. It's important to generate the voice first and then use the voice track as the driver for the talking head.
-   **talkingHeadProduct**: Talking head composited over product background
-   **productShowcase**: Product images/videos with voice-over
-   **bRoll**: Stock or AI-generated supplementary footage. Stock preferred. (check out https://www.pexels.com/ or similar)
-   **transition**: Pure transition between scenes

### Audio Processing

1.  Generate voice-over for each scene using specified TTS service
2.  Apply emotion and emphasis parameters where supported
3.  Mix background music at appropriate levels
4.  Master final audio for target platform (TikTok/YouTube loudness standards)

### Visual Processing

1.  **Asset Analysis**: Analyze provided product images/videos and select most appropriate ones for each scene
2.  **Aspect Ratio**: Ensure all content matches target aspect ratio (9:16 for vertical)
3.  **Talking Heads**: Generate using TTS and HeyGen API with provided portrait
4.  **Subtitles**: Generate and burn into video when enabled
5.  **Branding**: Overlay logo with specified position and visibility settings
6.  **Transitions**: Apply smooth transitions between scenes

### Output Requirements

1.  All media uploaded to S3 with secure URLs
2.  Final video in MP4 format at specified resolution
3.  Separate deliverables for scenes, audio tracks
4.  Respect target duration while prioritizing content completeness

## Input Validation

### Required Fields

-   metadata.webhookUrl
-   videoConfig.output.aspectRatio
-   assets.heroPortrait (for talking head scenes)
-   script.scenes (at least one scene)

## Service Integrations

The service must integrate with:

-   **HeyGen**: For talking head generation
-   **TTS Services**: ElevenLabs, OpenAI TTS, or Google TTS as specified
-   **Google Veo 3**: For generative b-roll (when available)
-   **Stock Footage API**: As fallback for b-roll
-   **Vision LLM**: For analyzing and selecting product assets (suggest Gemini Flash family of models)

## Robustness requirements

-   Job acceptance: < 2 seconds
-   10 concurrent jobs
-   100 jobs per hour
-   Webhook retries: 3 attempts with exponential backoff
-   Processing time: < 10 minutes for videos up to 60 seconds
