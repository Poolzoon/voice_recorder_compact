<template>
  <div class="voice-recorder" :class="[`is-${uiState}`, { 'is-disabled': isDisabled }]" role="group" aria-label="Voice recorder">
    <button
      v-if="isIdleWithoutAudio"
      class="icon-btn icon-btn-mic single"
      type="button"
      :disabled="isDisabled || transitionLock"
      :aria-label="startButtonLabel"
      @click="handleMicToggle"
    >
      <svg viewBox="0 0 24 24" aria-hidden="true">
        <path
          d="M12 15a3 3 0 0 0 3-3V7a3 3 0 1 0-6 0v5a3 3 0 0 0 3 3Zm5-3a1 1 0 1 1 2 0a7 7 0 0 1-6 6.93V22a1 1 0 1 1-2 0v-3.07A7 7 0 0 1 5 12a1 1 0 1 1 2 0a5 5 0 0 0 10 0Z"
        />
      </svg>
    </button>

    <div v-else class="compact-shell">
      <span v-if="showTimer" class="timer" aria-live="polite">{{ timerLabel }}</span>

      <canvas
        v-if="showVisualizer"
        ref="canvasRef"
        class="visualizer compact-wave"
        :class="{ 'is-paused': isPaused, 'is-seekable': canSeekWave }"
        width="520"
        height="84"
        aria-label="Audio visualizer"
        @pointerdown="handleWavePointerDown"
      />

      <div class="actions">
        <template v-if="isActive">
          <button
            class="icon-btn icon-btn-pause"
            type="button"
            :disabled="!canPauseResume || transitionLock"
            :aria-label="isPaused ? resumeButtonLabel : pauseButtonLabel"
            @click="handlePauseResume"
          >
            <svg v-if="isPaused" viewBox="0 0 24 24" aria-hidden="true">
              <path d="M8.5 6.5a1 1 0 0 1 1.53-.85l8 5a1 1 0 0 1 0 1.7l-8 5A1 1 0 0 1 8.5 16.5v-10Z" />
            </svg>
            <svg v-else viewBox="0 0 24 24" aria-hidden="true">
              <path d="M7 6a1 1 0 0 1 1-1h2.5a1 1 0 0 1 1 1v12a1 1 0 0 1-1 1H8a1 1 0 0 1-1-1V6Zm5.5 0a1 1 0 0 1 1-1H16a1 1 0 0 1 1 1v12a1 1 0 0 1-1 1h-2.5a1 1 0 0 1-1-1V6Z" />
            </svg>
          </button>
          <button
            class="icon-btn icon-btn-stop"
            type="button"
            :disabled="transitionLock"
            :aria-label="stopButtonLabel"
            @click="handleMicToggle"
          >
            <svg viewBox="0 0 24 24" aria-hidden="true">
              <rect x="7" y="7" width="10" height="10" rx="2" />
            </svg>
          </button>
        </template>

        <template v-else-if="hasPlayableAudio">
          <button
            class="icon-btn"
            type="button"
            :disabled="transitionLock || isDisabled"
            :aria-label="isPlaybackActive ? stopPlayButtonLabel : playButtonLabel"
            @click="handlePlay"
          >
            <svg v-if="isPlaybackActive" viewBox="0 0 24 24" aria-hidden="true">
              <rect x="7" y="7" width="10" height="10" rx="2" />
            </svg>
            <svg v-else viewBox="0 0 24 24" aria-hidden="true">
              <path d="M8.5 6.5a1 1 0 0 1 1.53-.85l8 5a1 1 0 0 1 0 1.7l-8 5A1 1 0 0 1 8.5 16.5v-10Z" />
            </svg>
          </button>
          <button
            class="icon-btn icon-btn-delete"
            type="button"
            :disabled="transitionLock || isDisabled"
            :aria-label="deleteButtonLabel"
            @click="handleDelete"
          >
            <svg viewBox="0 0 24 24" aria-hidden="true">
              <path d="M9.5 3a1 1 0 0 0-1 1V5H6a1 1 0 1 0 0 2h.29l.9 11.04A3 3 0 0 0 10.18 21h3.64a3 3 0 0 0 2.99-2.96L17.71 7H18a1 1 0 1 0 0-2h-2.5V4a1 1 0 0 0-1-1h-5Zm1 2V5h3V5h-3Z" />
            </svg>
          </button>
        </template>
      </div>
    </div>

    <p v-if="errorMessage" class="error" role="alert">{{ errorMessage }}</p>
  </div>
</template>

<script setup>
import { computed, nextTick, onBeforeUnmount, ref } from 'vue';

const props = defineProps({
  content: { type: Object, required: true },
  uid: { type: String, default: '' },
});

const emit = defineEmits(['trigger-event']);

/* wwEditor:start */
const __wwEditorMarker = true;
/* wwEditor:end */

