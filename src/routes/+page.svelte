<script>
  import MusicPlayer from '$lib/MusicPlayer.svelte';
  import Visualizer from '$lib/Visualizer.svelte';

  let currentSong = null;
  let audioEl = null;
  let mode = 'videoWaveform'; // Default mode set to 'videoWaveform'
</script>

<select bind:value={mode} style="position: fixed; top: 20px; right: 20px; z-index: 100;">
  <option value="videoWaveform">Waveform</option>
</select>

<MusicPlayer
  on:songChange={(e) => {
    currentSong = e.detail.song;
    audioEl = e.detail.audio;
  }}
/>

{#if audioEl}
  <Visualizer {audioEl} {mode} />
{/if}

<style>
  :global(body) {
     margin: 0;
      padding: 0;
      background: black;
      overflow: hidden;
      font-family: 'Arial', sans-serif;

  }
</style>