# Content Automation Workflow

> Automate video generation and multi-platform publishing for all content.

---

## Overview

This guide covers how to automatically generate videos and push content across all platforms with consistent branding.

---

## Video Generation Stack

### Recommended Tools by Content Type

| Content Type | Primary Tool | Backup | Why |
|--------------|--------------|--------|-----|
| **Talking Head / Avatar** | HeyGen | Synthesia | Best lip-sync, natural expressions |
| **Cinematic / B-roll** | Runway Gen-3 | Kling 2.6 | Fastest, most realistic |
| **Script-to-Video** | LTX Studio | InVideo AI | Handles long scripts (12k words) |
| **Quick Social Clips** | Canva AI | CapCut | Templates + speed |
| **YouTube Long-form** | Veo (Google) | Wan 2.6 | Native audio sync, 1080p |

### Tool Comparison

| Tool | Speed | Quality | Max Length | Audio | Price |
|------|-------|---------|------------|-------|-------|
| **Runway 4.5** | Fast | Excellent | Multi-shot | Yes | $12-76/mo |
| **HeyGen** | Fast | Excellent | 5 min | Yes (voice clone) | $24-180/mo |
| **Kling 2.6** | Slow (5-30 min) | Excellent | 2 min | Yes | $5-66/mo |
| **Veo (Google)** | Medium | Excellent | 60s | Yes (auto-sync) | Via Google Cloud |
| **LTX Studio** | Fast | Good | Long scripts | Yes | $24-96/mo |
| **InVideo AI** | Fast | Good | 15 min | Yes | $25-60/mo |

---

## Automated Publishing Stack

### Option 1: Developer API (Recommended)

**Best for full automation + custom workflows**