const uiState = ref('idle'); // idle | recording | paused | stopping | stopped | error
const transitionLock = ref(false);
const errorMessage = ref('');

const chunks = ref([]);
const mediaRecorderRef = ref(null);
const mediaStreamRef = ref(null);
const audioContextRef = ref(null);
const analyserRef = ref(null);
const sourceNodeRef = ref(null);
const rafIdRef = ref(null);
const tickIntervalRef = ref(null);
const canvasRef = ref(null);

const currentMimeType = ref('');
const stopReasonRef = ref('manual');
const lastObjectUrl = ref(null);
const latestPlayableUrl = ref('');
const audioPlayerRef = ref(null);
const isPlaybackActive = ref(false);
const playbackRenderRafRef = ref(null);
const playbackCurrentMs = ref(0);
const recordedDurationMs = ref(0);
const waveformBars = ref([]);
const isWaveSeeking = ref(false);
const wavePointerId = ref(null);

const nowMs = ref(Date.now());
const accumulatedRecordedMs = ref(0);
const segmentStartMs = ref(null);

const speechDetected = ref(false);
const silenceStartMs = ref(null);
const noiseSamples = ref([]);
const noiseFloor = ref(0.005);

const fallbackInternal = ref(null);
const componentVarApi = wwLib?.wwVariable?.useComponentVariable;
const internalVar = componentVarApi
  ? componentVarApi({
      uid: props?.uid,
      name: 'latestOutput',
      type: 'object',
      defaultValue: null,
    })
  : { value: fallbackInternal, setValue: value => (fallbackInternal.value = value) };

const mode = computed(() => (props?.content?.mode === 'autoStop' ? 'autoStop' : 'manual'));
const maxDurationSec = computed(() => {
  const value = Number(props?.content?.maxDurationSec ?? 60);
  return Number.isFinite(value) ? value : 60;
});
const showVisualizer = computed(() => props?.content?.showVisualizer !== false);
const showTimer = computed(() => props?.content?.showTimer !== false);
const isDisabled = computed(() => !!props?.content?.disabled);
const startButtonLabel = computed(() => String(props?.content?.startButtonLabel || 'Start'));
const stopButtonLabel = computed(() => String(props?.content?.stopButtonLabel || 'Stop'));
const pauseButtonLabel = computed(() => String(props?.content?.pauseButtonLabel || 'Pause'));
const resumeButtonLabel = computed(() => String(props?.content?.resumeButtonLabel || 'Resume'));
const playButtonLabel = computed(() => String(props?.content?.playButtonLabel || 'Play'));
const stopPlayButtonLabel = computed(() => String(props?.content?.stopPlayButtonLabel || 'Stop'));
const deleteButtonLabel = computed(() => String(props?.content?.deleteButtonLabel || 'Delete'));
const visualizerBackgroundColor = computed(() => String(props?.content?.visualizerBackgroundColor || '#111827'));
const visualizerBarsColor = computed(() => String(props?.content?.visualizerBarsColor || '#10b981'));
const visualizerPausedBarsColor = computed(() => String(props?.content?.visualizerPausedBarsColor || '#9ca3af'));
const outputFormat = computed(() => {
  const format = props?.content?.outputFormat;
  return format === 'file' || format === 'base64' ? format : 'blob';
});

const noiseCalibrationMs = computed(() => Math.max(0, Number(props?.content?.noiseCalibrationMs ?? 500)));
const silenceDurationMs = computed(() => Math.max(200, Number(props?.content?.silenceDurationMs ?? 1500)));
const thresholdMultiplier = computed(() => Math.max(1, Number(props?.content?.thresholdMultiplier ?? 2.2)));
const minSpeechMsBeforeAutoStop = computed(() => Math.max(0, Number(props?.content?.minSpeechMsBeforeAutoStop ?? 800)));

const isRecording = computed(() => uiState.value === 'recording');
const isPaused = computed(() => uiState.value === 'paused');
const isActive = computed(() => isRecording.value || isPaused.value);
const canPauseResume = computed(() => isActive.value);
const hasPlayableAudio = computed(() => !!latestPlayableUrl.value);
const canSeekWave = computed(() => isPlaybackActive.value && !isActive.value && !transitionLock.value && !isDisabled.value);
const isIdleWithoutAudio = computed(() => !hasPlayableAudio.value && !isActive.value && uiState.value !== 'stopping');

const elapsedMs = computed(() => {
  if (isRecording.value && segmentStartMs.value) {
    return accumulatedRecordedMs.value + (nowMs.value - segmentStartMs.value);
  }
  return accumulatedRecordedMs.value;
});

