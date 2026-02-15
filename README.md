# WeWeb Voice Recorder Component

Custom WeWeb Vue 3 component that records microphone audio, supports pause/resume, max-duration auto-stop, and optional silence auto-stop (energy-based VAD). It writes a structured result object to a selected WeWeb variable.

## Files
- `src/wwElement.vue`: component implementation
- `ww-config.js`: WeWeb editor configuration (properties + trigger events)
- `package.json`: serve/build scripts using `@weweb/cli`

## Local development
1. Install dependencies: `npm install`
2. Run dev server: `npm run serve -- --port=8080`
3. In WeWeb editor dev popup, add custom element from local URL.

## Build
- `npm run build -- --name=voice-recorder`

## Trigger events
- `onStart`
- `onPause`
- `onResume`
- `onStop`
- `onAutoStop`
- `onSave`
- `onError`
- `onPermissionDenied`

## Output object
The component writes this object to `outputVariable`:

```js
{
  status: 'ready' | 'error',
  blob: Blob | null,
  file: File | null,
  base64: string | null,
  mimeType: string,
  size: number,
  durationMs: number,
  createdAt: number,
  objectUrl: string | null,
  mode: 'manual' | 'autoStop',
  stoppedReason: 'manual' | 'silence' | 'maxDuration' | 'error',
  error: { code: string, message: string } | null,
}
```
