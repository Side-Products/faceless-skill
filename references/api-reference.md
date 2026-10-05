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

## credits

### Credit balance and history

`GET /credits`

Returns the team's current credit balance and a paginated ledger of credit transactions (purchases, subscription grants, video spends, refunds). Video creation charges credits at creation time; rendering and publishing are free. Costs: storyboard 20, motion_lite 50, motion_pro 100 credits per video.

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

Creates an AI faceless video from a finished narration script: TTS voiceover, AI-generated visuals per scene, captions and optional background music. Generation is asynchronous; the response returns the video id immediately. Poll GET /videos/{id}/status until status is completed, then call POST /videos/{id}/render to produce the final MP4. Credits are charged now, at creation.

- Scopes: `videos:write`
- Credits: 20 credits (storyboard), 50 (motion_lite) or 100 (motion_pro) per video, charged at creation
- Rate limit: 10 per 60s
- Supports `Idempotency-Key` header
- CLI: `faceless videos create`
- MCP tool: `faceless_create_video`

Body fields:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `script` | string | yes | The full narration script the video is generated from |
| `voiceId` | string | yes | TTS voice id for the narration. Full list: GET /voices |
| `model` | `storyboard` \| `motion_lite` \| `motion_pro` | no | Generation model: storyboard (20 credits, static scenes), motion_lite (50, animated transitions), motion_pro (100, cinematic) |
| `style` | string | no | Visual style for generated imagery. Full list: GET /options?kind=styles |
| `language` | string | no | Script language, e.g. English. Full list: GET /options?kind=languages |
| `name` | string | no | Project name; defaults to the start of the script |
| `enableBackgroundMusic` | boolean | no | Mix background music under the narration |
| `masterStyle` | string | no | Extra style directive applied to every generated scene |
| `globalNegativePrompt` | string | no | Things the image model should avoid in every scene |

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

Creates a project from a video or audio URL you already have: transcribes it, adds styled captions and emojis, and makes it editable. Use this to caption existing footage rather than generate new AI visuals (for that, use POST /videos). Processing is asynchronous; poll GET /videos/{id}/status.

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

Returns a single video project: script, voice, model, generation state, per-platform post metadata and the rendered MP4 URL once rendered.

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
| `voice` | string | no | TTS voice id for narration. Full list: GET /voices |
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
| `subreddit` | string | no | Subreddit to pull posts from when source is "Reddit post" |
| `backgroundVideo` | string | no | Background gameplay/footage id. Full list: GET /options?kind=backgrounds |
| `useRandomBackgroundVideo` | boolean | no | Pick a random background video per episode |
| `layout` | string | no | Video layout variant |
| `brollModel` | string | no | Generation model for visuals: storyboard, motion_lite or motion_pro |
| `showEmojis` | boolean | no | Overlay emojis on captions |
| `enableBackgroundMusic` | boolean | no | Mix background music under the narration |
| `enableVoiceover` | boolean | no | Story Slides and News Slides only: an AI voice reads each slide, using `voice` (a default voice when omitted). Defaults to false (music only) |
| `backgroundMusicMood` | string | no | Background music mood. Full list: GET /options?kind=music |
| `hashtags` | string | no | Hashtags appended to post captions |
| `tone` | string | no | Writing tone for generated scripts |
| `youtubePrivacyStatus` | `public` \| `unlisted` \| `private` | no | Privacy for auto-posted YouTube videos |

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
| `voice` | string | no | TTS voice id for narration. Full list: GET /voices |
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
| `subreddit` | string | no | Subreddit to pull posts from when source is "Reddit post" |
| `backgroundVideo` | string | no | Background gameplay/footage id. Full list: GET /options?kind=backgrounds |
| `useRandomBackgroundVideo` | boolean | no | Pick a random background video per episode |
| `layout` | string | no | Video layout variant |
| `brollModel` | string | no | Generation model for visuals: storyboard, motion_lite or motion_pro |
| `showEmojis` | boolean | no | Overlay emojis on captions |
| `enableBackgroundMusic` | boolean | no | Mix background music under the narration |
| `enableVoiceover` | boolean | no | Story Slides and News Slides only: an AI voice reads each slide, using `voice` (a default voice when omitted). Defaults to false (music only) |
| `backgroundMusicMood` | string | no | Background music mood. Full list: GET /options?kind=music |
| `hashtags` | string | no | Hashtags appended to post captions |
| `tone` | string | no | Writing tone for generated scripts |
| `youtubePrivacyStatus` | `public` \| `unlisted` \| `private` | no | Privacy for auto-posted YouTube videos |
| `paused` | boolean | no | Pause or resume automatic episode generation |

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

Returns the videos generated by a series, newest first, with their generation status, scheduled post time and posted state.

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
      "status": "posted",
      "scheduledTime": "2026-07-30T18:00:00.000Z"
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

Immediately publishes a rendered video to one connected platform (youtube, tiktok, instagram, x, facebook, linkedin or threads). The video must be rendered first (renderedVideoUrl present, or pass videoUrl explicitly) and the team must have that platform connected (GET /accounts). For future-dated posting use POST /posts/schedule instead.

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
| `authId` | string | no | Specific connected account id when the team has several for the platform (from GET /accounts) |

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

Brings a scheduled video forward to now: every pending posting task of the video (and its render, if that is what is still waiting) moves to the current time and goes out within a minute or two, to every platform it was scheduled for. Works for videos scheduled with POST /posts/schedule and for series episodes waiting for their slot. It does not post a video that was never scheduled (use POST /posts), a video that still needs approval (409; approve it first), or an episode of a paused series (409; resume the series first). Returns 400 when nothing is pending.

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

Returns the team's social posts (scheduled, posted and failed) within a date range, across all platforms. Use it to see what is queued before scheduling more.

- Scopes: `posts:read`
- Credits: none
- CLI: `faceless calendar`
- MCP tool: `faceless_get_calendar`

Query parameters:

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `startDate` | string | yes | Range start (inclusive) |
| `endDate` | string | yes | Range end (inclusive) |
| `platform` | `youtube` \| `tiktok` \| `instagram` \| `x` \| `facebook` \| `linkedin` \| `threads` | no | Filter to one platform |
| `status` | string | no | Filter by post status, e.g. scheduled, posted, failed |

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
      "id": "665f1b2a9c31a2b3c4d5ed01",
      "projectId": "665f1b2a9c31a2b3c4d5e801",
      "platform": "youtube",
      "status": "scheduled",
      "scheduledTime": "2026-08-01T18:00:00.000Z"
    }
  ]
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

Registers an already-hosted media file (video, image or audio URL) as a team asset usable in projects. This endpoint takes a URL, not raw bytes; host the file anywhere public (or use the dashboard for direct uploads).

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
| `fileType` | string | no | MIME type hint, e.g. video/mp4 |

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

## analytics

### Cross-platform posting analytics

`GET /analytics`

Returns aggregated performance for the team's published posts (views, likes, comments and posting volume) across YouTube, TikTok, Instagram and Facebook, over a trailing window.

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
      "youtube": {
        "totalViews": 88000,
        "postsSynced": 20,
        "viewsGained28d": 9400
      }
    },
    "topPosts": [
      {
        "platform": "youtube",
        "title": "Deep sea rivers",
        "views": 42000,
        "postUrl": "https://youtube.com/shorts/example"
      }
    ],
    "totals": {
      "totalViews": 128530,
      "postsSynced": 42,
      "posts7d": 7
    },
    "timeseriesReady": true
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