const timerLabel = computed(() => {
  if (isActive.value) return formatMsAsClock(elapsedMs.value);
  if (hasPlayableAudio.value && recordedDurationMs.value > 0 && !isPlaybackActive.value) {
    return formatMsAsClock(recordedDurationMs.value);
  }
  if (hasPlayableAudio.value && recordedDurationMs.value > 0 && isPlaybackActive.value) {
    return formatMsAsClock(playbackCurrentMs.value);
  }
  return formatMsAsClock(elapsedMs.value);
});

function frontWindow() {
  return wwLib?.getFrontWindow?.();
}

function emitTrigger(name, payload = {}) {
  emit('trigger-event', { name, event: payload });
}

function formatMsAsClock(ms) {
  const totalSec = Math.floor(Math.max(0, ms) / 1000);
  const mm = String(Math.floor(totalSec / 60)).padStart(2, '0');
  const ss = String(totalSec % 60).padStart(2, '0');
  return `${mm}:${ss}`;
}

function resolveOutputVariableId(target) {
  if (typeof target === 'string' && target) return target;
  if (target && typeof target === 'object') {
    return target?.id || target?._id || target?.variableId || target?.value || null;
  }
  return null;
}

async function writeToOutputVariable(value) {
  const variableId = resolveOutputVariableId(props?.content?.outputVariable);
  if (!variableId) throw new Error('Missing outputVariable binding.');

  if (typeof wwLib?.wwVariable?.updateValue === 'function') {
    await wwLib.wwVariable.updateValue(variableId, value);
    return;
  }
  if (typeof wwLib?.wwVariable?.setValue === 'function') {
    await wwLib.wwVariable.setValue(variableId, value);
    return;
  }
  if (typeof wwLib?.wwVariable?.setVariableValue === 'function') {
    await wwLib.wwVariable.setVariableValue(variableId, value);
    return;
  }
  throw new Error('No supported WeWeb variable write API found.');
}

function chooseMimeType() {
  const win = frontWindow();
  const MediaRecorderCtor = win?.MediaRecorder;
  if (!MediaRecorderCtor) return '';

  const preferred = props?.content?.preferredMimeType || 'audio/webm;codecs=opus';
  const candidates = [preferred, 'audio/webm;codecs=opus', 'audio/webm', 'audio/mp4', 'audio/aac'];

  for (const type of candidates) {
    if (type && MediaRecorderCtor?.isTypeSupported?.(type)) return type;
  }
  return '';
}

function startTicker() {
  stopTicker();
  tickIntervalRef.value = setInterval(() => {
    nowMs.value = Date.now();
    if (isRecording.value && maxDurationSec.value > 0 && elapsedMs.value >= maxDurationSec.value * 1000) {
      void requestStop('maxDuration');
    }
  }, 100);
}

function stopTicker() {
  if (tickIntervalRef.value) {
    clearInterval(tickIntervalRef.value);
    tickIntervalRef.value = null;
  }
}

function startVisualizerLoop() {
  stopVisualizerLoop();

  const win = frontWindow();
  const analyser = analyserRef.value;
  const canvas = canvasRef.value;
  if (!win || !analyser || !canvas) return;

  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  const freq = new Uint8Array(analyser.frequencyBinCount);
  const timeData = new Uint8Array(analyser.fftSize);

  const draw = () => {
    rafIdRef.value = win.requestAnimationFrame(draw);

    analyser.getByteFrequencyData(freq);
    analyser.getByteTimeDomainData(timeData);

    const rms = computeRmsFromTimeDomain(timeData);
    evaluateVad(rms);

    const width = canvas.width;
    const height = canvas.height;
    ctx.clearRect(0, 0, width, height);

    const bars = 40;
    const barWidth = Math.max(2, Math.floor(width / bars) - 2);
    const step = Math.max(1, Math.floor(freq.length / bars));

    for (let i = 0; i < bars; i += 1) {
      const value = freq[i * step] / 255;
      const barHeight = Math.max(2, Math.floor(value * (height - 4)));
      const x = i * (barWidth + 2);
      const y = height - barHeight;
      ctx.fillStyle = isPaused.value ? visualizerPausedBarsColor.value : visualizerBarsColor.value;
      ctx.fillRect(x, y, barWidth, barHeight);
    }
  };

  draw();
}

function stopVisualizerLoop() {
  const win = frontWindow();
  if (rafIdRef.value && win?.cancelAnimationFrame) {
    win.cancelAnimationFrame(rafIdRef.value);
  }
  rafIdRef.value = null;
}

function defaultWaveBars(count = 64) {
  const bars = [];
  for (let i = 0; i < count; i += 1) {
    const base = i % 5 === 0 ? 0.55 : i % 3 === 0 ? 0.4 : 0.28;
    bars.push(base);
  }
  return bars;
}

