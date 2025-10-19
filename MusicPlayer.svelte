<script>
  import { onMount, onDestroy, tick } from 'svelte';

  export let audioList = [];
  let audio;
  let currentIndex = 0;
  let playing = false;
  let duration = 0;
  let current = 0;
  let timer;
  let loadError = false;

  let offset = 0;
  let dir = 1;
  let scrollDistance = 0;
  let scrollInterval;
  let titleEl;
  let containerEl;
  
  let lyrics = [];
  let currentLyric = '';
  let lyricLoading = false;
  let lyricError = false;

  function formatTime(t) {
    const m = Math.floor(t / 60).toString().padStart(2, '0');
    const s = Math.floor(t % 60).toString().padStart(2, '0');
    return `${m}:${s}`;
  }

  function togglePlay() {
    if (!audio || loadError) return;
    if (playing) {
      audio.pause();
    } else {
      audio.play().catch(err => {
        document.dispatchEvent(new CustomEvent('toast', {detail: {message: '播放失败！'}}));
        playing = false;
      });
    }
    playing = !playing;
  }

  // 解析LRC歌词文件
  function parseLRC(lrcText) {
    const lyricArray = [];
    const timeRegex = /\[(\d{2}):(\d{2}\.\d{2})\]/g;
    const lines = lrcText.split('\n');
    
    lines.forEach(line => {
      const times = [];
      let match;
      while ((match = timeRegex.exec(line)) !== null) {
        const minutes = parseInt(match[1], 10);
        const seconds = parseFloat(match[2]);
        const timeInSeconds = minutes * 60 + seconds;
        times.push(timeInSeconds);
      }
      
      const text = line.replace(/\[(\d{2}):(\d{2}\.\d{2})\]/g, '').trim();
      
      if (times.length > 0 && text) {
        times.forEach(time => {
          lyricArray.push({ time, text });
        });
      }
    });
    
    lyricArray.sort((a, b) => a.time - b.time);
    return lyricArray;
  }
  
  // 加载歌词
  async function loadLyrics() {
    const track = audioList[currentIndex];
    if (!track || !track.lrc) {
      lyrics = [];
      currentLyric = '';
      return;
    }
    
    lyricLoading = true;
    lyricError = false;
    
    try {
      const response = await fetch(track.lrc);
      if (!response.ok) throw new Error('Failed to load lyrics');
      const text = await response.text();
      lyrics = parseLRC(text);
      currentLyric = '';
    } catch (error) {
      console.error('Error loading lyrics:', error);
      lyricError = true;
      lyrics = [];
      currentLyric = '';
    } finally {
      lyricLoading = false;
    }
  }
  
  // 更新当前歌词
  function updateCurrentLyric() {
    if (lyrics.length === 0) return;
    
    for (let i = 0; i < lyrics.length; i++) {
      if (current >= lyrics[i].time && (i === lyrics.length - 1 || current < lyrics[i + 1].time)) {
        if (currentLyric !== lyrics[i].text) {
          currentLyric = lyrics[i].text;
        }
        break;
      }
    }
  }

  function playCurrent() {
    const track = audioList[currentIndex];
    if (!track || !audio) return;
    loadError = false;
    audio.src = track.src;
    playing = false;
    setupScroll();
    loadLyrics();
  }

  function prev() {
    if (audioList.length === 0) return;
    currentIndex = (currentIndex - 1 + audioList.length) % audioList.length;
    playCurrent();
  }

  function next() {
    if (audioList.length === 0) return;
    currentIndex = (currentIndex + 1) % audioList.length;
    playCurrent();
  }

  function seek(e) {
    if (audio) {
      audio.currentTime = +e.target.value;
      current = audio.currentTime;
    }
  }

  onMount(async () => {
    audio = new Audio();
    audio.loop = false;
    audio.addEventListener('loadedmetadata', () => {
      duration = audio.duration;
      loadError = false;
    });
    audio.addEventListener('timeupdate', () => {
      current = audio.currentTime;
      updateCurrentLyric();
    });
    audio.addEventListener('ended', () => next());
    audio.addEventListener('error', (err) => {
      document.dispatchEvent(new CustomEvent('toast', {detail: {message: '音频加载失败！'}}));
      loadError = true;
      playing = false;
    });
    audio.addEventListener('play', () => {
      if (loadError) {
        audio.pause();
        playing = false;
      }
    });

    playCurrent();
    timer = setInterval(() => { 
      if (audio) {
        current = audio.currentTime;
        updateCurrentLyric();
      }
    }, 200);

    await tick();
    setupScroll();
  });

  onDestroy(() => {
    clearInterval(timer);
    clearInterval(scrollInterval);
    if (audio) {
      audio.pause();
      audio.src = '';
      audio = null;
    }
  });

  async function setupScroll() {
    clearInterval(scrollInterval);
    offset = 0;
    dir = 1;
    await tick();
    if (titleEl && containerEl) {
      const sw = titleEl.scrollWidth;
      const cw = containerEl.clientWidth;
      scrollDistance = sw - cw;
      if (scrollDistance > 0) {
        scrollInterval = setInterval(() => {
          offset += dir;
          if (offset >= scrollDistance || offset <= 0) dir = -dir;
        }, 100);
      }
    }
  }
