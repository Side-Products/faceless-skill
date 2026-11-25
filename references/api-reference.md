# Faceless.so API reference

> Generated from src/backend/api/v1/spec/registry.mjs by npm run generate:agents. Do not edit by hand.

Base URL: `https://faceless.so/api/v1`

Authentication: `Authorization: Bearer fl_live_...` (or `X-API-Key`). Create keys at https://faceless.so/developers.

Envelope: success responses are `{ "success": true, "data": ..., "pagination"?: { page, limit, total } }`. Errors are `{ "success": false, "error": { "type", "message" } }` with types: `invalid_input`, `unauthorized`, `forbidden_scope`, `not_found`, `conflict`, `insufficient_credits`, `payment_failed`, `usage_limit_reached`, `rate_limited`, `internal_error`.

## me

### Identify the caller

`GET /me`

Returns the team this API key belongs to, its credit balance, subscription plan and the scopes granted to the key. Call this first to verify authentication and discover the team context every other operation runs in.

- Scopes: any valid key
- Credits: none
- CLI: `faceless whoami`
- MCP tool: `faceless_get_me`

Example:

```bash
curl -s "https://faceless.so/api/v1/me" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "team": {
      "name": "My Channel",
      "credits": 340
    },
    "plan": "creator",
    "auth": {
      "via": "api_key",
      "scopes": [
        "videos:read",
        "videos:write"
      ]
    }
  }
}
```

## video-defaults

### Read the team's video defaults

`GET /video-defaults`

Returns the team's standing defaults for every new video made one at a time (POST /videos, POST /videos/audio, the Create page): the picture rule every picture follows (off, nature, no_people_animals_text, or custom with the team's own words), the caption style, size and weight, and imageBlur, the blur for the team's own pictures placed with faceless_place_images when a call does not say. The picture rule is applied by Faceless on every video, so a request does not need to repeat it.

- Scopes: `videos:read`
- Credits: none
- CLI: `faceless defaults`
- MCP tool: `faceless_get_video_defaults`

Example:

```bash
curl -s "https://faceless.so/api/v1/video-defaults" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "pictureRule": {
      "mode": "custom",
      "text": "1. Every scene matches the line being spoken.\n2. No faces.\n3. No text in images."
    },
    "captions": {
      "theme": "IMAN",
      "size": "extra_large",
      "weight": "bold"
    },
    "imageBlur": "none"
  }
}
```

### Set the team's video defaults

`PATCH /video-defaults`

Sets the team's defaults for every new video made one at a time, the same settings as Team settings > Video defaults. captions sets the caption style, size and weight (each part left out is unchanged; null resets captions to the product default); a video's own captions on create still win. pictureRule sets the rule every picture follows: mode off, nature, no_people_animals_text, or custom with text (the team's own words, up to 1,200 characters, stored as written). Every picture of every new video is checked against it, and a video with a picture that still breaks it waits for the team's review instead of posting. imageBlur (none, light or strong) blurs the team's own pictures placed with faceless_place_images or faceless_create_video_from_audio images when neither the picture nor the call sets blur. Free.

- Scopes: `videos:write`
- Credits: none
- CLI: `faceless defaults set`
- MCP tool: `faceless_update_video_defaults`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `captions` | object | no | The default caption look. null resets it |
| `pictureRule` | object | no | The picture rule every new video follows |
| `imageBlur` | `none` \| `light` \| `strong` | no | Default blur for the team's own placed pictures (faceless_place_images) when the call does not set blur. none (clear), light (softened: a stylistic blur; names, headings and most text can still be read) or strong (text, names and faces cannot be read; only colours and rough shapes remain). Use strong to hide names, faces or private details in a screenshot without editing it first |

Example:

```bash
curl -s -X PATCH "https://faceless.so/api/v1/video-defaults" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"captions":{"size":"extra_large","weight":"bold"}}'
```

Response:

```json
{
  "success": true,
  "data": {
    "pictureRule": {
      "mode": "off",
      "text": null
    },
    "captions": {
      "theme": null,
      "size": "extra_large",
      "weight": "bold"
    },
    "imageBlur": "none"
  }
}
```

## credits

### Credit balance and history

`GET /credits`

Returns the team's current credit balance and a paginated ledger of credit transactions (purchases, subscription grants, video spends, refunds). Video creation charges credits at creation time; rendering and publishing are free. Costs: storyboard 20 to 40 by length, motion_lite 60, motion_pro 120 credits per video.

- Scopes: `credits:read`
- Credits: none
- CLI: `faceless credits`
- MCP tool: `faceless_get_credits`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `page` | integer | no | Page number, starting at 1 |
| `limit` | integer | no | Items per page (max 100) |

Example:

```bash
curl -s "https://faceless.so/api/v1/credits" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "balance": 340,
    "history": [
      {
        "credits": -50,
        "type": "spending",
        "description": "Faceless video (motion_lite)",
        "createdAt": "2026-07-30T10:00:00.000Z"
      }
    ]
  },
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

### Buy a credit pack with an agent payment token

`POST /credits`

Buys a credit pack and adds the credits to the team immediately, paid with a Stripe Shared Payment Token (spt_...) that the customer approved in their Link agent wallet. Packs: pack_10 ($10, 100 credits), pack_25 ($25, 275), pack_50 ($50, 600); active subscribers get more credits per pack at their plan rate. Request the token for the Faceless.so Stripe profile (profile_61UDMKC11wsLfeXaMA6UDMKCL0SQKAF3aHKCFz33IPL6) with a USD max_amount of at least the pack price. Tokens are single use: retrying with the same token never charges twice. A declined, expired or already used token returns 402 payment_failed.

- Scopes: `credits:write`
- Credits: none (adds credits; charges the payment token the pack price in USD)
- Rate limit: 5 per 60s
- Supports `Idempotency-Key` header
- CLI: `faceless buy-credits`
- MCP tool: `faceless_buy_credits`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `packId` | `pack_10` \| `pack_25` \| `pack_50` | yes | Credit pack to buy: pack_10 ($10), pack_25 ($25) or pack_50 ($50) |
| `paymentToken` | string | yes | Stripe Shared Payment Token granted to Faceless.so by the customer's agent wallet |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/credits" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"packId":"pack_10","paymentToken":"spt_1Rx2abc"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "packId": "pack_10",
    "amountUsd": 10,
    "creditsAdded": 100,
    "balance": 440,
    "paymentIntentId": "pi_3Rx2abc"
  }
}
```

## videos

### Create a faceless video from a script

`POST /videos`

Creates an AI faceless video from a finished narration script: TTS voiceover, AI-generated visuals per scene, captions and optional background music. Generation is asynchronous; the response returns the video id immediately. Poll GET /videos/{id}/status until status is completed, then call POST /videos/{id}/render to produce the final MP4. The team's picture rule (GET /video-defaults) applies to every picture without being repeated here, and the captions follow the team's default look unless captions is set. Credits are charged now, at creation.

- Scopes: `videos:write`
- Credits: 20 credits under 60 s, 30 for 60-89 s, 40 from 90 s (storyboard), 60 (motion_lite) or 120 (motion_pro) per video, charged at creation
- Rate limit: 10 per 60s
- Supports `Idempotency-Key` header
- CLI: `faceless videos create`
- MCP tool: `faceless_create_video`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `script` | string | yes | The full narration script the video is generated from |
| `voiceId` | string | yes | TTS voice id for the narration. Full list: GET /voices |
| `model` | `storyboard` \| `motion_lite` \| `motion_pro` | no | Generation model: storyboard (20 credits under 60 s, 30 for 60-89 s, 40 from 90 s, static scenes), motion_lite (60, animated transitions), motion_pro (120, cinematic) |
| `style` | string | no | Visual style for generated imagery. Full list: GET /options?kind=styles |
| `language` | string | no | Script language, e.g. English. Full list: GET /options?kind=languages |
| `name` | string | no | Project name; defaults to the start of the script |
| `enableBackgroundMusic` | boolean | no | Mix background music under the narration. Omit to use the team's default (Team settings > Video defaults), else music is on for storyboard and motion |
| `enableSoundEffects` | boolean | no | Automatic sound effects (whooshes, hits and an ambience bed). false makes a video with only the voice and any music. Omit to use the team's default (Team settings > Video defaults), else on |
| `masterStyle` | string | no | Extra style directive applied to every generated scene |
| `globalNegativePrompt` | string | no | Things the image model should avoid in every scene |
| `captions` | object | no | This video's caption style, size and weight. Each part left out follows the team's default (Team settings > Video defaults > Captions, or faceless_update_video_defaults), else the product default |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/videos" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"script":"Did you know the ocean has lakes and rivers of its own? ...","voiceId":"EXAVITQu4vr4xnSDxMaL","model":"storyboard"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e801",
    "name": "Did you kn...",
    "model": "storyboard",
    "status": "processing",
    "creditsUsed": 20,
    "statusUrl": "/api/v1/videos/665f1b2a9c31a2b3c4d5e801/status"
  }
}
```

### Caption an existing video or audio file

`POST /videos/captions`

Creates a project from a video or audio URL you already have: transcribes it, adds styled captions and emojis, and makes it editable. Use this to caption existing footage, e.g. a video of you talking to the camera, rather than generate new AI visuals (for that, use POST /videos). Files up to 30 minutes long; it uses no credits. Processing is asynchronous; poll GET /videos/{id}/status.

- Scopes: `videos:write`
- Credits: none
- Rate limit: 10 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless videos captions`
- MCP tool: `faceless_create_caption_video`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `videoUrl` | string | no | Public URL of the video file to caption |
| `audioUrl` | string | no | Public URL of an audio file to turn into a captioned video |
| `name` | string | no | Project name |
| `language` | string | no | Spoken language of the file, e.g. English |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/videos/captions" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"videoUrl":"https://example.com/clip.mp4","language":"English"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e802",
    "status": "processing",
    "statusUrl": "/api/v1/videos/665f1b2a9c31a2b3c4d5e802/status"
  }
}
```

### Create a faceless video from your own voiceover recording

`POST /videos/audio`

Creates an AI faceless video over a voiceover you recorded yourself: AI-generated visuals for every line, captions, and no AI voice. Pass a public link to the audio file; a Google Drive share link (set to "Anyone with the link") or a Dropbox link works as it is. Any common phone format (m4a, mp3, wav, ogg, webm) up to 10 minutes; it is levelled to narration loudness. The same art direction as POST /videos applies: style, masterStyle and globalNegativePrompt (a people ban, e.g. "people, hands, animals", keeps every frame free of them and each picture is checked). Pass the words you say as script so the captions spell them your way (recitation and names included). Music and sound effects follow the team's Video defaults unless set here, and so do the captions (style, size and weight; pass captions to choose for this video). The team's picture rule (GET /video-defaults) applies to every picture without being repeated here. To show your own pictures (screenshots, photos) at chosen moments, pass images: each at a time or at words said, as an overlay card or full screen; the rest stays generated visuals (placing pictures costs nothing extra). Generation is asynchronous; poll GET /videos/{id}/status (it also reports which pictures were placed), then POST /videos/{id}/render, then schedule as usual. Credits are charged now, at creation: up to 90 seconds the same as POST /videos; a longer recording costs the model's price for every started minute (Storyboard: 2 minutes 40 credits, 3 minutes 60, 10 minutes 200).

- Scopes: `videos:write`
- Credits: 20 credits under 60 s, 30 for 60-89 s, 40 from 90 s (storyboard), 60 (motion_lite) or 120 (motion_pro) per video up to 90 seconds; a longer recording (up to 10 minutes) costs that price for every started minute. Charged at creation
- Rate limit: 10 per 60s
- Supports `Idempotency-Key` header
- CLI: `faceless videos from-audio`
- MCP tool: `faceless_create_video_from_audio`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `audioUrl` | string | yes | Public link to your recording: a Google Drive share link ("Anyone with the link"), a Dropbox link or a direct file URL. Up to 10 minutes (over 90 seconds is charged per started minute) |
| `model` | `storyboard` \| `motion_lite` \| `motion_pro` | no | Generation model: storyboard (20 credits under 60 s, 30 for 60-89 s, 40 from 90 s, static scenes), motion_lite (60, animated transitions), motion_pro (120, cinematic) |
| `style` | string | no | Visual style for generated imagery. Full list: GET /options?kind=styles |
| `masterStyle` | string | no | Extra style directive applied to every generated scene, e.g. "Nature landscapes only: mountains, rivers, sky at sunrise" |
| `globalNegativePrompt` | string | no | Things never to show in any scene, e.g. "people, faces, hands, animals, birds, insects, buildings, text" |
| `script` | string | no | The words you say in the recording. Captions take their spelling from it (the timing comes from your audio) and the visuals follow it. Omit to caption and picture what the transcriber hears |
| `language` | string | no | Spoken language of the recording, e.g. English (the default). For English speech with Quran recitation or Arabic supplications, pass ar (Classic Arabic): that transcriber hears both and keeps the English words English. Full list: GET /options?kind=languages |
| `name` | string | no | Project name |
| `enableBackgroundMusic` | boolean | no | Mix background music under your voice. Omit to use the team's default (Team settings > Video defaults), else off |
| `enableSoundEffects` | boolean | no | Sound effects on transitions. false keeps only your voice and any music. Omit to use the team's default (Team settings > Video defaults) |
| `images` | array | no | Your own pictures shown at chosen moments, e.g. a client-review screenshot while you read it. Words in at/until must be in script when you pass one. Up to 20 |
| `captions` | object | no | This video's caption style, size and weight. Each part left out follows the team's default (Team settings > Video defaults > Captions, or faceless_update_video_defaults), else the product default |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/videos/audio" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"audioUrl":"https://drive.google.com/file/d/1AbCdEfGhIjKlMnOpQrStUvWxYz012345/view?usp=sharing","model":"storyboard","masterStyle":"Nature landscapes only: mountains, rivers, lakes, sky at sunrise","globalNegativePrompt":"people, faces, hands, animals, birds, insects, buildings, text","enableBackgroundMusic":false,"enableSoundEffects":false}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e803",
    "name": "Morning reminder",
    "model": "storyboard",
    "status": "processing",
    "creditsUsed": 20,
    "audioSeconds": 41.2,
    "statusUrl": "/api/v1/videos/665f1b2a9c31a2b3c4d5e803/status"
  }
}
```