function drawPlaybackWave() {
  const canvas = canvasRef.value;
  if (!canvas || !showVisualizer.value) return;

  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  const bars = waveformBars.value.length ? waveformBars.value : defaultWaveBars();
  const width = canvas.width;
  const height = canvas.height;
  const spacing = 2;
  const barWidth = Math.max(2, Math.floor((width - spacing * (bars.length - 1)) / bars.length));
  const totalBarsWidth = bars.length * barWidth + (bars.length - 1) * spacing;
  const offsetX = Math.max(0, Math.floor((width - totalBarsWidth) / 2));

  const duration = Math.max(1, recordedDurationMs.value);
  const progressRatio = isPlaybackActive.value
    ? Math.min(1, Math.max(0, playbackCurrentMs.value / duration))
    : 1;

  ctx.clearRect(0, 0, width, height);
  ctx.fillStyle = visualizerBackgroundColor.value;
  ctx.fillRect(0, 0, width, height);

  for (let i = 0; i < bars.length; i += 1) {
    const bar = Math.min(1, Math.max(0.06, bars[i] || 0.1));
    const barHeight = Math.max(4, Math.floor(bar * (height - 10)));
    const x = offsetX + i * (barWidth + spacing);
    const y = Math.floor((height - barHeight) / 2);
    const ratio = bars.length <= 1 ? 1 : i / (bars.length - 1);
    ctx.fillStyle = ratio <= progressRatio ? visualizerBarsColor.value : '#d1d5db';
    ctx.fillRect(x, y, barWidth, barHeight);
  }
}

function startPlaybackRenderLoop() {
  stopPlaybackRenderLoop();
  const win = frontWindow();
  if (!win?.requestAnimationFrame || !audioPlayerRef.value) return;

  const draw = () => {
    playbackRenderRafRef.value = win.requestAnimationFrame(draw);
    if (!audioPlayerRef.value) return;
    playbackCurrentMs.value = Math.max(0, Math.round((audioPlayerRef.value.currentTime || 0) * 1000));
    drawPlaybackWave();
  };

  draw();
}

function stopPlaybackRenderLoop() {
  const win = frontWindow();
  if (playbackRenderRafRef.value && win?.cancelAnimationFrame) {
    win.cancelAnimationFrame(playbackRenderRafRef.value);
  }
  playbackRenderRafRef.value = null;
}

function computeRmsFromTimeDomain(timeDomainU8) {
  if (!timeDomainU8?.length) return 0;
  let sum = 0;
  for (let i = 0; i < timeDomainU8.length; i += 1) {
    const normalized = (timeDomainU8[i] - 128) / 128;
    sum += normalized * normalized;
  }
  return Math.sqrt(sum / timeDomainU8.length);
}

function evaluateVad(rms) {
  if (!isRecording.value || mode.value !== 'autoStop') return;

  const elapsed = elapsedMs.value;
  if (elapsed <= noiseCalibrationMs.value) {
    noiseSamples.value.push(rms);
    const avg = noiseSamples.value.reduce((acc, v) => acc + v, 0) / Math.max(1, noiseSamples.value.length);
    noiseFloor.value = Math.max(0.001, avg);
    return;
  }

  const threshold = Math.max(noiseFloor.value * thresholdMultiplier.value, noiseFloor.value + 0.005);
  const speaking = rms > threshold;

  if (speaking) {
    speechDetected.value = true;
    silenceStartMs.value = null;
    return;
  }

  const eligible = speechDetected.value && elapsed >= minSpeechMsBeforeAutoStop.value;
  if (!eligible) return;

  if (silenceStartMs.value == null) {
    silenceStartMs.value = Date.now();
    return;
  }

  if (Date.now() - silenceStartMs.value >= silenceDurationMs.value) {
    void requestStop('silence');
  }
}

function createErrorOutput(code, message) {
  return {
    status: 'error',
    blob: null,
    file: null,
    base64: null,
    mimeType: '',
    size: 0,
    durationMs: 0,
    createdAt: Date.now(),
    objectUrl: null,
    mode: mode.value,
    stoppedReason: 'error',
    error: { code, message },
  };
}

async function persistErrorOutput(code, message) {
  const payload = createErrorOutput(code, message);
  try {
    await writeToOutputVariable(payload);
  } catch (_) {
    // best effort
  }
  internalVar?.setValue?.(payload);
}

function handleFatalError(code, err) {
  const message = err?.message || 'Unexpected recorder error.';
  errorMessage.value = message;
  uiState.value = 'error';
  emitTrigger('onError', { code, message });
  void persistErrorOutput(code, message);
}

async function handleMicToggle() {
  if (transitionLock.value || isDisabled.value) return;
  if (!isActive.value) {
    await startRecording();
    return;
  }
  await requestStop('manual');
}

