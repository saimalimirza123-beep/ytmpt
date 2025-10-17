# Frontend Integration Guide

This API converts YouTube to MP3 using a two-step, asynchronous flow.

## Typical UX flow
1. User pastes a YouTube URL.
2. Call POST /prepare → get `conversion_id` + metadata instantly.
3. Immediately call POST /convert with the `conversion_id` and selected quality.
4. Poll GET /status/{conversion_id} every 2–5s.
5. When `status=completed`, use `download_url` to download.

Note: The status endpoint intentionally reports a simplified flow to improve UX:
- While work is ongoing (preparing/downloading/queued/converting), clients will see `status="preparing"` with a friendly `status_text` like "Preparing your audio…".
- Only terminal states surface as-is: `completed` or `failed`.

## Example calls

### Prepare
```js
const base = 'http://localhost:8080';
const res = await fetch(`${base}/prepare`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'X-API-Key': 'free_123' },
  body: JSON.stringify({ url: 'https://www.youtube.com/watch?v=VIDEO_ID' })
});
const data = await res.json();
const id = data.conversion_id;
```

### Convert
```js
const convertRes = await fetch(`${base}/convert`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'X-API-Key': 'free_123' },
  body: JSON.stringify({ conversion_id: id, quality: '320', start_time: '', end_time: '' })
});
const convertData = await convertRes.json();
// convertData.status: 'preparing' (fast path may be 'completed' if variant exists)
```

### Status (simplified top-level + detailed flow)
```js
const s = await fetch(`${base}/status/${id}`).then(r => r.json());
// s.status: 'preparing' | 'completed' | 'failed'
// s.flow: ordered steps with done/current flags
// Example s.flow names:
// [
//   { name: 'preparing', done: true },
//   { name: 'fetching_metadata', done: true },
//   { name: 'created', done: true },
//   { name: 'downloading', done: true },
//   { name: 'downloaded', done: true },
//   { name: 'Prossccing', current: true },
//   { name: 'converting' },
//   { name: 'completed' },
//   { name: 'failed' }
// ]
```

### Download
```js
if (s.status === 'completed' && s.download_url) {
  window.location.href = `${base}${s.download_url}`;
}
```

## Progress UI tips
- Show metadata immediately after /prepare (title/thumbnail/duration).
- Render a stepper from `s.flow` (use `done`/`current`).
- Show a single message from `s.status_text` (no queue position UI).
- When `s.status === 'completed'`, enable the Download button.
- If `s.status === 'failed'`, show `s.error` and allow retry.

## Error handling
- If /convert returns 202 but the job later fails, /status will show `status=failed` and may set `error`.
- Retry policy: re-queue /convert once after 30–60s if failed due to transient errors.
- Handle invalid/private videos: /prepare may succeed on metadata but download can fail; reflect errors from /status.

## API keys and priorities
- Send `X-API-Key` if your server requires it.
- Premium keys may be prioritized in queue; surface faster ETAs to users.

## CORS
- API sets CORS based on `ALLOWED_ORIGINS`. For local dev, set `*` or your host.

## Notes
- Conversion qualities: 128/192/256/320 (CBR) by default.
- Time range: set `start_time` and/or `end_time` as `HH:MM:SS`.
- No clip-length cap: only validate `start_time`/`end_time` format and that they fall within the video's duration.
- Videos longer than the configured `MAX_VIDEO_DURATION_SECONDS` (default 2400s = 40m) are rejected with an error including both the limit and the actual video duration.
- Identical requests may complete instantly if the variant was previously converted.