### List the team's videos

`GET /videos`

Returns a paginated list of the team's video projects, newest first, with generation status and the rendered MP4 URL when available.

- Scopes: `videos:read`
- Credits: none
- CLI: `faceless videos list`
- MCP tool: `faceless_list_videos`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `page` | integer | no | Page number, starting at 1 |
| `limit` | integer | no | Items per page (max 100) |
| `archived` | boolean | no | Only archived videos when true |

Example:

```bash
curl -s "https://faceless.so/api/v1/videos" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "id": "665f1b2a9c31a2b3c4d5e801",
      "name": "Ocean facts",
      "platform": "tts",
      "model": "storyboard",
      "renderedVideoUrl": "https://exports.faceless.so/renders/665f1b2a9c31a2b3c4d5e801/1730000000000.mp4",
      "createdAt": "2026-07-30T10:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

### Get one video

`GET /videos/{id}`

Returns a single video project: script, voice, model, generation state, per-platform post metadata, the rendered MP4 URL once rendered, and when it posts: scheduledTime (the next time it goes out, or null when nothing is scheduled), scheduledPlatforms (the platforms still to post) and schedule (its posting-calendar entry, as GET /calendar returns it, with a status of scheduled, needs_approval, on_hold, posted, failed, skipped or cancelled, and each platform's own status, time and post link). Covers videos scheduled with POST /posts/schedule and series episodes waiting for their slot.

- Scopes: `videos:read`
- Credits: none
- CLI: `faceless videos get`
- MCP tool: `faceless_get_video`

Example:

```bash
curl -s "https://faceless.so/api/v1/videos/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e801",
    "name": "Ocean facts",
    "platform": "tts",
    "model": "storyboard",
    "script": "Did you know...",
    "voiceId": "EXAVITQu4vr4xnSDxMaL",
    "status": "completed",
    "scheduledTime": "2026-08-01T18:00:00.000Z",
    "scheduledPlatforms": [
      "youtube",
      "tiktok"
    ],
    "schedule": {
      "id": "project:665f1b2a9c31a2b3c4d5e801",
      "videoId": "665f1b2a9c31a2b3c4d5e801",
      "seriesId": null,
      "seriesName": null,
      "title": "The ocean has rivers underwater",
      "topic": null,
      "status": "scheduled",
      "stage": "scheduled",
      "scheduledTime": "2026-08-01T18:00:00.000Z",
      "day": "2026-08-01",
      "timezone": "UTC",
      "platforms": [
        "youtube",
        "tiktok"
      ],
      "platformPosts": [
        {
          "platform": "youtube",
          "status": "scheduled",
          "scheduledTime": "2026-08-01T18:00:00.000Z",
          "postedAt": null,
          "postUrl": null
        },
        {
          "platform": "tiktok",
          "status": "scheduled",
          "scheduledTime": "2026-08-01T18:00:00.000Z",
          "postedAt": null,
          "postUrl": null
        }
      ],
      "seriesPaused": false,
      "views": null,
      "error": null,
      "thumbnailUrl": "https://exports.faceless.so/thumbnails/665f1b2a9c31a2b3c4d5e801/1/a1b2c3d4.jpg"
    },
    "renderedVideoUrl": "https://exports.faceless.so/renders/665f1b2a9c31a2b3c4d5e801/1722333444555.mp4",
    "thumbnailUrl": "https://exports.faceless.so/thumbnails/665f1b2a9c31a2b3c4d5e801/1/a1b2c3d4.jpg",
    "thumbnails": {
      "status": "ready",
      "generation": 1,
      "aspect": "9:16",
      "selectedVariantId": "a1b2c3d4-0000-4000-8000-000000000001",
      "selectedBy": "ai",
      "variants": [
        {
          "id": "a1b2c3d4-0000-4000-8000-000000000001",
          "url": "https://exports.faceless.so/thumbnails/665f1b2a9c31a2b3c4d5e801/1/a1b2c3d4.jpg",
          "hookText": "OCEAN RIVERS",
          "source": "ai",
          "score": 88,
          "grade": "S"
        }
      ]
    },
    "createdAt": "2026-07-30T10:00:00.000Z"
  }
}
```

### Update a video's name or post metadata

`PATCH /videos/{id}`

Updates a video's name and/or its per-platform post metadata (titles, captions, descriptions). Set the metadata for every platform you intend to schedule to BEFORE calling POST /posts/schedule; scheduling rejects platforms whose metadata is missing.

- Scopes: `videos:write`
- Credits: none
- CLI: `faceless videos update`
- MCP tool: `faceless_update_video`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `name` | string | no | Project name |
| `youtubePost` | object | no | YouTube post metadata; required before scheduling to YouTube |
| `tiktokPost` | object | no | TikTok post metadata; required before scheduling to TikTok |
| `instagramPost` | object | no | Instagram post metadata; required before scheduling to Instagram |
| `xPost` | object | no | X post metadata; required before scheduling to X |
| `facebookPost` | object | no | Facebook post metadata; required before scheduling to Facebook |
| `linkedinPost` | object | no | LinkedIn post metadata; required before scheduling to LinkedIn |
| `threadsPost` | object | no | Threads post metadata; required before scheduling to Threads |

Example:

```bash
curl -s -X PATCH "https://faceless.so/api/v1/videos/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"youtubePost":{"title":"The ocean has rivers underwater","privacyStatus":"public"},"tiktokPost":{"title":"underwater rivers are real #ocean"}}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e801",
    "name": "Ocean facts",
    "youtubePost": {
      "title": "The ocean has rivers underwater"
    },
    "thumbnailUrl": "https://exports.faceless.so/thumbnails/665f1b2a9c31a2b3c4d5e801/1/a1b2c3d4.jpg",
    "thumbnails": {
      "status": "ready",
      "selectedBy": "ai",
      "variants": [
        "https://exports.faceless.so/thumbnails/665f1b2a9c31a2b3c4d5e801/a.jpg"
      ]
    }
  }
}
```

### Choose which thumbnail a video uses

`POST /videos/{id}/thumbnail`

Selects one of the video's thumbnail variants as the effective thumbnail. Read the available variants from GET /videos/{id} (the thumbnails object). The choice is marked as a user pick, so later automatic regenerations will not overwrite it, and it is what gets pushed to YouTube and used as the Instagram Reels cover at post time. Thumbnails are free and generated automatically after a video completes.

- Scopes: `videos:write`
- Credits: none
- CLI: `faceless videos select-thumbnail`
- MCP tool: `faceless_select_video_thumbnail`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `variantId` | string | yes | Variant id from the thumbnails.variants array on GET /videos/{id} |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/videos/<id>/thumbnail" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"variantId":"a1b2c3d4-0000-4000-8000-000000000002"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e801",
    "thumbnailUrl": "https://exports.faceless.so/thumbnails/665f1b2a9c31a2b3c4d5e801/1/b2c3d4e5.jpg",
    "thumbnails": {
      "status": "ready",
      "selectedVariantId": "a1b2c3d4-0000-4000-8000-000000000002",
      "selectedBy": "user",
      "variants": [
        "https://exports.faceless.so/thumbnails/665f1b2a9c31a2b3c4d5e801/a.jpg"
      ]
    }
  }
}
```

