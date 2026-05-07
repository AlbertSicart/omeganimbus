# OmegaNimbus Day 5: Computer Vision & AI Image Analysis

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** Rekognition · Bedrock · Lambda · API Gateway · CloudFront · S3  
**Date:** May 2026

---

## Overview

This session focused on two objectives:

1. **AWS Rekognition** — real-time object detection, face analysis, and text extraction from user-uploaded images
2. **Amazon Bedrock + Claude Haiku 4.5** — AI-generated image descriptions combining Rekognition output with a large language model

The result is a live interactive tool at `omeganimbus.com/rekognition.html` where any image can be analyzed in real time using a fully serverless AWS pipeline.

---

## Architecture

```
Browser (POST /analyze with base64 image)
         │
         ▼
API Gateway HTTP API (omeganimbus-api-cfn)
         │  POST /analyze
         ▼
Lambda (omeganimbus-rekognition) — Python 3.12 / arm64
         │
         ├── Rekognition (eu-west-1)
         │     ├── detect_labels   → objects, scenes, concepts
         │     ├── detect_text     → OCR
         │     └── detect_faces    → face count + attributes
         │
         └── Bedrock (eu-west-1)
               └── Claude Haiku 4.5 → AI description
                         │
                         ▼
Returns JSON → rendered in rekognition.html
```

**Note on regions:** Rekognition is not available in eu-north-1 (Stockholm). Both Rekognition and Bedrock run in eu-west-1 (Ireland). The Lambda is deployed in eu-north-1 but makes cross-region API calls.

---

## Part 1 — Lambda Function

**Function name:** `omeganimbus-rekognition`  
**Runtime:** Python 3.12 / arm64  
**Timeout:** 30 seconds (increased from default 3s — Rekognition + Bedrock calls require more time)  
**IAM policies:** `AmazonRekognitionReadOnlyAccess` + `AmazonBedrockFullAccess`

```python
import json
import boto3
import base64

def lambda_handler(event, context):
    if event.get('requestContext', {}).get('http', {}).get('method') == 'OPTIONS':
        return {
            'statusCode': 200,
            'headers': {
                'Access-Control-Allow-Origin': 'https://omeganimbus.com',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Methods': 'POST, OPTIONS'
            },
            'body': ''
        }

    try:
        body = json.loads(event.get('body', '{}'))
        image_data = body.get('image', '')

        if ',' in image_data:
            image_data = image_data.split(',')[1]
        image_bytes = base64.b64decode(image_data)

        rek = boto3.client('rekognition', region_name='eu-west-1')

        labels_response = rek.detect_labels(
            Image={'Bytes': image_bytes},
            MaxLabels=15,
            MinConfidence=70
        )
        text_response = rek.detect_text(
            Image={'Bytes': image_bytes}
        )
        faces_response = rek.detect_faces(
            Image={'Bytes': image_bytes},
            Attributes=['ALL']
        )

        labels = [{'name': l['Name'], 'confidence': round(l['Confidence'], 1)}
                  for l in labels_response['Labels']]
        texts = [t['DetectedText'] for t in text_response['TextDetections']
                 if t['Type'] == 'LINE']
        faces = len(faces_response['FaceDetails'])

        # ─── BEDROCK DESCRIPTION ───
        label_names = [l['name'] for l in labels]
        face_str = f"{faces} face{'s' if faces != 1 else ''} detected" if faces > 0 else "no faces detected"
        text_str = f"Text found: {', '.join(texts)}" if texts else "no text detected"

        prompt = f"""You are a technical image analysis system. Computer vision has detected the following in an image:
- Detected objects and scenes: {', '.join(label_names)}
- Faces detected: {face_str}
- Text found: {text_str}

Based strictly on this data, write a precise 2-3 sentence description of what the image most likely shows.
Rules:
- No markdown, no headers, no bullet points
- No titles or preambles like "Image Analysis" or "This image"
- Be specific and analytical, not generic
- If the combination of labels suggests a specific context (e.g. a logo, artwork, document, scene), state it directly
- Plain prose only"""

        bedrock = boto3.client('bedrock-runtime', region_name='eu-west-1')
        bedrock_response = bedrock.invoke_model(
            modelId='eu.anthropic.claude-haiku-4-5-20251001-v1:0',
            contentType='application/json',
            accept='application/json',
            body=json.dumps({
                'anthropic_version': 'bedrock-2023-05-31',
                'max_tokens': 200,
                'messages': [{'role': 'user', 'content': prompt}]
            })
        )
        bedrock_body = json.loads(bedrock_response['body'].read())
        description = bedrock_body['content'][0]['text']

        return {
            'statusCode': 200,
            'headers': {
                'Access-Control-Allow-Origin': 'https://omeganimbus.com',
                'Access-Control-Allow-Headers': 'Content-Type',
                'Access-Control-Allow-Methods': 'POST, OPTIONS'
            },
            'body': json.dumps({
                'labels': labels,
                'text': texts,
                'faces': faces,
                'description': description
            })
        }

    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {
                'Access-Control-Allow-Origin': 'https://omeganimbus.com',
            },
            'body': json.dumps({'error': str(e)})
        }
```

