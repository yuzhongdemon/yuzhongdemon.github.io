<script lang="ts">
  import Icon from "@iconify/svelte";
  import { onMount } from "svelte";
  import playlist from "./audio-data";

  let currentTrack = (0);
  let isPlaying = (false);
  let currentTime = (0);
  let duration = (0);
  let volume = (0.7);
  let showVolume = (false);
  let isDragging = (false);
  let loading = (false);
  let playMode = <'sequential' | 'shuffle'>('sequential');
  let showPlaylist = (false);

  let progressBarEl: HTMLElement;
  let playlistPanelEl: HTMLElement;

  let audioCtx: AudioContext | null = null;
  let audioBuffer: AudioBuffer | null = null;
  let gainNode: GainNode | null = null;
  let source: AudioBufferSourceNode | null = null;
  let pauseOffset = 0;
  let playStartTime = 0;
  let animFrameId: number | null = null;
  let ending = false;

  function getAudioContext(): AudioContext {
    if (!audioCtx) {
      audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)();
    }
    return audioCtx;
  }

  function syncCurrentTime() {
    if (!audioBuffer) return;
    if (isPlaying && audioCtx && !isDragging) {
      const elapsed = audioCtx.currentTime - playStartTime;
      currentTime = Math.min(pauseOffset + elapsed, duration);
    }
  }

  function startTimer() {
    stopTimer();
    animFrameId = requestAnimationFrame(function frame() {
      syncCurrentTime();
      animFrameId = requestAnimationFrame(frame);
    });
  }

  function stopTimer() {
    if (animFrameId !== null) {
      cancelAnimationFrame(animFrameId);
      animFrameId = null;
    }
  }

  function stopSource() {
    if (source) {
      try { source.stop(); } catch (_) {}
      source.disconnect();
      source = null;
    }
  }

  function playFrom(offset: number) {
    if (!audioCtx || !audioBuffer || !gainNode) return;
    stopSource();
    const ctx = audioCtx;
    const newSource = ctx.createBufferSource();
    newSource.buffer = audioBuffer;
    newSource.connect(gainNode);
    newSource.start(0, offset);
    newSource.onended = () => {
      if (source !== newSource) return;
      if (ending) return;
      onTrackEnd();
    };
    source = newSource;
    playStartTime = ctx.currentTime;
    ending = false;
    isPlaying = true;
    startTimer();
  }

  function onTrackEnd() {
    pauseOffset = 0;
    currentTime = 0;
    isPlaying = false;
    stopTimer();
    stopSource();
    nextTrack();
  }

  function togglePlay() {
    if (loading) return;
    if (!audioBuffer) {
      loadTrack(currentTrack);
      return;
    }
    if (isPlaying) {
      pause();
    } else {
      resume();
    }
  }

  function pause() {
    if (!audioCtx || !source) return;
    ending = true;
    pauseOffset += audioCtx.currentTime - playStartTime;
    stopSource();
    isPlaying = false;
    stopTimer();
    syncCurrentTime();
  }

  function resume() {
    if (!audioCtx || !audioBuffer) return;
    const ctx = audioCtx;
    if (ctx.state === "suspended") {
      ctx.resume().catch(() => {});
    }
    playFrom(pauseOffset);
  }

  function prevTrack() {
    ending = true;
    stopTimer();
    stopSource();
    const prev = (currentTrack - 1 + playlist.length) % playlist.length;
    switchTrack(prev);
  }

  function nextTrack() {
    ending = true;
    stopTimer();
    stopSource();
    let next: number;
    if (playMode === 'shuffle') {
      if (playlist.length <= 1) { next = 0; }
      else { do { next = Math.floor(Math.random() * playlist.length); } while (next === currentTrack); }
    } else {
      next = (currentTrack + 1) % playlist.length;
    }
    switchTrack(next);
  }

  function switchTrack(index: number) {
    if (index < 0 || index >= playlist.length) return;
    showPlaylist = false;
    pauseOffset = 0;
    currentTime = 0;
    duration = 0;
    audioBuffer = null;
    currentTrack = index;
    loadTrack(index);
  }

  function togglePlayMode() {
    playMode = playMode === 'sequential' ? 'shuffle' : 'sequential';
  }

  function togglePlaylist() {
    showPlaylist = !showPlaylist;
  }

  (() => {
    volume = Math.max(0, Math.min(1, volume));
    if (gainNode) gainNode.gain.value = volume;
  });

  function toggleVolume() {
    showVolume = !showVolume;
  }

  function seekTo(e: MouseEvent) {
    if (!audioBuffer) return;
    const rect = progressBarEl.getBoundingClientRect();
    const ratio = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
    const newTime = ratio * duration;
    pauseOffset = newTime;
    if (isPlaying) {
      ending = true;
      stopSource();
      playFrom(newTime);
    } else {
      currentTime = newTime;
    }
  }

  function handleSeekStart(e: MouseEvent) {
    isDragging = true;
    seekTo(e);
  }

  function handleSeekMove(e: MouseEvent) {
    if (!isDragging || !audioBuffer) return;
    const rect = progressBarEl.getBoundingClientRect();
    const ratio = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
    currentTime = ratio * duration;
  }

  function handleSeekEnd(e: MouseEvent) {
    if (!isDragging) return;
    isDragging = false;
    seekTo(e);
    if (isPlaying) startTimer();
  }

  function formatTime(s: number): string {
    if (!s || isNaN(s) || !isFinite(s)) return "0:00";
    const m = Math.floor(s / 60);
    const sec = Math.floor(s % 60);
    return m + ":" + sec.toString().padStart(2, "0");
  }

  let progressPercent = (
    duration > 0 ? (currentTime / duration) * 100 : 0
  );

  function base64ToArrayBuffer(base64: string): ArrayBuffer {
    const binary = atob(base64);
    const bytes = new Uint8Array(binary.length);
    for (let i = 0; i < binary.length; i++) {
      bytes[i] = binary.charCodeAt(i);
    }
    return bytes.buffer;
  }

  async function loadTrack(index: number) {
    if (loading) return;
    const track = playlist[index];
    if (!track) return;
    loading = true;
    try {
      const ctx = getAudioContext();
      if (ctx.state === "suspended") await ctx.resume();
      const arrayBuffer = base64ToArrayBuffer(track.data);
      audioBuffer = await ctx.decodeAudioData(arrayBuffer);
      if (gainNode) gainNode.disconnect();
      gainNode = ctx.createGain();
      gainNode.gain.value = volume;
      gainNode.connect(ctx.destination);
      duration = audioBuffer.duration;
      pauseOffset = 0;
      currentTime = 0;
      loading = false;
      playFrom(0);
    } catch (e) {
      console.error("Failed to load audio:", e);
      isPlaying = false;
      loading = false;
    }
  }

  function cleanup() {
    stopTimer();
    ending = true;
    stopSource();
    if (gainNode) { gainNode.disconnect(); gainNode = null; }
    if (audioCtx) { audioCtx.close().catch(() => {}); audioCtx = null; }
    audioBuffer = null;
    isPlaying = false;
    pauseOffset = 0;
    currentTime = 0;
    duration = 0;
  }

  function handleClickOutside(e: MouseEvent) {
    if (!showPlaylist) return;
    if (playlistPanelEl && !playlistPanelEl.contains(e.target as Node)) {
      showPlaylist = false;
    }
  }

  onMount(() => {
    loadTrack(currentTrack);
    document.addEventListener("mousemove", handleSeekMove);
    document.addEventListener("mouseup", handleSeekEnd);
    document.addEventListener("click", handleClickOutside);
    return () => {
      document.removeEventListener("mousemove", handleSeekMove);
      document.removeEventListener("mouseup", handleSeekEnd);
      document.removeEventListener("click", handleClickOutside);
      cleanup();
    };
  });