### Poll video generation progress

`GET /videos/{id}/status`

Returns generation progress for a video: overall status (pending, processing, completed, failed), percent complete, per-step events and any error messages. Poll every 10-30 seconds after POST /videos until status is completed, then render it with POST /videos/{id}/render. A failed generation refunds nothing automatically; check errorMessages.

- Scopes: `videos:read`
- Credits: none
- Terminal states: `completed`, `failed`
- CLI: `faceless videos status`
- MCP tool: `faceless_get_video_status`

Example:

```bash
curl -s "https://faceless.so/api/v1/videos/<id>/status" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e801",
    "status": "processing",
    "percentCompleted": 62,
    "readyForEditing": false,
    "errorMessages": [
      "b-roll fetch timed out, retried"
    ],
    "renderedVideoUrl": "https://exports.faceless.so/renders/665f1b2a9c31a2b3c4d5e801/1730000000000.mp4"
  }
}
```

### Render a video to MP4

`POST /videos/{id}/render`

Starts a cloud render of a generated video and returns a renderId. Rendering is asynchronous: poll GET /renders/{renderId} until status is done to get the MP4 URL. The video must have finished generating first (getVideoStatus status completed). Rendering is free; credits were charged at creation.

- Scopes: `videos:write`
- Credits: none
- Rate limit: 10 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless videos render`
- MCP tool: `faceless_render_video`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `codec` | `h264` \| `vp8` | no | Output codec; h264 for MP4 (default) |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/videos/<id>/render" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"codec":"h264"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "renderId": "abcd1234efgh",
    "statusUrl": "/api/v1/renders/abcd1234efgh"
  }
}
```

### Place your own pictures in a video

`POST /videos/{id}/images`

Shows your own pictures (screenshots, photos) in a video that has finished generating: each at a time or at words said in it, as an overlay card over the scene (whole picture visible, clear of the captions) or full screen. Images are ids from faceless_upload_image / faceless_list_assets or public links. Captions stay on top. blur (light or strong, per picture or for the whole call) blurs a picture on screen, e.g. a screenshot with names or private details, with no editing beforehand; it can be changed later in the editor (Blur in the picture's settings). replace: true first removes pictures placed earlier (to move or change them). Free: no credits and no AI generation. Then POST /videos/{id}/render for the MP4 and schedule as usual. Pictures whose words are not found are skipped and listed with the reason.

- Scopes: `videos:write`
- Credits: none
- Rate limit: 20 per 60s
- CLI: `faceless videos place-images`
- MCP tool: `faceless_place_images`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `images` | array | yes | The pictures and when and how each shows. Up to 20 |
| `replace` | boolean | no | Remove the pictures placed earlier by this tool before placing these (default false: add to them) |
| `blur` | `none` \| `light` \| `strong` | no | Blur for every picture in this call that does not set its own. none (clear), light (softened: a stylistic blur; names, headings and most text can still be read) or strong (text, names and faces cannot be read; only colours and rough shapes remain). Use strong to hide names, faces or private details in a screenshot without editing it first. Left out: the team's default (faceless_update_video_defaults imageBlur), else none |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/videos/<id>/images" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"images":[{"image":"665f1b2a9c31a2b3c4d5ef02","at":"Sarah wrote to us","until":"best decision I made","mode":"overlay","blur":"light"},{"image":"665f1b2a9c31a2b3c4d5ef03","at":21.5,"duration":3,"mode":"fullscreen"}]}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e803",
    "placed": [
      {
        "picture": 1,
        "image": "665f1b2a9c31a2b3c4d5ef02",
        "mode": "overlay",
        "fit": "contain",
        "blur": "light",
        "startSeconds": 8.42,
        "endSeconds": 14.1
      },
      {
        "picture": 2,
        "image": "665f1b2a9c31a2b3c4d5ef03",
        "mode": "fullscreen",
        "fit": "contain",
        "blur": "none",
        "startSeconds": 21.5,
        "endSeconds": 24.5
      }
    ],
    "skipped": [],
    "removed": 0,
    "ownImagesInVideo": 2,
    "renderNeeded": true,
    "message": "Placed. Call POST /videos/{id}/render (faceless_render_video) for an MP4 with the pictures in it."
  }
}
```

### Delete a video

`DELETE /videos/{id}`

Permanently deletes a video project and its generated assets. This cannot be undone and does not refund credits.

- Scopes: `videos:write`
- Credits: none
- CLI: `faceless videos delete`
- MCP tool: `faceless_delete_video`

Example:

```bash
curl -s -X DELETE "https://faceless.so/api/v1/videos/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

## renders

### Poll render progress

`GET /renders/{id}`

Returns the progress of a render started by POST /videos/{id}/render: status in-progress (with overallProgress 0-1), done (with the final MP4 url) or error. Poll every 5-15 seconds; renders typically take under two minutes.

- Scopes: `videos:read`
- Credits: none
- Terminal states: `done`, `error`
- CLI: `faceless renders get`
- MCP tool: `faceless_get_render`

Example:

```bash
curl -s "https://faceless.so/api/v1/renders/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "renderId": "abcd1234efgh",
    "status": "done",
    "overallProgress": 1,
    "url": "https://exports.faceless.so/renders/665f1b2a9c31a2b3c4d5e801/1722333444555.mp4"
  }
}
```

## series

### Create an automated video series

`POST /series`

Creates a series that automatically generates and posts a new faceless video on a schedule (a set-and-forget channel). Choose a content source and niche or custom prompt, a voice, style and posting cadence; each generated episode charges credits like a single video. The first episode starts generating right after creation. Use GET /series/{id}/episodes to track output, POST /series/{id}/generate to force the next episode now.

- Scopes: `series:write`
- Credits: none at creation; each generated episode charges its model's video cost (20 to 100 credits)
- Rate limit: 5 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless series create`
- MCP tool: `faceless_create_series`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `name` | string | yes | Series name shown in the dashboard |
| `source` | string | yes | Content source that scripts every episode, e.g. "Facts & stories", "Reddit post", "Custom prompt". Full list: GET /options?kind=sources |
| `niche` | string | no | Content niche, e.g. scary stories, motivation. Full list: GET /options?kind=niches |
| `customPrompt` | string | no | Custom topic prompt used instead of (or alongside) a niche |
| `voice` | string | no | TTS voice id for narration. Full list: GET /voices. Not used where the video model voices the characters so the lips match: Character Vlog, Time-Traveller POV and Body Reacts (unless set to the narrator in the app), Fruit Drama, Micro Drama, Street Interview and Talent Show |
| `style` | string | no | Visual style for generated imagery. Full list: GET /options?kind=styles |
| `language` | string | no | Video language, e.g. English. Omitted on create, it is the voice's own language when the voice is made for one language (a Spanish voice gives Spanish), else English. Full list: GET /options?kind=languages |
| `duration` | `30` \| `60` \| `90` | no | Target episode length in seconds (30, 60 or 90) |
| `destination` | string | no | Primary auto-post destination platform, e.g. youtube or tiktok |
| `destinationAccounts` | mixed | no | Which connected account each platform posts through, by platform, e.g. {"youtube":["<authId>"],"tiktok":["<authId>"]} with authIds from GET /accounts. Every id must be an account of this team on that platform, or the request is rejected. A platform with no entry posts through the team's default account. On update only the platforms you send change; null clears every pin |
| `generateThumbnails` | boolean | no | Generate 3 AI thumbnail variants for every episode and auto-select the best. Free. Defaults to true. |
| `thumbnailSettings` | object | no | Thumbnail generation options. Ignored when generateThumbnails is false. |
| `autoPostTime` | string | no | Daily auto-post time "HH:mm" in the series timezone |
| `postingDays` | array | no | Days of the week to post as numbers, 0 is Sunday and 6 is Saturday, e.g. [1,3] for Monday and Wednesday. Omit for every day |
| `timezone` | string | no | IANA timezone for scheduling, e.g. America/New_York |
| `platformPostTimes` | object | no | Post one platform at a different time of day, in the series timezone and on the same posting days, e.g. {"youtube":"09:05"}. Useful because YouTube's shared daily upload limit resets at midnight US Pacific. null clears it |
| `captionStyle` | string | no | Caption theme name. Full list: GET /options?kind=captionThemes |
| `captionPosition` | `middle` \| `lower` | no | Where the captions sit on every new episode of a "Facts & stories" series: "middle" or "lower" (the lower third). null or unset keeps the caption theme's own position |
| `subreddit` | string | no | Subreddit to pull posts from when source is "Reddit post" |
| `backgroundVideo` | string | no | Background gameplay/footage id. Full list: GET /options?kind=backgrounds |
| `useRandomBackgroundVideo` | boolean | no | Pick a random background video per episode |
| `layout` | string | no | Video layout variant |
| `brollModel` | string | no | Generation model for visuals: storyboard, motion_lite or motion_pro |
| `showEmojis` | boolean | no | Overlay emojis on captions |
| `enableBackgroundMusic` | boolean | no | Mix background music under the narration |
| `enableVoiceover` | boolean | no | Story Slides and News Slides only: an AI voice reads each slide, using `voice` (a default voice when omitted). Defaults to false (music only) |
| `slidesLength` | `short` \| `medium` \| `long` | no | Story Slides and News Slides only: episode length. "short" (about 20s, 5-6 slides, the default), "medium" (about 40s, about 12 slides) or "long" (about 60s, about 18 slides). Each slide stays on screen just as long, so longer means more slides. Priced per slide: medium costs 2x and long 3x the short price |
| `backgroundMusicMood` | string | no | Background music mood. Full list: GET /options?kind=music |
| `hashtags` | string | no | Hashtags appended to post captions |
| `tone` | string | no | Writing tone for generated scripts |
| `scriptureTranslation` | `kjv` \| `web` \| `drb` | no | Bible translation scripture is quoted from word for word: kjv, web or drb. null follows the brief, else kjv |
| `youtubePrivacyStatus` | `public` \| `unlisted` \| `private` | no | Privacy for auto-posted YouTube videos |
| `youtubeMadeForKids` | boolean | no | Declare auto-posted YouTube videos as made for kids (COPPA). Omitted: on for kids formats (Nursery Rhymes, bedtime-stories, good-morals, kids language lessons), not sent otherwise |
| `tiktokPrivacyLevel` | `PUBLIC_TO_EVERYONE` \| `MUTUAL_FOLLOW_FRIENDS` \| `SELF_ONLY` | no | Who can see auto-posted TikTok posts: PUBLIC_TO_EVERYONE (the default), MUTUAL_FOLLOW_FRIENDS or SELF_ONLY. A private TikTok account posts to its followers when PUBLIC_TO_EVERYONE is asked |
| `tiktokAllowComments` | boolean | no | Allow comments on auto-posted TikTok posts; defaults to true. An account that has comments disabled in TikTok posts with them off |
| `tiktokAllowDuet` | boolean | no | Allow other accounts to Duet auto-posted TikTok posts; defaults to true. An account that has Duet disabled in TikTok posts with it off |
| `tiktokAllowStitch` | boolean | no | Allow other accounts to Stitch auto-posted TikTok posts; defaults to true. An account that has Stitch disabled in TikTok posts with it off |
| `episodeTitleMode` | `unique` \| `series_numbered` | no | How episodes are titled when they post. "unique" (the default): every episode posts its own title. "series_numbered": every episode posts as "<seriesDisplayTitle> - Episode N" on YouTube, and every other platform's caption opens with that line above the episode's usual caption. N is the episode's place in a serial (Micro Drama) or the next number of the series, and never changes once an episode is made |
| `seriesDisplayTitle` | string | no | The series title numbered episodes carry. Empty uses a serial's own story title, then the series name |
| `episodeTitleSeparator` | `hyphen` \| `em_dash` \| `pipe` \| `colon` | no | What joins the series title and the episode number: hyphen ("Title - Episode 1", the default), em_dash, pipe ("Title | Episode 1") or colon ("Title: Episode 1") |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/series" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"Deep sea facts","source":"Facts & stories","niche":"Ocean facts","voice":"EXAVITQu4vr4xnSDxMaL","duration":"60","autoPostTime":"18:00","timezone":"America/New_York"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e901",
    "name": "Deep sea facts",
    "source": "Facts & stories",
    "niche": "Ocean facts",
    "createdAt": "2026-07-30T10:00:00.000Z"
  }
}
```