async function handlePauseResume() {
  if (transitionLock.value || !canPauseResume.value) return;
  const recorder = mediaRecorderRef.value;
  if (!recorder) return;

  try {
    if (isRecording.value && recorder.state === 'recording') {
      recorder.pause();
      accumulatedRecordedMs.value += Date.now() - (segmentStartMs.value || Date.now());
      segmentStartMs.value = null;
      uiState.value = 'paused';
      emitTrigger('onPause', { elapsedMs: elapsedMs.value, mode: mode.value });
      return;
    }

    if (isPaused.value && recorder.state === 'paused') {
      recorder.resume();
      segmentStartMs.value = Date.now();
      uiState.value = 'recording';
      emitTrigger('onResume', { elapsedMs: elapsedMs.value, mode: mode.value });
    }
  } catch (err) {
    handleFatalError('PAUSE_RESUME_FAILED', err);
  }
}

function handlePlay() {
  if (!latestPlayableUrl.value || isActive.value || transitionLock.value) return;
  const win = frontWindow();
  const AudioCtor = win?.Audio;
  if (!AudioCtor) return;

  if (isPlaybackActive.value && audioPlayerRef.value) {
    stopPlayback();
    return;
  }

  if (!audioPlayerRef.value) {
    audioPlayerRef.value = new AudioCtor(latestPlayableUrl.value);
    audioPlayerRef.value.onended = () => {
      isPlaybackActive.value = false;
      playbackCurrentMs.value = 0;
      stopPlaybackRenderLoop();
      drawPlaybackWave();
    };
    audioPlayerRef.value.onpause = () => {
      isPlaybackActive.value = false;
      stopPlaybackRenderLoop();
      drawPlaybackWave();
    };
    audioPlayerRef.value.ontimeupdate = () => {
      playbackCurrentMs.value = Math.max(0, Math.round((audioPlayerRef.value?.currentTime || 0) * 1000));
      drawPlaybackWave();
    };
    audioPlayerRef.value.onloadedmetadata = () => {
      const durationSec = Number(audioPlayerRef.value?.duration || 0);
      if (Number.isFinite(durationSec) && durationSec > 0) {
        recordedDurationMs.value = Math.max(recordedDurationMs.value, Math.round(durationSec * 1000));
      }
      drawPlaybackWave();
    };
  }

  if (audioPlayerRef.value.src !== latestPlayableUrl.value) {
    audioPlayerRef.value.src = latestPlayableUrl.value;
  }

  try {
    audioPlayerRef.value.currentTime = 0;
    playbackCurrentMs.value = 0;
    const playPromise = audioPlayerRef.value.play?.();
    isPlaybackActive.value = true;
    startPlaybackRenderLoop();
    if (playPromise && typeof playPromise.catch === 'function') {
      playPromise.catch(() => {
        isPlaybackActive.value = false;
        stopPlaybackRenderLoop();
        drawPlaybackWave();
      });
    }
    emitTrigger('onPlay', {
      objectUrl: latestPlayableUrl.value,
      mimeType: currentMimeType.value,
    });
  } catch (_) {
    isPlaybackActive.value = false;
  }
}

function stopPlayback() {
  if (!audioPlayerRef.value) return;
  try {
    audioPlayerRef.value.pause?.();
    audioPlayerRef.value.currentTime = 0;
  } catch (_) {
    // no-op
  } finally {
    isPlaybackActive.value = false;
    playbackCurrentMs.value = 0;
    stopPlaybackRenderLoop();
    drawPlaybackWave();
  }
}

function seekPlaybackByClientX(clientX) {
  if (!canSeekWave.value || !audioPlayerRef.value || !canvasRef.value) return;
  const rect = canvasRef.value.getBoundingClientRect();
  if (!rect?.width) return;

  const ratio = Math.min(1, Math.max(0, (clientX - rect.left) / rect.width));
  const durationSec = Number(audioPlayerRef.value.duration || 0);
  if (!Number.isFinite(durationSec) || durationSec <= 0) return;

  audioPlayerRef.value.currentTime = ratio * durationSec;
  playbackCurrentMs.value = Math.round(audioPlayerRef.value.currentTime * 1000);
  drawPlaybackWave();
}

function onWavePointerMove(event) {
  if (!isWaveSeeking.value) return;
  if (wavePointerId.value != null && event.pointerId !== wavePointerId.value) return;
  seekPlaybackByClientX(event.clientX);
}

function stopWaveSeek() {
  if (!isWaveSeeking.value) return;
  isWaveSeeking.value = false;
  wavePointerId.value = null;
  const doc = wwLib?.getFrontDocument?.();
  doc?.removeEventListener?.('pointermove', onWavePointerMove);
  doc?.removeEventListener?.('pointerup', stopWaveSeek);
  doc?.removeEventListener?.('pointercancel', stopWaveSeek);
}