</script>

<div class="player">
  <div class="cover-and-controls">
    <!-- 显示当前播放的歌曲封面 -->
    <button
      type="button"
      class="cover-button"
      aria-label={audioList[currentIndex]?.name ?? 'Album Cover'}
    >
    <!-- 可加入封面点击事件 on:click={togglePlay}
    和封面旋转 class:spinning={playing}  -->

      {#if audioList[currentIndex]?.cover}
        <img
          src={audioList[currentIndex]?.cover}
          alt=""
          class="cover"
        />
      {:else}
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor" class="cover">
          <path d="M13 9.17071C12.6872 9.06015 12.3506 9 12 9C10.3431 9 9 10.3431 9 12C9 13.6569 10.3431 15 12 15C13.6569 15 15 13.6569 15 12V2.4578C19.0571 3.73207 22 7.52236 22 12C22 17.5228 17.5228 22 12 22C6.47715 22 2 17.5228 2 12C2 6.47715 6.47715 2 12 2C12.3375 2 12.6711 2.01672 13 2.04938V9.17071Z"></path>
        </svg>
      {/if}
    </button>

    <!-- 播放控制按钮 -->
    <div class="controls">
      <button on:click={prev} aria-label="Last" class="ctrl">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor"><path d="M8 11.3333L18.2227 4.51823C18.4524 4.36506 18.7628 4.42714 18.916 4.65691C18.9708 4.73904 19 4.83555 19 4.93426V19.0657C19 19.3419 18.7761 19.5657 18.5 19.5657C18.4013 19.5657 18.3048 19.5365 18.2227 19.4818L8 12.6667V19C8 19.5523 7.55228 20 7 20C6.44772 20 6 19.5523 6 19V5C6 4.44772 6.44772 4 7 4C7.55228 4 8 4.44772 8 5V11.3333Z"></path></svg>
      </button>
      <button on:click={togglePlay} aria-label="Play/Pause" class="ctrl play-pause">
        {#if playing}
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor"><path d="M6 5H8V19H6V5ZM16 5H18V19H16V5Z"></path></svg>
        {:else}
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor"><path d="M19.376 12.4161L8.77735 19.4818C8.54759 19.635 8.23715 19.5729 8.08397 19.3432C8.02922 19.261 8 19.1645 8 19.0658V4.93433C8 4.65818 8.22386 4.43433 8.5 4.43433C8.59871 4.43433 8.69522 4.46355 8.77735 4.5183L19.376 11.584C19.6057 11.7372 19.6678 12.0477 19.5146 12.2774C19.478 12.3323 19.4309 12.3795 19.376 12.4161Z"></path></svg>
        {/if}
      </button>
      <button on:click={next} aria-label="Next" class="ctrl">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor"><path d="M16 12.6667L5.77735 19.4818C5.54759 19.6349 5.23715 19.5729 5.08397 19.3431C5.02922 19.261 5 19.1645 5 19.0657V4.93426C5 4.65812 5.22386 4.43426 5.5 4.43426C5.59871 4.43426 5.69522 4.46348 5.77735 4.51823L16 11.3333V5C16 4.44772 16.4477 4 17 4C17.5523 4 18 4.44772 18 5V19C18 19.5523 17.5523 20 17 20C16.4477 20 16 19.5523 16 19V12.6667Z"></path></svg>
      </button>
    </div>
  </div>

  <div class="info">
    <!-- 显示当前播放的歌曲名称 -->
    <div bind:this={containerEl} class="title-container">
      <div
        bind:this={titleEl}
        class="title"
        style="transform: translateX(-{offset}px);"
      >
        {audioList[currentIndex]?.name}
      </div>
    </div>
    
    <!-- 歌词显示区域 -->
    <div class="lyric-container">
      {#if lyricLoading}
        <span class="lyric-placeholder">加载歌词中...</span>
      {:else if lyricError}
        <span class="lyric-placeholder">歌词加载失败</span>
      {:else if currentLyric}
        <div class="current-lyric">{currentLyric}</div>
      {:else if lyrics.length === 0 && audioList[currentIndex]?.lrc}
        <span class="lyric-placeholder">暂无歌词</span>
      {/if}
    </div>

    <!-- 显示播放进度条 -->
    <div class="time">{formatTime(current)} / {formatTime(duration)}</div>

    <input
      type="range"
      min="0"
      max={duration}
      step="0.01"
      bind:value={current}
      on:input={seek}
      class="progress"
    />
  </div>
</div>

<style>
  /* Light mode */
  .player {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem;
    box-sizing: border-box;
    width: 100%;
    color: #333;
    border: 1px solid rgba(0, 0, 0, 0.1);
    background: rgba(255, 255, 255, 0.8);
    backdrop-filter: blur(10px);
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  }

  /* Dark mode via Tailwind's .dark class */
  :global(html.dark) .player {
    color: #eee;
    border: 1px solid rgba(255, 255, 255, 0.1);
    background: rgba(30, 30, 30, 0.8);
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
  }
  :global(html.dark) .time { color: #aaa; }
  :global(html.dark) .progress { background: #555; }
  :global(html.dark) .ctrl { background: #444; }

  /* 控制按钮和封面 */
  .cover-and-controls {
    display: flex;
    align-items: center;
    gap: 0.25rem;
    height: 100%;
    flex-direction: column;
  }
  .cover {
    width: 100px;
    height: 100px;
    justify-items: center;
    /* border-radius: 50%; */
    object-fit: cover;
    cursor: pointer;
    flex-shrink: 0;
    display: block;
  }
  .cover-button {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    margin: 0;
    background: none;
    border: none;
  }
  
  .controls {
    display: flex;
    align-items: center;
    gap: 0.3rem;
    flex-shrink: 0;
    height: 100%;
  }

  .ctrl {
    background: #eee;
    width: 28px;
    height: 28px;
    border: none;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    padding: 0;
  }
  .ctrl svg { width: 18px; height: 18px; fill: currentColor; }

  /* 封面旋转动画 */
  /* .spinning { animation: spin 10s linear infinite; }
  @keyframes spin { to { transform: rotate(360deg); } } */

  /* 歌曲信息和歌词 */
  .info {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    gap: 0.5rem;
  }
  .title-container {
    overflow: hidden;
    width: 100%;
    margin-top: 0;
    margin-bottom: auto;
  }
  .title {
    display: inline-block;
    white-space: nowrap;
  }
  
  .lyric-container {
    text-align: center;
    min-height: 1.2rem;
    margin: 0.25rem 0;
    font-size: 0.85rem;
    color: inherit;
    overflow: hidden;
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .current-lyric {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .lyric-placeholder {
    color: #888;
    font-size: 0.8rem;
  }
  .time {
    font-size: 0.75rem;
    color: #666;
  }
  .progress {
    background: #ddd;
    width: 100%;
    height: 4px;
    border-radius: 2px;
    appearance: none;
    cursor: pointer;
  }
  .progress::-webkit-slider-thumb {
    appearance: none;
    width: 12px;
    height: 12px;
    background: currentColor;
    border-radius: 50%;
    cursor: pointer;
  }
</style>