### List the team's series

`GET /series`

Returns all automated series for the team with their source, niche, cadence and pause state.

- Scopes: `series:read`
- Credits: none
- CLI: `faceless series list`
- MCP tool: `faceless_list_series`

Example:

```bash
curl -s "https://faceless.so/api/v1/series" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "id": "665f1b2a9c31a2b3c4d5e901",
      "name": "Deep sea facts",
      "source": "Facts & stories",
      "niche": "Ocean facts",
      "paused": false
    }
  ]
}
```

### Get one series

`GET /series/{id}`

Returns a single series with its full configuration: source, niche, voice, style, schedule, destinations and pause state.

- Scopes: `series:read`
- Credits: none
- CLI: `faceless series get`
- MCP tool: `faceless_get_series`

Example:

```bash
curl -s "https://faceless.so/api/v1/series/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e901",
    "name": "Deep sea facts",
    "source": "Facts & stories",
    "niche": "Ocean facts",
    "voice": "EXAVITQu4vr4xnSDxMaL",
    "destination": [
      "youtube"
    ],
    "destinationAccounts": {
      "youtube": [
        "665f1b2a9c31a2b3c4d5ee01"
      ]
    },
    "autoPostTime": "18:00",
    "paused": false
  }
}
```

### Update a series

`PATCH /series/{id}`

Updates any subset of a series' configuration (niche, voice, schedule, destinations, pause state and so on). Only the fields you send change; in destinationAccounts only the platforms you send change.

- Scopes: `series:write`
- Credits: none
- CLI: `faceless series update`
- MCP tool: `faceless_update_series`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `name` | string | no | Series name shown in the dashboard |
| `source` | string | no | Content source that scripts every episode, e.g. "Facts & stories", "Reddit post", "Custom prompt". Full list: GET /options?kind=sources |
| `niche` | string | no | Content niche, e.g. scary stories, motivation. Full list: GET /options?kind=niches |
| `customPrompt` | string | no | Custom topic prompt used instead of (or alongside) a niche |
| `voice` | string | no | TTS voice id for narration. Full list: GET /voices. Not used where the video model voices the characters so the lips match: Character Vlog, Time-Traveller POV and Body Reacts (unless set to the narrator in the app), Fruit Drama, Micro Drama, Street Interview and Talent Show |
| `style` | string | no | Visual style for generated imagery. Full list: GET /options?kind=styles |
| `language` | string | no | Video language, e.g. English. Omitted on create, it is the voice's own language when the voice is made for one language (a Spanish voice gives Spanish), else English. Full list: GET /options?kind=languages |
| `duration` | `30` \| `60` \| `90` | no | Target episode length in seconds (30, 60 or 90) |
| `destination` | string | no | Primary auto-post destination platform, e.g. youtube or tiktok |
| `destinationAccounts` | mixed | no | Which connected account each platform posts through, by platform, e.g. {"youtube":["<authId>"],"tiktok":["<authId>"]} with authIds from GET /accounts. Every id must be an account of this team on that platform, or the request is rejected. A platform with no entry posts through the team's default account. On update only the platforms you send change; null clears every pin |
| `generateThumbnails` | boolean | no | Generate 3 AI thumbnail variants for every episode and auto-select the best. Free. Defaults to true. |
| `thumbnailSettings` | object | no | Thumbnail generation options. Ignored when generateThumbnails is false. |
| `autoPostTime` | string | no | Daily auto-post time "HH:mm" in the series timezone |
| `postingDays` | array | no | Days of the week to post as numbers, 0 is Sunday and 6 is Saturday, e.g. [1,3] for Monday and Wednesday. Omit for every day |
| `timezone` | string | no | IANA timezone for scheduling, e.g. America/New_York |
| `platformPostTimes` | object | no | Post one platform at a different time of day, in the series timezone and on the same posting days, e.g. {"youtube":"09:05"}. Useful because YouTube's shared daily upload limit resets at midnight US Pacific. null clears it |
| `captionStyle` | string | no | Caption theme name. Full list: GET /options?kind=captionThemes |
| `captionPosition` | `middle` \| `lower` | no | Where the captions sit on every new episode of a "Facts & stories" series: "middle" or "lower" (the lower third). null or unset keeps the caption theme's own position |
| `subreddit` | string | no | Subreddit to pull posts from when source is "Reddit post" |
| `backgroundVideo` | string | no | Background gameplay/footage id. Full list: GET /options?kind=backgrounds |
| `useRandomBackgroundVideo` | boolean | no | Pick a random background video per episode |
| `layout` | string | no | Video layout variant |
| `brollModel` | string | no | Generation model for visuals: storyboard, motion_lite or motion_pro |
| `showEmojis` | boolean | no | Overlay emojis on captions |
| `enableBackgroundMusic` | boolean | no | Mix background music under the narration |
| `enableVoiceover` | boolean | no | Story Slides and News Slides only: an AI voice reads each slide, using `voice` (a default voice when omitted). Defaults to false (music only) |
| `slidesLength` | `short` \| `medium` \| `long` | no | Story Slides and News Slides only: episode length. "short" (about 20s, 5-6 slides, the default), "medium" (about 40s, about 12 slides) or "long" (about 60s, about 18 slides). Each slide stays on screen just as long, so longer means more slides. Priced per slide: medium costs 2x and long 3x the short price |
| `backgroundMusicMood` | string | no | Background music mood. Full list: GET /options?kind=music |
| `hashtags` | string | no | Hashtags appended to post captions |
| `tone` | string | no | Writing tone for generated scripts |
| `scriptureTranslation` | `kjv` \| `web` \| `drb` | no | Bible translation scripture is quoted from word for word: kjv, web or drb. null follows the brief, else kjv |
| `youtubePrivacyStatus` | `public` \| `unlisted` \| `private` | no | Privacy for auto-posted YouTube videos |
| `youtubeMadeForKids` | boolean | no | Declare auto-posted YouTube videos as made for kids (COPPA). Omitted: on for kids formats (Nursery Rhymes, bedtime-stories, good-morals, kids language lessons), not sent otherwise |
| `tiktokPrivacyLevel` | `PUBLIC_TO_EVERYONE` \| `MUTUAL_FOLLOW_FRIENDS` \| `SELF_ONLY` | no | Who can see auto-posted TikTok posts: PUBLIC_TO_EVERYONE (the default), MUTUAL_FOLLOW_FRIENDS or SELF_ONLY. A private TikTok account posts to its followers when PUBLIC_TO_EVERYONE is asked |
| `tiktokAllowComments` | boolean | no | Allow comments on auto-posted TikTok posts; defaults to true. An account that has comments disabled in TikTok posts with them off |
| `tiktokAllowDuet` | boolean | no | Allow other accounts to Duet auto-posted TikTok posts; defaults to true. An account that has Duet disabled in TikTok posts with it off |
| `tiktokAllowStitch` | boolean | no | Allow other accounts to Stitch auto-posted TikTok posts; defaults to true. An account that has Stitch disabled in TikTok posts with it off |
| `episodeTitleMode` | `unique` \| `series_numbered` | no | How episodes are titled when they post. "unique" (the default): every episode posts its own title. "series_numbered": every episode posts as "<seriesDisplayTitle> - Episode N" on YouTube, and every other platform's caption opens with that line above the episode's usual caption. N is the episode's place in a serial (Micro Drama) or the next number of the series, and never changes once an episode is made |
| `seriesDisplayTitle` | string | no | The series title numbered episodes carry. Empty uses a serial's own story title, then the series name |
| `episodeTitleSeparator` | `hyphen` \| `em_dash` \| `pipe` \| `colon` | no | What joins the series title and the episode number: hyphen ("Title - Episode 1", the default), em_dash, pipe ("Title | Episode 1") or colon ("Title: Episode 1") |
| `paused` | boolean | no | Pause or resume automatic episode generation |
| `seasonEndsOn` | string | no | Season end: the last day (YYYY-MM-DD, in the series timezone) a new episode is scheduled for. No episode is made or charged for a later slot, episodes already made still post, and after the last one posts the series pauses itself with seasonFinished true. null removes the season end |
| `seasonFinale` | boolean | no | Whether the last episode before the season end is written as a season finale that resolves the story (default true). false keeps its usual ending (a serial keeps its cliffhanger, for a story that continues elsewhere); scheduling still stops after seasonEndsOn. Needs a season end (sent with seasonEndsOn or already set) |