function handleWavePointerDown(event) {
  if (!canSeekWave.value || !showVisualizer.value) return;
  isWaveSeeking.value = true;
  wavePointerId.value = event.pointerId;
  seekPlaybackByClientX(event.clientX);

  const doc = wwLib?.getFrontDocument?.();
  doc?.addEventListener?.('pointermove', onWavePointerMove);
  doc?.addEventListener?.('pointerup', stopWaveSeek);
  doc?.addEventListener?.('pointercancel', stopWaveSeek);
}

async function handleDelete() {
  if (!hasPlayableAudio.value || isActive.value || transitionLock.value) return;

  const hadAudio = !!latestPlayableUrl.value;
  stopPlayback();
  revokeObjectUrl();
  latestPlayableUrl.value = '';
  recordedDurationMs.value = 0;
  playbackCurrentMs.value = 0;
  waveformBars.value = [];

  const clearedOutput = {
    status: 'ready',
    blob: null,
    file: null,
    base64: null,
    mimeType: '',
    size: 0,
    durationMs: 0,
    createdAt: Date.now(),
    objectUrl: null,
    mode: mode.value,
    stoppedReason: 'manual',
    error: null,
  };

  try {
    await writeToOutputVariable(clearedOutput);
  } catch (_) {
    // best effort only
  }
  internalVar?.setValue?.(clearedOutput);
  emitTrigger('onDelete', { hadAudio });
}

async function startRecording() {
  transitionLock.value = true;
  errorMessage.value = '';

  try {
    const win = frontWindow();
    const mediaDevices = win?.navigator?.mediaDevices;
    const MediaRecorderCtor = win?.MediaRecorder;
    const AudioContextCtor = win?.AudioContext || win?.webkitAudioContext;

    if (!mediaDevices?.getUserMedia) throw new Error('getUserMedia is not available.');
    if (!MediaRecorderCtor) throw new Error('MediaRecorder is not supported in this browser.');

    const mimeType = chooseMimeType();
    currentMimeType.value = mimeType || 'audio/webm';

    const stream = await mediaDevices.getUserMedia({ audio: true });
    mediaStreamRef.value = stream;

    chunks.value = [];
    accumulatedRecordedMs.value = 0;
    segmentStartMs.value = Date.now();
    nowMs.value = Date.now();
    speechDetected.value = false;
    silenceStartMs.value = null;
    noiseSamples.value = [];
    noiseFloor.value = 0.005;
    stopReasonRef.value = 'manual';

    const recorder = mimeType ? new MediaRecorderCtor(stream, { mimeType }) : new MediaRecorderCtor(stream);
    mediaRecorderRef.value = recorder;

    recorder.ondataavailable = event => {
      if (event?.data && event.data.size > 0) chunks.value.push(event.data);
    };

    recorder.onerror = event => {
      handleFatalError('RECORDER_ERROR', event?.error || new Error('MediaRecorder error'));
    };

    if (AudioContextCtor) {
      const audioContext = new AudioContextCtor();
      audioContextRef.value = audioContext;
      sourceNodeRef.value = audioContext.createMediaStreamSource(stream);
      analyserRef.value = audioContext.createAnalyser();
      analyserRef.value.fftSize = 2048;
      analyserRef.value.smoothingTimeConstant = 0.85;
      sourceNodeRef.value.connect(analyserRef.value);
    }

    recorder.start(250);
    uiState.value = 'recording';
    await nextTick();
    startVisualizerLoop();
    startTicker();

    emitTrigger('onStart', {
      mode: mode.value,
      mimeType: currentMimeType.value,
      startedAt: Date.now(),
    });
  } catch (err) {
    const name = err?.name || '';
    if (name === 'NotAllowedError' || name === 'PermissionDeniedError') {
      const payload = { code: 'PERMISSION_DENIED', message: 'Microphone permission denied.' };
      errorMessage.value = payload.message;
      uiState.value = 'error';
      emitTrigger('onPermissionDenied', payload);
      emitTrigger('onError', payload);
      void persistErrorOutput(payload.code, payload.message);
    } else {
      handleFatalError('START_FAILED', err);
    }

    await fullCleanup({ keepErrorState: true });
  } finally {
    transitionLock.value = false;
  }
}

function requestStop(reason) {
  if (!mediaRecorderRef.value) return Promise.resolve();
  if (uiState.value === 'stopping' || uiState.value === 'stopped') return Promise.resolve();

  stopReasonRef.value = reason;
  uiState.value = 'stopping';
  transitionLock.value = true;

  if (isRecording.value && segmentStartMs.value) {
    accumulatedRecordedMs.value += Date.now() - segmentStartMs.value;
    segmentStartMs.value = null;
  }

  stopTicker();

  const recorder = mediaRecorderRef.value;
  return new Promise(resolve => {
    const finish = async () => {
      recorder.onstop = null;
      await finalizeRecording();
      transitionLock.value = false;
      resolve();
    };

    recorder.onstop = () => {
      void finish();
    };

    try {
      if (recorder.state !== 'inactive') recorder.stop();
      else void finish();
    } catch (err) {
      handleFatalError('STOP_FAILED', err);
      void finish();
    }
  });
}