---

## Part 2 — API Gateway

Added route `POST /analyze` to the existing `omeganimbus-api-cfn` HTTP API with Lambda proxy integration. Resource-based policy added to the Lambda:

```
Source ARN: arn:aws:execute-api:eu-north-1:247906201225:judzwkiy9h/*/*/analyze
```

---

## Part 3 — Frontend (rekognition.html)

Live at `omeganimbus.com/rekognition.html`.

**Features:**
- Drag & drop or click-to-browse image upload (JPG, PNG, WEBP, max 5MB)
- Image preview before analysis
- Scan animation while processing
- AI description (Claude Haiku 4.5 via Bedrock) — shown first
- Labels grid with confidence score and visual confidence bar
- Face count
- Extracted text (OCR)
- "Upload new image" reset button — clears state without page refresh

---

## Debugging Notes

**Timeout error (3s default)**
Rekognition + Bedrock calls take 3-8 seconds combined. Lambda default timeout is 3s. Fixed by increasing timeout to 30s in Configuration → General configuration.

**Rekognition not available in eu-north-1**
`Could not connect to the endpoint URL: https://rekognition.eu-north-1.amazonaws.com`
Rekognition is not deployed in Stockholm. Switched client to `eu-west-1`.

**Bedrock model Legacy error**
`anthropic.claude-3-haiku-20240307-v1:0` is marked as Legacy and requires active usage in the last 30 days. Switched to Claude Haiku 4.5.

**On-demand throughput not supported**
New Anthropic models on Bedrock require inference profiles instead of direct model IDs. Fixed by using the `eu.` prefix: `eu.anthropic.claude-haiku-4-5-20251001-v1:0`

**Anthropic use case form required**
First-time use of Anthropic models on Bedrock requires submitting a use case form. After submission, access is granted within ~15 minutes.

**Markdown in AI description**
Claude was prefixing responses with `# Image Analysis` header. Fixed by adding explicit prompt instructions: "No markdown, no headers, no bullet points. Plain prose only."

---

## Bedrock Model Selection

| Model | ID | Cost | Notes |
|---|---|---|---|
| Claude 3 Haiku | `anthropic.claude-3-haiku-20240307-v1:0` | Low | Legacy — deprecated |
| Claude Haiku 4.5 | `eu.anthropic.claude-haiku-4-5-20251001-v1:0` | Low | ✓ Active — used in project |
| Claude 3.5 Sonnet | `eu.anthropic.claude-sonnet-4-5-...` | Medium | Overkill for this use case |
| Amazon Nova Micro | `amazon.nova-micro-v1:0` | Very low | AWS native alternative |

Claude Haiku 4.5 selected for balance of quality, speed, and cost. The inference profile prefix `eu.` routes requests through European infrastructure.

---

## Cost Summary

| Service | Usage | Cost |
|---|---|---|
| Rekognition | $0.001 per image (detect_labels + detect_text + detect_faces) | ~$0 at portfolio scale |
| Bedrock Claude Haiku 4.5 | ~$0.0008 per 1K input tokens | ~$0 at portfolio scale |
| Lambda | <1M requests/month | ~$0 |
| API Gateway | <1M calls/month | ~$0 |

**Total: ~$0/month at portfolio traffic levels**

---

## Key Concepts Practiced

- AWS Rekognition API: `detect_labels`, `detect_text`, `detect_faces`
- Amazon Bedrock: `invoke_model` with Anthropic Claude via inference profiles
- Cross-region API calls from Lambda (eu-north-1 → eu-west-1)
- Base64 image encoding/decoding for binary data transfer
- Lambda timeout configuration for long-running operations
- Bedrock model versioning and inference profile routing
- Prompt engineering for structured plain-text output
- Frontend drag & drop file handling with FileReader API
- Stateful reset pattern without page refresh

---

## What's Next

| Feature | Details | Priority |
|---|---|---|
| CloudFront cache invalidation in pipeline | Auto-invalidate on every deploy | High |
| Rekognition page improvements | Reset button polish, loading UX | Medium |
| Amazon Lex chatbot | Interactive assistant on the site | Medium |
| WAF + Shield | DDoS protection on CloudFront | Low |
| Study Notes | CCP notes section polish | Medium |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*