Example:

```bash
curl -s -X PATCH "https://faceless.so/api/v1/series/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"autoPostTime":"09:00","paused":false,"destinationAccounts":{"youtube":["665f1b2a9c31a2b3c4d5ee01"]}}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5e901",
    "name": "Deep sea facts",
    "autoPostTime": "09:00",
    "paused": false
  }
}
```

### Delete a series

`DELETE /series/{id}`

Deletes a series and stops all future automatic generation. Already-generated videos are kept.

- Scopes: `series:write`
- Credits: none
- CLI: `faceless series delete`
- MCP tool: `faceless_delete_series`

Example:

```bash
curl -s -X DELETE "https://faceless.so/api/v1/series/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

### Generate the next episode now

`POST /series/{id}/generate`

Immediately generates the next episode of a series instead of waiting for its schedule. Charges the series' per-episode credit cost. The episode appears in GET /series/{id}/episodes; poll the created video's status like any other video.

- Scopes: `series:write`
- Credits: the series' per-episode video cost (20 to 100 credits depending on model)
- Rate limit: 5 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless series generate`
- MCP tool: `faceless_generate_series_episode`

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/series/<id>/generate" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "projectId": "665f1b2a9c31a2b3c4d5ea01",
    "statusUrl": "/api/v1/videos/665f1b2a9c31a2b3c4d5ea01/status"
  }
}
```

### List a series' episodes

`GET /series/{id}/episodes`

Returns the videos generated by a series, newest first, with their generation status and when they post: scheduledTime (the next time the episode goes out, or null), scheduledPlatforms, postedAt, and schedule (its posting-calendar entry, as GET /calendar returns it, with a status of scheduled, needs_approval, on_hold, posted, failed, skipped or cancelled). Slots the series has not made yet are on GET /calendar.

- Scopes: `series:read`
- Credits: none
- CLI: `faceless series episodes`
- MCP tool: `faceless_list_series_episodes`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `page` | integer | no | Page number, starting at 1 |
| `limit` | integer | no | Items per page (max 100) |

Example:

```bash
curl -s "https://faceless.so/api/v1/series/<id>/episodes" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "id": "665f1b2a9c31a2b3c4d5eb01",
      "projectId": "665f1b2a9c31a2b3c4d5ea01",
      "seriesId": "665f1b2a9c31a2b3c4d5eb09",
      "name": "Why the Dead Sea is sinking",
      "status": "completed",
      "scheduledTime": "2026-07-30T18:00:00.000Z",
      "scheduledPlatforms": [
        "youtube"
      ],
      "schedule": {
        "id": "project:665f1b2a9c31a2b3c4d5ea01",
        "videoId": "665f1b2a9c31a2b3c4d5ea01",
        "seriesId": "665f1b2a9c31a2b3c4d5eb09",
        "seriesName": "Ocean facts daily",
        "title": "Why the Dead Sea is sinking",
        "topic": "Why the Dead Sea is sinking",
        "status": "scheduled",
        "stage": "scheduled",
        "scheduledTime": "2026-07-30T18:00:00.000Z",
        "day": "2026-07-30",
        "timezone": "UTC",
        "platforms": [
          "youtube"
        ],
        "platformPosts": [
          {
            "platform": "youtube",
            "status": "scheduled",
            "scheduledTime": "2026-07-30T18:00:00.000Z",
            "postedAt": null,
            "postUrl": null
          }
        ],
        "seriesPaused": false,
        "views": null,
        "error": null,
        "thumbnailUrl": null
      },
      "postedAt": null,
      "renderedVideoUrl": "https://exports.faceless.so/renders/665f1b2a9c31a2b3c4d5ea01/1722333444555.mp4",
      "createdAt": "2026-07-30T06:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

## posts

### Publish a rendered video to a platform now

`POST /posts`

Immediately publishes a rendered video to one connected platform (youtube, tiktok, instagram, x, facebook, linkedin or threads). The video must be rendered first (renderedVideoUrl present, or pass videoUrl explicitly) and the team must have that platform connected (GET /accounts). Every connected account gets a video once: publishing it again to an account that already has it answers 409 conflict unless allowRepost is true. For future-dated posting use POST /posts/schedule instead.

- Scopes: `posts:write`
- Credits: none
- Rate limit: 10 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless posts publish`
- MCP tool: `faceless_publish_post`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `videoId` | string | yes | The video (project) to publish |
| `platform` | `youtube` \| `tiktok` \| `instagram` \| `x` \| `facebook` \| `linkedin` \| `threads` | yes | Destination platform; must be connected (see GET /accounts) |
| `title` | string | no | Post title or caption (platform-appropriate); falls back to the video's stored post metadata |
| `description` | string | no | Longer description for platforms that support one (YouTube, Facebook, LinkedIn) |
| `privacyStatus` | `public` \| `unlisted` \| `private` | no | YouTube only; defaults to public |
| `madeForKids` | boolean | no | YouTube only: declare the video made for kids (COPPA). Omitted: the video's stored choice, else on for kids formats, else not sent |
| `tiktokPrivacyLevel` | `PUBLIC_TO_EVERYONE` \| `MUTUAL_FOLLOW_FRIENDS` \| `SELF_ONLY` | no | TikTok only: who can see the post; defaults to PUBLIC_TO_EVERYONE (followers, on a private TikTok account) |
| `tiktokAllowComments` | boolean | no | TikTok only: allow comments on the post; defaults to true. An account that has comments disabled in TikTok posts with them off |
| `authId` | string | no | Specific connected account id when the team has several for the platform (from GET /accounts) |
| `allowRepost` | boolean | no | Post it again even though this video is already on that channel or account. Without it, a repeat is refused with 409 conflict and nothing is posted |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/posts" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"videoId":"665f1b2a9c31a2b3c4d5e801","platform":"youtube","title":"The ocean has rivers underwater"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "platform": "youtube",
    "videoId": "665f1b2a9c31a2b3c4d5e801",
    "postUrl": "https://youtube.com/shorts/dQw4w9WgXcQ"
  }
}
```

### Schedule a video to one or more platforms

`POST /posts/schedule`

Schedules a video to be rendered (if needed) and posted to multiple platforms at a future time. Every platform in the list must have its post metadata already set on the video (PATCH /videos/{id}); the request is rejected otherwise. Cancel with DELETE /posts/{videoId}. See the posting calendar with GET /calendar.

- Scopes: `posts:write`
- Credits: none
- Rate limit: 20 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless posts schedule`
- MCP tool: `faceless_schedule_post`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `videoId` | string | yes | The video (project) to schedule |
| `platforms` | array | yes | Platforms to post to; each needs its post metadata set on the video first |
| `scheduledTime` | string | yes | When to post (ISO 8601, future) |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/posts/schedule" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"videoId":"665f1b2a9c31a2b3c4d5e801","platforms":["youtube","tiktok"],"scheduledTime":"2026-08-01T18:00:00Z"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "taskId": "665f1b2a9c31a2b3c4d5ec01",
    "platforms": [
      "youtube",
      "tiktok"
    ],
    "scheduledTime": "2026-08-01T18:00:00Z"
  }
}
```

### Post a scheduled video now instead of at its scheduled time

`POST /posts/post-now`

Brings a scheduled video forward to now: every pending posting task of the video (and its render, if that is what is still waiting) moves to the current time and goes out within a minute or two, to every platform it was scheduled for. Works for videos scheduled with POST /posts/schedule and for series episodes waiting for their slot. It does not post a video that was never scheduled (use POST /posts), a video that still needs approval (409; approve it first), or an episode of a deleted series or of one paused for any reason but credits (409; resume the series first). Returns 400 when nothing is pending.

- Scopes: `posts:write`
- Credits: none
- Rate limit: 20 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless posts now`
- MCP tool: `faceless_post_now`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `videoId` | string | yes | The scheduled video (project) to post now |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/posts/post-now" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"videoId":"665f1b2a9c31a2b3c4d5e801"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "videoId": "665f1b2a9c31a2b3c4d5e801",
    "taskIds": [
      "665f1b2a9c31a2b3c4d5ec01",
      "665f1b2a9c31a2b3c4d5ec02"
    ],
    "scheduledTime": "2026-10-03T21:04:11.000Z"
  }
}
```

### Cancel a scheduled post

`DELETE /posts/{id}`

Cancels the pending scheduled posting for a video (the id is the video id used in POST /posts/schedule). Posts that already went out are not affected.

- Scopes: `posts:write`
- Credits: none
- CLI: `faceless posts cancel`
- MCP tool: `faceless_cancel_post`

Example:

```bash
curl -s -X DELETE "https://faceless.so/api/v1/posts/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "cancelled": true
  }
}
```

