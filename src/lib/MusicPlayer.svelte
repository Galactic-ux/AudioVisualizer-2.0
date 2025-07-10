
<script>
  import { createEventDispatcher, onMount } from 'svelte';
  import { slide, fade } from 'svelte/transition';

  const dispatch = createEventDispatcher();

  let files = [];
  let currentSong = null;
  let audio;
  let isPlaying = false;
  let currentTime = 0;
  let duration = 0;
  let fileInput;
  let dropdownOpen = false;
  let isFullscreen = false;

  function toggleFullscreen() {
    const elem = document.documentElement;
    if (!document.fullscreenElement) {
      elem.requestFullscreen?.().then(() => {
        isFullscreen = true;
      }).catch(err => {
        console.warn("Fullscreen failed:", err);
      });
    } else {
      document.exitFullscreen?.().then(() => {
        isFullscreen = false;
      });
    }
  }

  function handleFileUpload(event) {
    const uploadedFiles = Array.from(event.target.files);
    const newSongs = uploadedFiles.map(file => ({
      name: file.name,
      url: URL.createObjectURL(file),
      file
    }));
    files = [...files, ...newSongs];
    if (!currentSong && files.length > 0) {
      playSong(files[0]);
    }
  }

  function triggerFileUpload() {
    fileInput.click();
  }

  function toggleDropdown() {
    dropdownOpen = !dropdownOpen;
  }

  function selectSong(index) {
    playSong(files[index]);
    dropdownOpen = false;
  }

  function playSong(song) {
    if (!audio) return;

    if (currentSong?.url !== song.url) {
      currentSong = song;
      audio.src = song.url;
      dispatch('songChange', { song, audio });
    }

    const playAudio = () => {
      audio.play().then(() => {
        isPlaying = true;
      }).catch(err => {
        console.warn("Play failed", err);
        isPlaying = false;
      });
    };

    if (window.audioContext && window.audioContext.state === 'suspended') {
      window.audioContext.resume().then(() => {
        playAudio();
      }).catch(console.error);
    } else {
      playAudio();
    }
  }

  function togglePlayPause() {
    if (!audio || !currentSong) return;

    if (isPlaying) {
      audio.pause();
      isPlaying = false;
    } else {
      const playAudio = () => {
        audio.play().then(() => {
          isPlaying = true;
        }).catch(err => {
          console.warn("Play failed", err);
          isPlaying = false;
        });
      };

      if (window.audioContext && window.audioContext.state === 'suspended') {
        window.audioContext.resume().then(() => {
          playAudio();
        }).catch(console.error);
      } else {
        playAudio();
      }
    }
  }

  function handleSeek(event) {
    if (!audio) return;
    audio.currentTime = event.target.value;
  }

  function formatTime(time) {
    const mins = Math.floor(time / 60);
    const secs = Math.floor(time % 60).toString().padStart(2, '0');
    return `${mins}:${secs}`;
  }

  onMount(() => {
    audio.addEventListener('timeupdate', () => {
      currentTime = audio.currentTime;
      duration = audio.duration || 0;
    });
  });
</script>

<!-- Upload Button -->
<div id="upload-container">
  <input type="file" accept="audio/*" multiple bind:this={fileInput} on:change={handleFileUpload} id="file-input" />
  <label for="file-input" class="custom-upload-btn"><span>Upload Music</span></label>
</div>

<!-- Playlist Dropdown -->
{#if files.length > 0}
  <div id="playlist-container">
    <button id="current-song" type="button" aria-haspopup="listbox" aria-expanded={dropdownOpen} on:click={toggleDropdown}>
      <div id="song-display">
        <span id="song-title-display">{currentSong ? currentSong.name : 'Select a Song'}</span>
      </div>
      <span id="dropdown-button" aria-hidden="true">▼</span>
    </button>
    {#if dropdownOpen}
      <div id="song-dropdown">
        {#each files as song, i}
          <button type="button" class="song-option" on:click={() => selectSong(i)}>{song.name}</button>
        {/each}
      </div>
    {/if}
  </div>
{/if}

<!-- Player Controls with transition -->
{#if currentSong}
  <div id="animated-controls" transition:slide|fade={{ duration: 300 }}>
    <button class="control-btn" on:click={togglePlayPause}>{isPlaying ? '❚❚' : '▶'}</button>
    <input type="range" min="0" max={duration} value={currentTime} on:input={handleSeek} />
    <div id="time-display">{formatTime(currentTime)} / {formatTime(duration)}</div>
    <button class="control-btn" on:click={toggleFullscreen} title="Toggle Fullscreen">{isFullscreen ? '🡼' : '⛶'}</button>
  </div>
{/if}

<audio bind:this={audio}></audio>


<style>
  #upload-container {
    position: fixed;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 100;
    backdrop-filter: blur(10px);
    background: rgba(255, 255, 255, 0.1);
    border-radius: 30px;
    padding: 8px 20px;
    transition: background 0.3s;
  }

  #upload-container:hover {
    background: rgba(255, 255, 255, 0.2);
  }

  #file-input {
    display: none;
  }

  .custom-upload-btn {
    color: white;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 14px;
    padding: 8px 20px;
  }

  #playlist-container {
    position: fixed;
    top: 20px;
    left: 20px;
    z-index: 100;
    backdrop-filter: blur(10px);
    background: rgba(255, 255, 255, 0.1);
    border-radius: 15px;
    max-width: 300px;
    min-width: 200px;
    transition: background 0.3s;
  }

  #playlist-container:hover {
    background: rgba(255, 255, 255, 0.2);
  }

  #current-song {
    all: unset;
    display: flex;
    align-items: center;
    padding: 10px 15px;
    width: 90%;
    cursor: pointer;
    color: white;
    text-align: left;
    background: none;
    backdrop-filter: blur(10px);
  }

  #song-display {
    flex-grow: 1;
    overflow: hidden;
    white-space: nowrap;
    position: relative;
    padding-right: 10px;
  }

  #dropdown-button {
    background: none;
    border: none;
    color: white;
    font-size: 18px;
    cursor: pointer;
    padding: 0 5px;
  }

  #song-dropdown {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    background: rgba(0, 0, 0, 0.8);
    border-radius: 0 0 15px 15px;
    max-height: 200px;
    overflow-y: auto;
    backdrop-filter: blur(10px);
  }

  .song-option {
    all: unset;
    display: block;
    width: 100%;
    text-align: left;
    padding: 10px 15px;
    color: white;
    cursor: pointer;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .song-option:hover {
    background: rgba(255, 255, 255, 0.1);
  }

  #animated-controls {
    position: fixed;
    bottom: 30px;
    left: 0;
    right: 0;
    margin: 0 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 15px;
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    padding: 15px 30px;
    border-radius: 10px;
    z-index: 100;
    transition: all 0.5s ease-in-out;
    max-width: unset;
  }

  .control-btn {
    height: 30px;
    width: 35px;
    background: none;
    border: none;
    color: white;
    font-size: 24px;
    cursor: pointer;
    border-radius: 30%;
    transition: all 0.3s;
  }

  .control-btn:hover {
    background: rgba(255, 255, 255, 0.1);
  }

  input[type="range"] {
    width: 200px;
    height: 4px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 2px;
  }

  input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 12px;
    height: 12px;
    background: white;
    border-radius: 50%;
    cursor: pointer;
  }

  #time-display {
    color: white;
    font-size: 14px;
    min-width: 100px;
    text-align: center;
  }
</style>