</script>

<div class="card-base p-3 pb-3 onload-animation" role="region" aria-label="播放器">
  <div class="font-bold transition text-base text-neutral-900 dark:text-neutral-100 relative ml-1 mb-3
    before:w-1 before:h-4 before:rounded-md before:bg-[var(--primary)]
    before:absolute before:-left-3 before:top-[2px]">
    播放器
  </div>

  <div class="flex items-center gap-2 mb-2 px-1">
    <div class="w-8 h-8 rounded-lg bg-[var(--btn-regular-bg)] flex items-center justify-center flex-shrink-0 transition">
      <Icon icon="material-symbols:music-note-rounded" class="text-[1.1rem] text-[var(--btn-content)]"></Icon>
    </div>
    <div class="min-w-0 flex-1">
      <div class="text-sm font-medium transition text-neutral-800 dark:text-neutral-200 truncate">
        {playlist[currentTrack]?.title || "???"}
      </div>
      <div class="text-xs transition text-black/50 dark:text-white/50 truncate">
        {playlist[currentTrack]?.artist || ""}
      </div>
    </div>
  </div>

  <div
    bind:this={progressBarEl}
    class="relative h-2 rounded-full bg-[var(--btn-regular-bg)] cursor-pointer mb-1 group transition"
    onmousedown={handleSeekStart}
    role="slider"
    aria-label="????"
    aria-valuemin={0}
    aria-valuemax={100}
    aria-valuenow={Math.round(progressPercent)}
  >
    <div
      class="absolute h-full rounded-full transition-[width] duration-100"
      class:bg-[var(--primary)]={isPlaying || isDragging}
      class:bg-[var(--btn-content)]={!isPlaying && !isDragging}
      style="width: {progressPercent}%"
    ></div>
    <div
      class="absolute top-1/2 -translate-y-1/2 w-3 h-3 rounded-full bg-white shadow transition-opacity opacity-0 group-hover:opacity-100"
      style="left: calc({progressPercent}% - 6px)"
    ></div>
  </div>

  <div class="flex items-center justify-between px-0.5">
    <span class="text-xs transition text-black/40 dark:text-white/40 tabular-nums">
      {formatTime(currentTime)}
    </span>

    <div class="flex items-center gap-1">
      <div class="relative">
        <button onclick={toggleVolume}
          class="btn-plain rounded-lg h-7 w-7 active:scale-90 flex items-center justify-center"
          aria-label="??">
          <Icon
            icon={volume === 0
              ? "material-symbols:volume-mute-rounded"
              : volume < 0.5
                ? "material-symbols:volume-down-rounded"
                : "material-symbols:volume-up-rounded"
            }
            class="text-[1rem]"
          ></Icon>
        </button>
        {#if showVolume}
          <div class="absolute bottom-full right-0 mb-2 p-2 rounded-lg bg-[var(--card-bg)] shadow-xl border border-[var(--line-divider)] transition z-50">
            <input
              type="range"
              min="0"
              max="1"
              step="0.05"
              bind:value={volume}
              class="volume-slider"
              aria-label="音乐播放器"
              style="writing-mode: vertical-lr; direction: rtl; height: 5rem; width: 1.25rem;"
            >
          </div>
        {/if}
      </div>

      <button onclick={togglePlay}
        class="btn-regular rounded-lg h-8 w-8 active:scale-90 flex items-center justify-center transition"
        aria-label={isPlaying ? "??" : "??"}
      >
        <Icon
          icon={isPlaying ? "material-symbols:pause-rounded" : "material-symbols:play-arrow-rounded"}
          class="text-[1.25rem]"
        ></Icon>
      </button>
    </div>

    <span class="text-xs transition text-black/40 dark:text-white/40 tabular-nums">
      {formatTime(duration)}
    </span>
  </div>

  <!-- ???????? + ?? + ?? -->
    <div class="flex items-center justify-center gap-3 px-0.5 mt-2 relative">
      <button onclick={prevTrack}
        class="btn-plain rounded-lg h-7 w-7 active:scale-90 flex items-center justify-center"
        aria-label="???">
        <Icon icon="material-symbols:skip-previous-rounded" class="text-[1rem]"></Icon>
      </button>

      <button onclick={togglePlayMode}
        class="btn-plain rounded-lg h-7 w-7 active:scale-90 flex items-center justify-center"
        class:text-[var(--primary)]={playMode === 'shuffle'}
        aria-label={playMode === 'shuffle' ? "????" : "????"}>
        <Icon icon={playMode === 'shuffle' ? "material-symbols:shuffle-rounded" : "material-symbols:repeat-rounded"}
          class="text-[1rem]"></Icon>
      </button>

      <button onclick={nextTrack}
        class="btn-plain rounded-lg h-7 w-7 active:scale-90 flex items-center justify-center"
        aria-label="???">
        <Icon icon="material-symbols:skip-next-rounded" class="text-[1rem]"></Icon>
      </button>

      <div class="w-px h-4 bg-[var(--line-divider)] mx-1"></div>

      <button onclick={togglePlaylist}
        class="btn-plain rounded-lg h-7 w-7 active:scale-90 flex items-center justify-center"
        class:text-[var(--primary)]={showPlaylist}
        aria-label="??">
        <Icon icon="material-symbols:queue-music-rounded" class="text-[1rem]"></Icon>
      </button>
    </div>

</div>
{#if showPlaylist}
  <div
    bind:this={playlistPanelEl}
    class="playlist-panel"
    style="position: relative; margin-top: -0.5rem; margin-bottom: 0.25rem;"
  >
    <div class="card-base px-2 py-2 rounded-lg border border-[var(--line-divider)] shadow-lg transition max-h-48 overflow-y-auto">
      {#each playlist as track, i}
        <button
          onclick={() => switchTrack(i)}
          class="w-full text-left px-2 py-1.5 rounded-md transition text-sm flex items-center gap-2
            hover:bg-[var(--btn-regular-bg)]"
          class:active-track={i === currentTrack}
          class:text-[var(--primary)]={i === currentTrack}
        >
          {#if i === currentTrack && isPlaying}
            <Icon icon="material-symbols:play-arrow-rounded" class="text-[1rem] flex-shrink-0"></Icon>
          {:else}
            <span class="w-4 flex-shrink-0 text-xs text-black/30 dark:text-white/30">{i + 1}</span>
          {/if}
          <div class="min-w-0 flex-1 truncate">{track.title}</div>
          <div class="text-xs text-black/40 dark:text-white/40 truncate max-w-[5rem]">{track.artist}</div>
        </button>
      {/each}
    </div>
  </div>
{/if}

<style>
  .volume-slider {
    -webkit-appearance: none;
    appearance: none;
    background: var(--btn-regular-bg);
    border-radius: 0.5rem;
    outline: none;
    cursor: pointer;
  }
  .volume-slider::-webkit-slider-runnable-track {
    height: 0.375rem;
    border-radius: 0.25rem;
    background: var(--btn-regular-bg);
  }
  .volume-slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 0.75rem;
    height: 0.75rem;
    border-radius: 50%;
    background: var(--primary);
    box-shadow: 0 1px 3px rgba(0,0,0,0.2);
    margin-top: -3px;
  }
  .volume-slider::-moz-range-track {
    height: 0.375rem;
    border-radius: 0.25rem;
    background: var(--btn-regular-bg);
  }
  .volume-slider::-moz-range-thumb {
    width: 0.75rem;
    height: 0.75rem;
    border-radius: 50%;
    background: var(--primary);
    border: none;
    box-shadow: 0 1px 3px rgba(0,0,0,0.2);
  }

  .active-track {
    background-color: color-mix(in srgb, var(--primary) 10%, transparent);
  }
</style>