### Posting calendar

`GET /calendar`

Returns the team's posting calendar for a date range, the same entries the Calendar page in the app shows: upcoming posts (videos scheduled with POST /posts/schedule, series episodes waiting for their slot, and series slots not made yet), posts that went out and posts that failed. One entry per video or series slot, with scheduledTime, the target platforms and a status: scheduled, needs_approval (waits for the creator's approval), on_hold (its series is paused or deleted, so it will not post until resumed), posted, failed, skipped or cancelled (its posting was stopped). platformPosts gives each platform's own status, time and post link. Without a range it covers today and the next 7 days. Use it to confirm what is queued before scheduling more.

- Scopes: `posts:read`
- Credits: none
- CLI: `faceless calendar`
- MCP tool: `faceless_get_calendar`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `startDate` | string | no | Range start (inclusive). A date (2026-08-01) means the start of that day in `timezone`. Defaults to today |
| `endDate` | string | no | Range end (inclusive). A date (2026-08-31) means the end of that day in `timezone`. Defaults to 7 days after startDate; at most 92 days |
| `timezone` | string | no | IANA time zone for date-only bounds and each entry's `day`, e.g. America/Los_Angeles. Defaults to UTC |
| `platform` | `youtube` \| `tiktok` \| `instagram` \| `x` \| `facebook` \| `linkedin` \| `threads` | no | Only entries that post to this platform |
| `status` | `scheduled` \| `needs_approval` \| `on_hold` \| `posted` \| `failed` \| `skipped` \| `cancelled` | no | Only entries with this status, on the entry or on one of its platforms: scheduled, needs_approval, on_hold, posted, failed, skipped, cancelled |

Example:

```bash
curl -s "https://faceless.so/api/v1/calendar" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "id": "project:665f1b2a9c31a2b3c4d5e801",
      "videoId": "665f1b2a9c31a2b3c4d5e801",
      "seriesId": null,
      "seriesName": null,
      "title": "The ocean has rivers underwater",
      "topic": null,
      "status": "scheduled",
      "stage": "scheduled",
      "scheduledTime": "2026-08-01T18:00:00.000Z",
      "day": "2026-08-01",
      "timezone": "America/Los_Angeles",
      "platforms": [
        "youtube",
        "tiktok"
      ],
      "platformPosts": [
        {
          "platform": "youtube",
          "status": "scheduled",
          "scheduledTime": "2026-08-01T18:00:00.000Z",
          "postedAt": null,
          "postUrl": null
        },
        {
          "platform": "tiktok",
          "status": "scheduled",
          "scheduledTime": "2026-08-01T18:00:00.000Z",
          "postedAt": null,
          "postUrl": null
        }
      ],
      "seriesPaused": false,
      "views": null,
      "error": null,
      "thumbnailUrl": "https://exports.faceless.so/thumbnails/665f1b2a9c31a2b3c4d5e801/1/a1b2c3d4.jpg"
    },
    {
      "id": "slot:665f1b2a9c31a2b3c4d5eb01:1785693600000",
      "videoId": null,
      "seriesId": "665f1b2a9c31a2b3c4d5eb01",
      "seriesName": "Ocean facts daily",
      "title": null,
      "topic": "Why the Dead Sea is sinking",
      "status": "scheduled",
      "stage": "planned",
      "scheduledTime": "2026-08-02T18:00:00.000Z",
      "day": "2026-08-02",
      "timezone": "America/Los_Angeles",
      "platforms": [
        "youtube"
      ],
      "platformPosts": [
        {
          "platform": "youtube",
          "status": "scheduled",
          "scheduledTime": "2026-08-02T18:00:00.000Z",
          "postedAt": null,
          "postUrl": null
        }
      ],
      "seriesPaused": false,
      "views": null,
      "error": null,
      "thumbnailUrl": null
    }
  ],
  "range": {
    "start": "2026-08-01T07:00:00.000Z",
    "end": "2026-08-09T06:59:59.999Z",
    "timezone": "America/Los_Angeles"
  }
}
```

## accounts

### List connected social accounts

`GET /accounts`

Returns the social accounts connected to the team (YouTube, TikTok, Instagram, X, Facebook, LinkedIn, Threads) with their ids. Use the ids as authId in publishPost and destinationAccounts in series. Connecting accounts itself happens in the dashboard (OAuth), not over the API.

- Scopes: `accounts:read`
- Credits: none
- CLI: `faceless accounts`
- MCP tool: `faceless_list_accounts`

Example:

```bash
curl -s "https://faceless.so/api/v1/accounts" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "youtube": [
      {
        "authId": "665f1b2a9c31a2b3c4d5ee01",
        "label": "Deep Sea Facts",
        "isDefault": true
      }
    ],
    "tiktok": [
      {
        "authId": "665f1b2a9c31a2b3c4d5e901",
        "label": "mychannel",
        "isDefault": true
      }
    ],
    "instagram": [
      {
        "authId": "665f1b2a9c31a2b3c4d5e902",
        "label": "mychannel",
        "isDefault": false
      }
    ]
  }
}
```

## catalog

### List TTS voices

`GET /voices`

Returns the text-to-speech voices available for narration, including the team's custom cloned voices and Recommended Studio Quality voices. Recommended ids are opaque: pass the returned id unchanged and Faceless will resolve the appropriate ElevenLabs model. Use a voice's id as voiceId in createVideo or voice in createSeries.

- Scopes: `catalog:read`
- Credits: none
- CLI: `faceless voices`
- MCP tool: `faceless_list_voices`

Example:

```bash
curl -s "https://faceless.so/api/v1/voices" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "id": "eleven_quality:EXAVITQu4vr4xnSDxMaL",
      "baseVoiceId": "EXAVITQu4vr4xnSDxMaL",
      "name": "Sarah - Studio Quality",
      "previewUrl": "https://storage.googleapis.com/eleven-public-prod/premade/voices/EXAVITQu4vr4xnSDxMaL.mp3",
      "labels": {
        "language": "en",
        "gender": "female",
        "accent": "american"
      },
      "isCustom": false,
      "isRecommended": true,
      "qualityTier": "studio",
      "modelPolicy": "quality_auto",
      "targetLanguages": [
        "en"
      ]
    }
  ]
}
```

### List content option catalogs

`GET /options`

Enumerates the option catalogs used by createVideo and createSeries: sources, styles, niches, languages, durations, models (with credit costs), captionThemes, music moods, background videos and example subreddits. Call without kind to list the available kinds; pass kind to get that catalog's items.

- Scopes: `catalog:read`
- Credits: none
- CLI: `faceless options`
- MCP tool: `faceless_list_options`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `kind` | `sources` \| `styles` \| `niches` \| `languages` \| `durations` \| `models` \| `captionThemes` \| `music` \| `backgrounds` \| `subreddits` | no | Which catalog to return; omit to list available kinds |

Example:

```bash
curl -s "https://faceless.so/api/v1/options" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"kind":"models"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "kind": "models",
    "items": [
      {
        "value": "storyboard",
        "name": "Faceless Storyboard",
        "credits": 20
      }
    ]
  }
}
```

## assets

### Register a media asset by URL

`POST /assets`

Registers an already-hosted media file (video, image or audio URL) as a team asset usable in projects. The link is recorded as it is, not copied; for a picture to show in a video, use POST /assets/images (faceless_upload_image), which copies and checks it.

- Scopes: `assets:write`
- Credits: none
- Supports `Idempotency-Key` header
- CLI: `faceless assets create`
- MCP tool: `faceless_create_asset`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | yes | Public URL of the media file |
| `name` | string | no | Display name for the asset |
| `fileType` | string | no | Kind (image, video, audio) or MIME type, e.g. video/mp4. Omit to infer it from the link |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/assets" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/broll.mp4","name":"Intro b-roll"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5ef01",
    "url": "https://example.com/broll.mp4",
    "name": "Intro b-roll"
  }
}
```

### Upload your own image

`POST /assets/images`

Adds a picture (a screenshot, photo or logo) to the team's media library so it can be shown in videos (faceless_create_video_from_audio images, faceless_place_images) and in the editor. Pass url (a Google Drive share link set to "Anyone with the link", a Dropbox link or a direct image URL; up to 15 MB) or data (base64 or a data:image/...;base64, URI; up to 2.5 MB). JPG, PNG, WebP, GIF (first frame) or AVIF; an iPhone HEIC must be exported as JPG first. The picture is checked, turned upright, scaled to at most 2160 px on its long side and hosted on faceless.so. Returns the image id. Free.

- Scopes: `assets:write`
- Credits: none
- Rate limit: 30 per 60s
- Supports `Idempotency-Key` header
- CLI: `faceless assets upload-image`
- MCP tool: `faceless_upload_image`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | no | Public link to the picture: a Google Drive share link ("Anyone with the link"), a Dropbox link or a direct image URL |
| `data` | string | no | The picture itself as base64, or a data:image/png;base64,... URI (up to 2.5 MB). Use url for anything larger |
| `name` | string | no | Display name in the media library, e.g. "Review from Sarah" |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/assets/images" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://drive.google.com/file/d/1AbCdEfGhIjKlMnOpQrStUvWxYz012345/view?usp=sharing","name":"Review from Sarah"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5ef02",
    "url": "https://exports.faceless.so/uploads/own-images/665f1b2a9c31a2b3c4d5e000/0d4c7a52-7f0e-4d3b-9a43-3c5c1d9b2e10.jpg",
    "name": "Review from Sarah",
    "fileType": "image",
    "createdAt": "2026-11-17T10:00:00.000Z",
    "width": 1170,
    "height": 2532,
    "bytes": 412345
  }
}
```

### List the team's media library

`GET /assets`

Lists the team's uploaded media, newest first: pictures added with faceless_upload_image and files uploaded in the Faceless editor. Use type=image to find picture ids for faceless_place_images or faceless_create_video_from_audio images.

- Scopes: `assets:read`
- Credits: none
- CLI: `faceless assets list`
- MCP tool: `faceless_list_assets`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `type` | `image` \| `video` \| `audio` | no | Only this kind of media |
| `page` | integer | no | Page number, starting at 1 |
| `limit` | integer | no | Items per page (max 100) |

Example:

```bash
curl -s "https://faceless.so/api/v1/assets" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"type":"image"}'
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "id": "665f1b2a9c31a2b3c4d5ef02",
      "url": "https://exports.faceless.so/uploads/own-images/665f1b2a9c31a2b3c4d5e000/0d4c7a52-7f0e-4d3b-9a43-3c5c1d9b2e10.jpg",
      "name": "Review from Sarah",
      "fileType": "image",
      "createdAt": "2026-11-17T10:00:00.000Z",
      "width": 1170,
      "height": 2532,
      "bytes": 412345
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

## analytics

### Cross-platform posting analytics

`GET /analytics`

Returns performance for the team's published posts over a trailing window: aggregated views, likes, comments and posting volume for TikTok, Instagram and Facebook, and top posts per platform. YouTube is reported per video only (videos published in the last 30 days): views, likes and comments from the YouTube Data API plus average view percentage, average view duration and engaged views from the YouTube Analytics API when available; youtubeVideoStats explains the sources and any missing metric.

- Scopes: `analytics:read`
- Credits: none
- CLI: `faceless analytics`
- MCP tool: `faceless_get_analytics`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `platform` | `youtube` \| `tiktok` \| `instagram` \| `facebook` | no | Filter to one platform |
| `authId` | string | no | Filter to one connected account |
| `range` | mixed | no | Trailing analytics window: 7, 28, or 90 days (default 28) |

Example:

```bash
curl -s "https://faceless.so/api/v1/analytics" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "rangeDays": 28,
    "rollup": {
      "rangeDays": 28,
      "viewsGained": 12500,
      "totalViews": 128530,
      "postsSynced": 42,
      "ready": true
    },
    "windows": {
      "d7": {
        "viewsGained": 3200,
        "ready": true
      },
      "d28": {
        "viewsGained": 12500,
        "ready": true
      }
    },
    "byPlatform": {
      "tiktok": {
        "totalViews": 88000,
        "postsSynced": 20,
        "viewsGained28d": 9400
      }
    },
    "topPosts": [
      {
        "platform": "youtube",
        "videoId": "dQw4w9WgXcQ",
        "title": "Deep sea rivers",
        "views": 42000,
        "likes": 1800,
        "comments": 64,
        "shares": null,
        "averageViewPercentage": 87.4,
        "averageViewDurationSeconds": 31,
        "engagedViews": 30500,
        "analyticsStatus": "ok",
        "postUrl": "https://youtube.com/shorts/dQw4w9WgXcQ"
      },
      {
        "platform": "tiktok",
        "title": "Deep sea rivers",
        "views": 12500,
        "likes": 900,
        "comments": 31,
        "shares": 12,
        "postUrl": "https://www.tiktok.com/@example/video/1"
      }
    ],
    "youtubeVideoStats": {
      "videosShown": 12,
      "videosInWindow": 30,
      "windowDays": 28,
      "analytics": {
        "available": true,
        "metrics": [
          "averageViewPercentage",
          "averageViewDuration",
          "engagedViews"
        ],
        "reason": null
      }
    },
    "totals": {
      "totalViews": 128530,
      "postsSynced": 42,
      "posts7d": 7
    },
    "timeseriesReady": true
  }
}
```

## fast-track

### List Fast Track video types, categories and scripts

`GET /fast-track`

Fast Track is the quickest way to a video: pick a video type, a category and a script. These are finished videos from the Fast Track library and one can be sold to up to 3 channels; each copy gets its own title, music, caption style and framing, and a narrated story a different voice. Call with no query for the video types (how many scripts each has and its price in credits), with source for that type's categories (niche, language, count), and with source, niche and language for up to 20 scripts (id, preview, coverUrl, seconds). Get one with POST /fast-track. Free.

- Scopes: `videos:read`
- Credits: none
- CLI: `faceless fast-track list`
- MCP tool: `faceless_list_fast_track`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `source` | string | no | A video type from the list without a query, e.g. Facts & stories |
| `niche` | string | no | A category of that type, e.g. scary-stories (needs source) |
| `language` | string | no | Language code of the scripts, e.g. en (default en) |

Example:

```bash
curl -s "https://faceless.so/api/v1/fast-track" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"source":"Facts & stories","niche":"scary-stories","language":"en"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "videos": [
      {
        "id": "665f1b2a9c31a2b3c4d5ef01",
        "preview": "The lighthouse keeper heard knocking at midnight. Nobody had climbed the stairs in years.",
        "coverUrl": "https://exports.faceless.so/thumbs/665f1b2a9c31a2b3c4d5ef01.jpg",
        "seconds": 52
      }
    ],
    "credits": 10
  }
}
```

### Get a Fast Track video

`POST /fast-track`

Gets the picked Fast Track script (an id from GET /fast-track) as a video in the team's library, made its own (title, music, caption style, framing; a narrated story a different voice), and charges its price at once: 10 credits, or 3 for Slideshows, Single Fade-in Video and 2x2 Grid Video. The title, captions, music and cover can be changed in the editor; the voice cannot. The video opens straight away; its file is rendered within minutes. 409 when the script is no longer available: list again and pick another.

- Scopes: `videos:write`
- Credits: 10 credits (3 for Slideshows, Single Fade-in Video and 2x2 Grid Video), charged at purchase
- Rate limit: 10 per 60s
- Supports `Idempotency-Key` header
- CLI: `faceless fast-track buy`
- MCP tool: `faceless_buy_fast_track_video`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `fastTrackVideoId` | string | yes | A script id from GET /fast-track |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/fast-track" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"fastTrackVideoId":"665f1b2a9c31a2b3c4d5ef01"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5ea01",
    "name": "The lighthouse keeper heard knocking at midnight.",
    "platform": "tts",
    "model": "storyboard",
    "status": "completed",
    "thumbnailUrl": "https://exports.faceless.so/thumbs/665f1b2a9c31a2b3c4d5ef01.jpg",
    "renderedVideoUrl": null,
    "createdAt": "2026-10-21T18:13:00.000Z",
    "creditsUsed": 10,
    "statusUrl": "/api/v1/videos/665f1b2a9c31a2b3c4d5ea01/status"
  }
}
```

## edits

### Edit your own video with the AI Video Editor

`POST /edits`