async function finalizeRecording() {
  try {
    const blob = new Blob(chunks.value, { type: currentMimeType.value || 'audio/webm' });
    const createdAt = Date.now();
    const durationMs = await estimateDurationMs(blob, elapsedMs.value);
    recordedDurationMs.value = durationMs;
    playbackCurrentMs.value = 0;
    waveformBars.value = await buildWaveformBars(blob);

    const extension = mimeToExtension(blob.type || currentMimeType.value);
    const fileName = `voice-recording-${createdAt}.${extension}`;
    const file = tryCreateFile(blob, fileName);
    const objectUrl = createFreshObjectUrl(blob);

    let encodedBase64 = null;
    if (outputFormat.value === 'base64') encodedBase64 = await blobToBase64(blob);

    const output = {
      status: 'ready',
      blob: outputFormat.value === 'blob' ? blob : null,
      file: outputFormat.value === 'file' ? file : null,
      base64: outputFormat.value === 'base64' ? encodedBase64 : null,
      mimeType: blob.type || currentMimeType.value || '',
      size: blob.size || 0,
      durationMs,
      createdAt,
      objectUrl,
      mode: mode.value,
      stoppedReason: stopReasonRef.value,
      error: null,
    };

    await writeToOutputVariable(output);
    internalVar?.setValue?.(output);
    latestPlayableUrl.value = objectUrl || '';

    emitTrigger('onStop', { ...output, elapsedMs: elapsedMs.value });
    if (stopReasonRef.value === 'silence' || stopReasonRef.value === 'maxDuration') {
      emitTrigger('onAutoStop', { ...output, elapsedMs: elapsedMs.value });
    }
    emitTrigger('onSave', { ...output, elapsedMs: elapsedMs.value });

    uiState.value = 'stopped';
    await fullCleanup({ preserveObjectUrl: true });
    uiState.value = 'idle';
    await nextTick();
    drawPlaybackWave();
  } catch (err) {
    handleFatalError('FINALIZE_FAILED', err);
    await fullCleanup({ keepErrorState: true });
  }
}

async function buildWaveformBars(blob) {
  const barCount = 64;
  try {
    const win = frontWindow();
    const AudioContextCtor = win?.AudioContext || win?.webkitAudioContext;
    if (!AudioContextCtor) return defaultWaveBars(barCount);

    const context = new AudioContextCtor();
    const buffer = await blob.arrayBuffer();
    const decoded = await context.decodeAudioData(buffer.slice(0));
    await context.close();

    const channel = decoded?.numberOfChannels ? decoded.getChannelData(0) : null;
    if (!channel?.length) return defaultWaveBars(barCount);

    const chunkSize = Math.max(1, Math.floor(channel.length / barCount));
    const bars = [];

    for (let i = 0; i < barCount; i += 1) {
      const start = i * chunkSize;
      const end = Math.min(channel.length, start + chunkSize);
      let peak = 0;
      for (let j = start; j < end; j += 1) {
        const sample = Math.abs(channel[j] || 0);
        if (sample > peak) peak = sample;
      }
      bars.push(peak);
    }

    const max = Math.max(...bars, 0.001);
    return bars.map(value => Math.max(0.06, Math.min(1, value / max)));
  } catch (_) {
    return defaultWaveBars(barCount);
  }
}

function mimeToExtension(mime) {
  if (mime?.includes('webm')) return 'webm';
  if (mime?.includes('mp4')) return 'mp4';
  if (mime?.includes('aac')) return 'aac';
  if (mime?.includes('ogg')) return 'ogg';
  return 'dat';
}

function tryCreateFile(blob, fileName) {
  const win = frontWindow();
  const FileCtor = win?.File;
  if (!FileCtor) return null;
  try {
    return new FileCtor([blob], fileName, { type: blob.type || 'application/octet-stream' });
  } catch (_) {
    return null;
  }
}

function createFreshObjectUrl(blob) {
  const win = frontWindow();
  const urlApi = win?.URL;
  if (!urlApi?.createObjectURL) return null;

  if (lastObjectUrl.value && urlApi?.revokeObjectURL) {
    urlApi.revokeObjectURL(lastObjectUrl.value);
  }

  const next = urlApi.createObjectURL(blob);
  lastObjectUrl.value = next;
  return next;
}

function revokeObjectUrl() {
  const win = frontWindow();
  const urlApi = win?.URL;
  if (lastObjectUrl.value && urlApi?.revokeObjectURL) {
    urlApi.revokeObjectURL(lastObjectUrl.value);
  }
  lastObjectUrl.value = null;
}