| Service | Platforms | Price | Best For |
|---------|-----------|-------|----------|
| [**Outstand**](https://www.outstand.so/) | 10+ (X, LinkedIn, IG, TikTok, YT, FB) | Custom | Enterprise, high volume |
| [**Late**](https://getlate.dev) | 13 platforms | Free tier + paid | Startups, developers |
| [**Post for Me**](https://www.postforme.dev/) | 9 platforms | $10/mo | Budget-friendly |
| [**Post Bridge**](https://www.post-bridge.com) | 9 platforms | $5/mo | Cheapest |
| [**Ayrshare**](https://github.com/ayrshare/social-post-api-python) | 10 platforms | $29-199/mo | Python devs |

**Example API Call (Late):**
```bash
curl -X POST https://api.getlate.dev/post \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "platforms": ["linkedin", "twitter", "tiktok", "instagram", "youtube"],
    "content": "The AI agent wars just exploded...",
    "media_url": "https://your-cdn.com/video.mp4",
    "schedule": "2026-01-16T09:00:00Z"
  }'
```

### Option 2: No-Code Automation

**Best for quick setup without coding**

| Tool | Integrations | Price | Best For |
|------|--------------|-------|----------|
| [**Postiz**](https://postiz.com/) | 9 platforms + Zapier/n8n/Make | Free (self-host) or $10/mo | Open source fans |
| **Buffer** | 8 platforms | $6-120/mo | Simple scheduling |
| **Hootsuite** | All major | $99-739/mo | Enterprises |
| **Later** | 7 platforms | $25-80/mo | Visual planning |

---

## Recommended Automation Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTENT AUTOMATION PIPELINE                   │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   CONTENT    │     │    VIDEO     │     │   PUBLISH    │
│    HUB       │────▶│  GENERATION  │────▶│   & TRACK    │
│  (GitHub)    │     │    (AI)      │     │    (API)     │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       ▼                    ▼                    ▼
  ┌─────────┐        ┌───────────┐        ┌───────────┐
  │ Scripts │        │  HeyGen   │        │   Late    │
  │ stored  │        │  Runway   │        │   API     │
  │ in repo │        │  LTX      │        │           │
  └─────────┘        └───────────┘        └───────────┘
                           │                    │
                           ▼                    ▼
                    ┌───────────┐        ┌───────────┐
                    │  Videos   │        │ LinkedIn  │
                    │  stored   │        │ Twitter   │
                    │  in CDN   │        │ YouTube   │
                    └───────────┘        │ TikTok    │
                                         │ Instagram │
                                         └───────────┘
```

---

## Step-by-Step Setup

### Phase 1: Video Generation (HeyGen + Runway)

**1. Create HeyGen Account**
- Sign up at [heygen.com](https://heygen.com)
- Create your AI avatar (upload video of yourself or use stock)
- Clone your voice for authentic talking-head videos

**2. Generate Avatar Videos**
```
For each post in content-hub/posts/:
  1. Copy the YouTube/TikTok script
  2. Paste into HeyGen
  3. Select your avatar
  4. Generate video
  5. Download to content-hub/assets/
```

**3. Create B-roll with Runway**
- Sign up at [runwayml.com](https://runwayml.com)
- Use Gen-3 Alpha for cinematic clips
- Generate 5-10 second clips for key points
- Stitch together in editor (CapCut/DaVinci)

### Phase 2: Automated Publishing (Late API)

**1. Get API Access**
```bash
# Sign up at getlate.dev and get API key
export LATE_API_KEY="your_api_key_here"
```

**2. Create Publishing Script**

```python
#!/usr/bin/env python3
"""
publish_content.py - Automated multi-platform publisher
"""

import requests
import json
from datetime import datetime, timedelta

API_KEY = "your_late_api_key"
BASE_URL = "https://api.getlate.dev"

def publish_post(content_id: str, platforms: list, content: dict, schedule: str = None):
    """
    Publish content to multiple platforms

    Args:
        content_id: Unique identifier (e.g., "001")
        platforms: List of platforms ["linkedin", "twitter", "tiktok", "instagram", "youtube"]
        content: Dict with text, media_url, etc.
        schedule: ISO-8601 datetime or None for immediate
    """

    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json"
    }

    payload = {
        "platforms": platforms,
        "content": content["text"],
        "media_url": content.get("video_url"),
        "metadata": {
            "content_id": content_id,
            "campaign": content.get("campaign", "default")
        }
    }

    if schedule:
        payload["schedule"] = schedule

    response = requests.post(f"{BASE_URL}/post", headers=headers, json=payload)
    return response.json()


def publish_ai_agents_post():
    """Publish the AI Agents Revolution content"""

    # LinkedIn
    linkedin_result = publish_post(
        content_id="001",
        platforms=["linkedin"],
        content={
            "text": open("content-hub/posts/2026-01-15-ai-agents-revolution.md").read(),
            "video_url": "https://your-cdn.com/ai-agents-linkedin.mp4"
        },
        schedule="2026-01-16T14:00:00Z"  # 9 AM EST
    )

    # Twitter Thread
    twitter_result = publish_post(
        content_id="001",
        platforms=["twitter"],
        content={
            "text": "🚨 BREAKING: OpenAI, Anthropic, and Google just joined forces...",
            "video_url": "https://your-cdn.com/ai-agents-twitter.mp4"
        },
        schedule="2026-01-16T15:00:00Z"  # 10 AM EST
    )

    # TikTok
    tiktok_result = publish_post(
        content_id="001",
        platforms=["tiktok"],
        content={
            "text": "OpenAI + Anthropic + Google = 🤯 #AIAgents #Tech",
            "video_url": "https://your-cdn.com/ai-agents-tiktok.mp4"
        },
        schedule="2026-01-16T23:00:00Z"  # 6 PM EST
    )

    # YouTube
    youtube_result = publish_post(
        content_id="001",
        platforms=["youtube"],
        content={
            "text": "OpenAI, Anthropic & Google Just Joined Forces (AI Agents Explained)",
            "video_url": "https://your-cdn.com/ai-agents-youtube.mp4",
            "description": "Full description with timestamps..."
        },
        schedule="2026-01-17T17:00:00Z"  # 12 PM EST
    )

    # Instagram
    instagram_result = publish_post(
        content_id="001",
        platforms=["instagram"],
        content={
            "text": "The AI agent wars just exploded 🔥...",
            "video_url": "https://your-cdn.com/ai-agents-reel.mp4"
        },
        schedule="2026-01-17T16:00:00Z"  # 11 AM EST
    )

    return {
        "linkedin": linkedin_result,
        "twitter": twitter_result,
        "tiktok": tiktok_result,
        "youtube": youtube_result,
        "instagram": instagram_result
    }


if __name__ == "__main__":
    results = publish_ai_agents_post()
    print(json.dumps(results, indent=2))
```

**3. Run Publisher**
```bash
python publish_content.py
```

### Phase 3: GitHub Actions Automation (Optional)

Create `.github/workflows/publish-content.yml`:

```yaml
name: Publish Content

on:
  # Manual trigger
  workflow_dispatch:
    inputs:
      content_id:
        description: 'Content ID to publish'
        required: true
        default: '001'

  # Or trigger on new post files
  push:
    paths:
      - 'content-hub/posts/**.md'

jobs:
  generate-and-publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install requests

      - name: Generate videos
        env:
          HEYGEN_API_KEY: ${{ secrets.HEYGEN_API_KEY }}
          RUNWAY_API_KEY: ${{ secrets.RUNWAY_API_KEY }}
        run: python scripts/generate_videos.py ${{ github.event.inputs.content_id }}

      - name: Publish to platforms
        env:
          LATE_API_KEY: ${{ secrets.LATE_API_KEY }}
        run: python scripts/publish_content.py ${{ github.event.inputs.content_id }}

      - name: Update tracker
        run: python scripts/update_tracker.py ${{ github.event.inputs.content_id }}

      - name: Commit tracker updates
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add content-hub/distribution/tracker.md
          git commit -m "Update tracker for content ${{ github.event.inputs.content_id }}"
          git push
```

---

## Quick Start Checklist

### Today
- [ ] Sign up for [HeyGen](https://heygen.com) (avatar videos)
- [ ] Sign up for [Late](https://getlate.dev) (publishing API)
- [ ] Create your AI avatar in HeyGen

### This Week
- [ ] Generate first video from AI Agents script
- [ ] Connect social accounts to Late
- [ ] Test publish to one platform

### Ongoing
- [ ] Set up GitHub Actions for automation
- [ ] Build video generation pipeline
- [ ] Track performance in tracker.md

---

## Cost Estimate (Monthly)

| Service | Plan | Cost |
|---------|------|------|
| HeyGen | Creator | $24/mo |
| Runway | Standard | $12/mo |
| Late API | Starter | $0-20/mo |
| CDN (Cloudflare R2) | Pay-as-you-go | ~$5/mo |
| **Total** | | **~$41-61/mo** |

---

## Platform-Specific Requirements

### LinkedIn
- Video: MP4, 3s-10min, max 5GB
- Optimal: 1920x1080 or 1080x1920 (vertical)
- Best time: Tue-Thu 9-11 AM

### Twitter/X
- Video: MP4, max 2:20, max 512MB
- Optimal: 1280x720 or 720x1280
- Best time: Mon-Fri 8 AM-4 PM

### YouTube
- Video: MP4, any length, max 256GB
- Optimal: 1920x1080 (horizontal)
- Shorts: Under 60s, 1080x1920

### TikTok
- Video: MP4, 5s-10min, max 4GB
- Optimal: 1080x1920 (9:16)
- Best time: Tue-Thu 7-9 PM

### Instagram
- Reels: MP4, 15-90s, 1080x1920
- Stories: 15s max per story
- Best time: Mon-Fri 11 AM-1 PM

---

## API Keys Needed

Store these in `.env` or GitHub Secrets:

```env
# Video Generation
HEYGEN_API_KEY=your_key_here
RUNWAY_API_KEY=your_key_here

# Publishing
LATE_API_KEY=your_key_here

# Optional: Individual platform APIs
LINKEDIN_ACCESS_TOKEN=your_token
TWITTER_API_KEY=your_key
YOUTUBE_API_KEY=your_key
TIKTOK_ACCESS_TOKEN=your_token
```

---

## Next Steps

1. **Generate your first video** → HeyGen with the YouTube script
2. **Test the API** → Single post to LinkedIn via Late
3. **Scale up** → Automate with GitHub Actions

---

## Resources

- [HeyGen Documentation](https://docs.heygen.com)
- [Runway API Docs](https://docs.runwayml.com)
- [Late API Reference](https://getlate.dev/docs)
- [Postiz Self-Hosting Guide](https://docs.postiz.com)

---

*Last updated: January 15, 2026*