Edits a video you recorded yourself (a talking head, a podcast clip, a vlog, a product demo) into a finished video, the way a human editor would: it removes silences, umms and false starts by the words you say, keeps your face and voice in sync, frames your face (vertical or landscape) with punch-ins on the lines that land, puts a bold hook line on screen in the first 2 seconds, word-by-word captions in your caption style, AI scenery cutaways over your footage while your voice keeps playing (drawn under your team's picture rule and negative prompt; still pictures with a camera move unless motionCutaways is true), and your brand bar, intro and end card from your brief and brand kit. Pass a public link to the video: a Google Drive share link set to "Anyone with the link", a Dropbox link or a direct file URL (MP4 or MOV, up to 10 minutes, up to 3 GB). Say in brief what you want: your hook line, where cutaways go (by time in your video, "at 0:45", or by the words you say, "when I say peace"), how many, your business name, a call to action. Pictures to show (your logo, client review screenshots, blurred with your team's picture blur) go in imageUrls. The video is fetched in the background: poll GET /edits/{id} until status is ready (it returns the finished film's URL and the project id to open it in the editor) or failed. Credits are charged when the fetched video starts editing: per second of your video at 1 credit (1.5 on the newer pricing), plus 3 credits per AI cutaway (14 for moving ones); cutaways left to the editor are priced at about one per 30 seconds of video, 2 to 12. Nothing is charged if the link cannot be fetched.

- Scopes: `videos:write`
- Credits: 1 credit per second of your video (1.5 on the newer pricing) plus 3 per AI cutaway (14 moving), charged when the video starts editing; a change to a finished edit is 35% of that
- Rate limit: 6 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless edits create`
- MCP tool: `faceless_edit_video`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `videoUrl` | string | yes | Public link to your video: a Google Drive share link ("Anyone with the link"), a Dropbox link or a direct file URL. MP4 or MOV, up to 10 minutes |
| `videoUrls` | array | no | More videos to edit together (more takes, your own b-roll), in order. Up to 6 in all, 10 minutes in all |
| `brief` | string | yes | How to edit it, in your words: the hook line, where the cutaways go ("at 0:45", "when I say peace"), what they show, your business name for the brand bar, the call to action for the end card. The editor follows it; where you say nothing it chooses |
| `preset` | `talking-head` \| `podcast-clips` \| `vlog` \| `product-demo` \| `shorts-from-long` | no | What kind of footage it is: talking-head (default), podcast-clips (the best moment as a clip), vlog, product-demo, shorts-from-long (the best moment as a vertical Short) |
| `aspect` | `9:16` \| `16:9` \| `1:1` \| `4:5` \| `auto` | no | The finished video's shape. auto (default) keeps your video's own; 9:16 from landscape footage is cropped around your face |
| `cutaways` | integer | no | How many AI cutaways. 0 keeps your face on screen throughout; omit to let the editor choose (priced at about one per 30 s of video) |
| `motionCutaways` | boolean | no | Moving cutaways (short generated clips) instead of still pictures with a camera move. 14 credits each instead of 3 |
| `music` | boolean | no | A quiet music bed under your voice. Off by default |
| `captions` | string | no | A caption theme for this video (GET /options?kind=captionThemes). Omit to use the team's caption style (GET /video-defaults) |
| `imageUrls` | array | no | Pictures to use: your logo, client review screenshots to show as overlays. Public links, up to 15 MB each |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/edits" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"videoUrl":"https://drive.google.com/file/d/1AbCdEfGhIjKlMnOpQrStUvWxYz012345/view?usp=sharing","brief":"Edit my talking video for Reels. Hook line: 'Stress is not your enemy'. Cut the pauses and umms, zoom in when I make a point, put calm sea scenery when I say peace and at 1:10, keep my face at the start and the end, brand bar with Muslim Stress Management Mastery, end card: book a free call.","preset":"talking-head","aspect":"9:16","cutaways":4}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "6b0412a9c31a2b3c4d5e8f01",
    "status": "importing",
    "headline": "Fetching your video. Nothing is charged until it is in.",
    "error": null,
    "creditsCharged": 0,
    "filmUrl": null,
    "posterUrl": null,
    "projectId": null,
    "studioUrl": null,
    "statusUrl": "/api/v1/edits/6b0412a9c31a2b3c4d5e8f01"
  }
}
```

### Follow an AI Video Editor job

`GET /edits/{id}`

Returns where an edit is: importing (the video is being fetched; nothing charged yet), queued, editing (the editor is cutting it; headline says what it is doing), rendering, ready (filmUrl is the finished MP4, projectId opens it in the editor, studioUrl shows it in the studio) or failed (error says why; a failed edit is refunded). revising while a change you asked for is being made. Poll every 30-60 seconds: an edit takes about 20 to 60 minutes.

- Scopes: `videos:read`
- Credits: none
- Terminal states: `ready`, `failed`
- CLI: `faceless edits get`
- MCP tool: `faceless_get_edit`

Example:

```bash
curl -s "https://faceless.so/api/v1/edits/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "6b0412a9c31a2b3c4d5e8f01",
    "runId": "6b0412b0c31a2b3c4d5e8f09",
    "status": "ready",
    "headline": "Your edit is ready.",
    "error": null,
    "title": "Stress Is Not Your Enemy",
    "filmUrl": "https://exports.faceless.so/motion-director/6b0412b0c31a2b3c4d5e8f09/t1/k2j3-film.mp4",
    "posterUrl": "https://exports.faceless.so/motion-director/6b0412b0c31a2b3c4d5e8f09/t1/k2j3-poster.jpg",
    "seconds": 108.4,
    "version": 1,
    "projectId": "6b0412b0c31a2b3c4d5e8f0a",
    "creditsCharged": 132,
    "changeCredits": 46,
    "studioUrl": "https://faceless.so/motion-director/6b0412b0c31a2b3c4d5e8f09"
  }
}
```

### Ask for a change to an edit

`POST /edits/{id}/notes`

Sends a note to the editor. While the edit is being made it is free and the editor folds it in. Once the edit is ready it asks for a change (a new version of the same film), charged at changeCredits from GET /edits/{id} (35% of the edit's price). Then poll GET /edits/{id} again.

- Scopes: `videos:write`
- Credits: none while the edit is being made; a change to a finished edit is 35% of its price (changeCredits)
- Rate limit: 10 per 300s
- Supports `Idempotency-Key` header
- CLI: `faceless edits change`
- MCP tool: `faceless_change_edit`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `text` | string | yes | What to change, e.g. "make the hook line bigger and use a forest instead of the sea at 0:45" |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/edits/<id>/notes" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text":"Make the hook line bigger and use a forest instead of the sea at 0:45."}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "6b0412a9c31a2b3c4d5e8f01",
    "runId": "6b0412b0c31a2b3c4d5e8f09",
    "status": "revising",
    "headline": "Waiting for the studio.",
    "error": null,
    "revision": true,
    "creditsCharged": 46,
    "studioUrl": "https://faceless.so/motion-director/6b0412b0c31a2b3c4d5e8f09"
  }
}
```

## ads

### Search Meta ads shown in the EU and UK

`GET /ads/search`

Searches the official Meta Ad Library by keywords in the ad text or by one advertiser's exact Facebook Page ID. Meta only returns commercial ads delivered in the EU and UK, so the country is one of those or EU_UK for all of them. Returns up to 25 ads (copy variants, delivery dates, platforms, a public Meta link), a resultToken to save or brief any of them within 15 minutes, and an opaque nextCursor. Free; each team has a daily search allowance and cached repeats do not count against it. No media files are returned.

- Scopes: `ads:read`
- Credits: none
- Rate limit: 10 per 60s
- CLI: `faceless ads search`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `q` | string | no | Words in the ad text. Send q or pageId, not both. |
| `pageId` | string | no | One advertiser's exact Facebook Page ID (from its Ad Library link, view_all_page_id). |
| `country` | `EU_UK` \| `AT` \| `BE` \| `BG` \| `HR` \| `CY` \| `CZ` \| `DK` \| `EE` \| `FI` \| `FR` \| `DE` \| `GR` \| `HU` \| `IE` \| `IT` \| `LV` \| `LT` \| `LU` \| `MT` \| `NL` \| `PL` \| `PT` \| `RO` \| `SK` \| `SI` \| `ES` \| `SE` \| `GB` | no | Where the ad was shown |
| `mediaType` | `ALL` \| `IMAGE` \| `VIDEO` | no |  |
| `platform` | `ALL` \| `FACEBOOK` \| `INSTAGRAM` \| `MESSENGER` \| `AUDIENCE_NETWORK` \| `THREADS` \| `WHATSAPP` | no |  |
| `status` | `ACTIVE` \| `INACTIVE` \| `ALL` | no | ACTIVE: running now |
| `cursor` | string | no | nextCursor from the previous page of the same search; keep every other parameter unchanged. |

Example:

```bash
curl -s "https://faceless.so/api/v1/ads/search" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "adId": "2716161098763684",
        "pageId": "123456789",
        "pageName": "Example brand",
        "bodies": [
          "Turn one idea into a week of videos."
        ],
        "titles": [
          "Videos on autopilot"
        ],
        "captions": [],
        "descriptions": [],
        "startedAt": "2026-08-01T00:00:00.000Z",
        "stoppedAt": null,
        "platforms": [
          "FACEBOOK",
          "INSTAGRAM"
        ],
        "mediaType": null,
        "observedStatus": "ACTIVE",
        "sourceUrl": "https://www.facebook.com/ads/library/?id=2716161098763684",
        "fetchedAt": "2026-09-26T10:00:00.000Z",
        "savedId": null
      }
    ],
    "resultToken": "22df1b25-533f-4bd7-862a-aedc4a4ea0cc",
    "expiresAt": "2026-09-26T10:15:00.000Z",
    "nextCursor": "0b6f3c1e-8f4a-4c55-9e0b-7d2a1c9e4f10"
  }
}
```

### List the team's saved ads

`GET /saved-ads`

Returns the ads the team saved from Meta Ad Library searches, newest first, 24 per page, with each ad's details as Meta showed them when it was saved. Filter by one advertiser with pageId. Works even when fresh search is paused.

- Scopes: `ads:read`
- Credits: none
- CLI: `faceless ads saved`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `pageId` | string | no | Only this advertiser's ads |
| `page` | integer | no |  |

Example:

```bash
curl -s "https://faceless.so/api/v1/saved-ads" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "665f1b2a9c31a2b3c4d5f001",
        "ad": {
          "adId": "2716161098763684",
          "pageName": "Example brand"
        },
        "savedAt": "2026-09-26T10:01:00.000Z"
      }
    ],
    "advertisers": [
      {
        "pageId": "123456789",
        "pageName": "Example brand"
      }
    ],
    "savedCount": 1,
    "savedLimit": 500,
    "page": 1,
    "pages": 1
  }
}
```

### Save an ad from a search

`POST /saved-ads`

Saves one ad from a searchAds result to the team's saved ads, using that search's resultToken (valid 15 minutes; search again after that). Saving the same ad twice returns the first save. A team can keep 500 saved ads.

- Scopes: `ads:write`
- Credits: none
- CLI: `faceless ads save`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `adId` | string | yes | adId from searchAds |
| `resultToken` | string | yes | resultToken of the search page that returned the ad |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/saved-ads" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"adId":"2716161098763684","resultToken":"22df1b25-533f-4bd7-862a-aedc4a4ea0cc"}'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "665f1b2a9c31a2b3c4d5f001",
    "ad": {
      "adId": "2716161098763684",
      "pageName": "Example brand"
    },
    "savedAt": "2026-09-26T10:01:00.000Z"
  }
}
```

### Remove a saved ad

`DELETE /saved-ads/{id}`

Removes one of the team's saved ads. An id the team does not have answers 404.

- Scopes: `ads:write`
- Credits: none
- CLI: `faceless ads unsave`

Example:

```bash
curl -s -X DELETE "https://faceless.so/api/v1/saved-ads/<id>" \
  -H "Authorization: Bearer $FACELESS_API_KEY"
```

Response:

```json
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

### Write an ad brief for your brand from a competitor's ad

`POST /ads/briefs`

Writes an original ad brief (hook, promise, script, visual direction, call to action) for the team's brand, inspired by one ad from searchAds or the saved ads. Only the ad's approach carries over: every claim comes from the brand description you send, and the advertiser's name, offers, prices and results never appear. target sizes the script: motion (20 s motion-graphics ad), product-video (30 s narrated video), ugc (20 s creator talking to camera) or image (one static ad). Free; use the brief with createVideo or any other operation. 30 briefs per hour.

- Scopes: `ads:write`
- Credits: none
- Rate limit: 30 per 3600s
- CLI: `faceless ads brief`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `adId` | string | yes |  |
| `resultToken` | string | no | From searchAds; send this or savedId |
| `savedId` | string | no | A saved ad's id; send this or resultToken |
| `target` | `motion` \| `product-video` \| `ugc` \| `image` | yes | What the brief is for |
| `brand` | object | yes | Your brand |
| `notes` | string | no | A real offer, a feature to lead with, a tone |

Example:

```bash
curl -s -X POST "https://faceless.so/api/v1/ads/briefs" \
  -H "Authorization: Bearer $FACELESS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"adId":"2716161098763684","resultToken":"22df1b25-533f-4bd7-862a-aedc4a4ea0cc","target":"product-video","brand":{"name":"Acme Coffee","description":"Small-batch coffee beans roasted to order and shipped within two days."}}'
```

Response:

```json
{
  "success": true,
  "data": {
    "brief": {
      "hook": "Your coffee was roasted months ago",
      "benefit": "Beans roasted the day you order them, at your door two days later.",
      "script": "Your coffee was roasted months ago. Ours is roasted when you order. It reaches you two days later. Taste the difference.",
      "visualDirection": "Close-ups of beans leaving the roaster, a kraft bag being sealed, a morning pour.",
      "cta": "Order a fresh bag"
    },
    "source": {
      "adId": "2716161098763684",
      "pageName": "Example brand",
      "sourceUrl": "https://www.facebook.com/ads/library/?id=2716161098763684"
    }
  }
}
```