function blobToBase64(blob) {
  return new Promise((resolve, reject) => {
    const win = frontWindow();
    const FileReaderCtor = win?.FileReader;
    if (!FileReaderCtor) {
      reject(new Error('FileReader not available.'));
      return;
    }

    const reader = new FileReaderCtor();
    reader.onerror = () => reject(new Error('Failed to convert blob to base64.'));
    reader.onloadend = () => {
      resolve(typeof reader.result === 'string' ? reader.result : '');
    };
    reader.readAsDataURL(blob);
  });
}

async function estimateDurationMs(blob, fallbackMs) {
  try {
    const win = frontWindow();
    const AudioContextCtor = win?.AudioContext || win?.webkitAudioContext;
    if (!AudioContextCtor) return Math.max(0, Math.round(fallbackMs));

    const context = new AudioContextCtor();
    const buffer = await blob.arrayBuffer();
    const decoded = await context.decodeAudioData(buffer.slice(0));
    await context.close();

    const ms = Math.round((decoded?.duration || 0) * 1000);
    return ms > 0 ? ms : Math.max(0, Math.round(fallbackMs));
  } catch (_) {
    return Math.max(0, Math.round(fallbackMs));
  }
}

async function fullCleanup(options = {}) {
  const { preserveObjectUrl = false, keepErrorState = false } = options;

  stopTicker();
  stopVisualizerLoop();

  const recorder = mediaRecorderRef.value;
  if (recorder) {
    recorder.ondataavailable = null;
    recorder.onerror = null;
    recorder.onstop = null;
  }
  mediaRecorderRef.value = null;

  if (sourceNodeRef.value) {
    try {
      sourceNodeRef.value.disconnect();
    } catch (_) {
      // no-op
    }
    sourceNodeRef.value = null;
  }
  analyserRef.value = null;

  if (audioContextRef.value) {
    try {
      await audioContextRef.value.close();
    } catch (_) {
      // no-op
    }
    audioContextRef.value = null;
  }

  if (mediaStreamRef.value) {
    const tracks = mediaStreamRef.value?.getTracks?.() || [];
    for (const track of tracks) {
      try {
        track.stop();
      } catch (_) {
        // no-op
      }
    }
    mediaStreamRef.value = null;
  }

  chunks.value = [];
  nowMs.value = Date.now();
  accumulatedRecordedMs.value = 0;
  segmentStartMs.value = null;
  silenceStartMs.value = null;
  speechDetected.value = false;

  if (!preserveObjectUrl) revokeObjectUrl();
  if (!preserveObjectUrl) latestPlayableUrl.value = '';
  if (!preserveObjectUrl) {
    recordedDurationMs.value = 0;
    playbackCurrentMs.value = 0;
    waveformBars.value = [];
  }

  stopPlayback();
  if (!keepErrorState && uiState.value !== 'idle') uiState.value = 'idle';
}

onBeforeUnmount(() => {
  stopWaveSeek();
  stopPlaybackRenderLoop();
  void fullCleanup();
});
</script>

<style scoped>
.voice-recorder {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.compact-shell {
  width: 100%;
  min-height: 64px;
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 12px;
  padding: 8px 10px 8px 14px;
  border-radius: 999px;
  border: 1px solid #eceff5;
  background: #fff;
  box-shadow: 0 8px 18px rgba(17, 24, 39, 0.08);
}

.actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.icon-btn {
  width: 42px;
  height: 42px;
  border: 0;
  border-radius: 999px;
  background: #8b5cf6;
  color: #fff;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  flex: 0 0 auto;
}

.icon-btn svg {
  width: 20px;
  height: 20px;
  fill: currentColor;
}

.icon-btn-delete {
  background: #ef4444;
}

.icon-btn-pause {
  background: #0ea5e9;
}

.icon-btn-stop {
  background: #f59e0b;
}

.icon-btn-mic {
  background: #8b5cf6;
}

.icon-btn.single {
  align-self: flex-start;
}

.icon-btn:disabled {
  cursor: not-allowed;
  opacity: 0.45;
}

.timer {
  min-width: 50px;
  font-variant-numeric: tabular-nums;
  font-size: 30px;
  line-height: 1;
  color: #8b5cf6;
}

.visualizer {
  width: 100%;
  height: 84px;
  border: 0;
  border-radius: 10px;
  background: v-bind(visualizerBackgroundColor);
}

.compact-wave {
  min-width: 60px;
}

.visualizer.is-paused {
  opacity: 0.7;
}

.visualizer.is-seekable {
  cursor: ew-resize;
}

.error {
  margin: 0;
  color: #b91c1c;
}

@media (max-width: 640px) {
  .compact-shell {
    gap: 8px;
    padding: 8px;
  }

  .timer {
    font-size: 24px;
    min-width: 40px;
  }

  .icon-btn {
    width: 38px;
    height: 38px;
  }
}
</style>
