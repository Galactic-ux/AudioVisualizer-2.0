<script>
  import { onMount, onDestroy } from 'svelte';

  export let audioEl = null;
  export let mode = 'waveform';
  export let albumArt = null;

  let canvasRef;
  let analyser;
  let ctx;
  let animationFrame;
  let audioCtx;
  
  let frequencyData;
  let timeDomainData;

  let albumArtImage = new Image();
  let isArtLoaded = false;
  
  // NEW: Decoupled scaling for album art to make it more static
  let albumArtPulseScale = 1; 
  const BASE_ALBUM_ART_SIZE = 140; // Diameter of 280px

  $: if (albumArt && albumArt !== albumArtImage.src) {
    isArtLoaded = false;
    albumArtImage.src = albumArt;
    albumArtImage.onload = () => { isArtLoaded = true; };
  }
  
  let currentGradient = [], targetGradient = [], gradientAngle = 0;
  const linNum = 80;
  let circles = [];
  let globalRotation = 0;

  const NUM_POINTS = 200, // Fewer points for a smoother line
  SMOOTHING_FACTOR = 0.1, // Smoothing for the base wave's animation

  // Controls how high the beat-synced peaks jump.
  SUPER_PEAK_AMPLITUDE = 500;
  // Controls the rise speed of peaks. (0.0 to 1.0). Lower = slower rise.
  const PEAK_RISE_SPEED = 0.1, 
  // Controls the decay speed of peaks. (0.0 to 1.0). Higher = slower decay.
  PEAK_DECAY_SPEED = 0.8, 
  // How wide the super peaks are. Higher number = narrower peak.
  PEAK_FALLOFF = 1000, 
  // Beat detection sensitivity. Higher number = needs a louder sound to trigger.
  BEAT_DETECTION_THRESHOLD = 1.2;

  let smoothedMagnitudes = new Array(NUM_POINTS).fill(0), smoothedPeakBoosts = new Array(NUM_POINTS).fill(0);
  let beatPeaks = [], lastAvgMagnitude = 0;
  const colorPalettes = [
      // Enhanced Midnight series
      [[20, 30, 50], [35, 50, 75], [50, 70, 100]],    // Vivid blue
      // Crimson Night
      [[50, 15, 25], [80, 25, 35], [110, 40, 50]],    // Rich reds
      // Abyssal Glow
      [[15, 35, 45], [25, 55, 70], [40, 75, 90]],    // Luminous teal
      // Twilight Pulse
      [[35, 20, 50], [60, 35, 80], [85, 50, 110]],   // Electric purple
      // Forest Aurora
      [[25, 40, 30], [45, 60, 45], [65, 80, 60]],    // Glowing green
      // Volcanic Flame
      [[40, 20, 15], [70, 35, 25], [100, 50, 35]],   // Intense oranges
      // Mystic Haze
      [[30, 35, 45], [50, 55, 65], [70, 75, 85]],    // Frosted blues
      // Nebula Core
      [[20, 25, 40], [35, 40, 65], [50, 55, 90]],    // Cosmic purple
      // Cyber Neon
      [[30, 40, 50], [50, 70, 90], [70, 100, 130]],  // Cyan accents
      // Amber Wave
      [[40, 30, 20], [65, 50, 35], [90, 70, 50]],    // Golden tones
      // Bio Luminescence
      [[15, 40, 35], [25, 60, 50], [35, 80, 65]],    // Alien green
      // Deep Magma
      [[45, 15, 25], [70, 25, 35], [95, 35, 45]],   // Blood orange
      // Arctic Dawn
      [[25, 35, 50], [40, 50, 70], [55, 65, 90]],    // Ice blue
      // Poison Mist
      [[30, 40, 20], [50, 60, 35], [70, 80, 50]],   // Toxic green
      // Royal Veil
      [[40, 20, 50], [60, 35, 75], [80, 50, 100]],   // Regal purple
      // Sunset Horizon
      [[50, 30, 40], [75, 50, 60], [100, 70, 80]],    // Pinkish hues
      [[10, 20, 60], [25, 35, 90], [40, 50, 130]],    // Neon azure
      // Scarlet Inferno
      [[60, 10, 20], [95, 20, 30], [130, 30, 40]],    // Fire reds
      // Abyssal Surge
      [[10, 40, 50], [20, 60, 80], [30, 85, 110]],    // Intense teal
      // Neon Pulse
      [[45, 15, 60], [70, 25, 90], [100, 35, 130]],   // Electric violet
      // Jungle Shock
      [[20, 50, 25], [35, 75, 40], [50, 100, 55]],    // Acid green
      // Lava Core
      [[50, 25, 10], [85, 40, 20], [120, 55, 30]],    // Flaming orange
      // Crystal Haze
      [[20, 40, 50], [35, 60, 80], [50, 85, 110]],    // Arctic cyan
      // Galactic Core
      [[25, 20, 50], [40, 30, 80], [55, 40, 110]],    // Deep cosmos
      // Cyber Glow
      [[20, 50, 60], [40, 80, 110], [60, 110, 150]],  // Neon cyan
      // Solar Flare
      [[50, 35, 15], [80, 55, 25], [110, 75, 35]],    // Liquid gold
      // Toxic Bloom
      [[10, 50, 30], [20, 75, 45], [30, 100, 60]],    // Radioactive green
      // Magma Burst
      [[55, 10, 20], [85, 20, 30], [115, 30, 40]],    // Molten core
      // Polar Glow
      [[20, 30, 60], [35, 45, 85], [50, 60, 110]],    // Glacier blue
      // Venom Rush
      [[40, 50, 15], [60, 75, 25], [80, 100, 35]],    // Poison dart
      // Majestic Veil
      [[50, 15, 60], [75, 25, 90], [100, 35, 120]],   // Royal amethyst
      // Neon Sunset
      [[60, 25, 45], [90, 40, 65], [120, 55, 85]]     // Hot pink

  ];
  
  const lerp = (a, b, t) => a + (b - a) * t;
  const map = (value, s1, st1, s2, st2) => s2 + (st2 - s2) * ((value - s1) / (st1 - s1));
  const random = (min, max) => Math.random() * (max - min) + min;
  
  const getLinearAverages = (data, numBins) => {
    const averages = new Array(numBins).fill(0);
    const binSize = Math.floor(data.length / numBins);
    for (let i = 0; i < numBins; i++) {
        let sum = 0;
        for (let j = 0; j < binSize; j++) sum += data[i * binSize + j];
        averages[i] = sum / binSize;
    }
    return averages;
  };

  function animate() {
    if (!analyser || !ctx || !audioEl || audioEl.paused) {
      if (ctx) ctx.clearRect(0, 0, canvasRef.width, canvasRef.height);
      animationFrame = requestAnimationFrame(animate);
      return;
    }
    analyser.getByteFrequencyData(frequencyData);
    analyser.getByteTimeDomainData(timeDomainData);
    const width = canvasRef.width, height = canvasRef.height;
    
    // NEW: Calculate the subtle pulse for the album art
    const avg = frequencyData.reduce((a, b) => a + b, 0) / frequencyData.length;
    const artTargetScale = 1 + (avg / 800); // Much less sensitive pulse
    albumArtPulseScale = lerp(albumArtPulseScale, artTargetScale, 0.1);

    drawBackground(ctx, width, height);
    if (mode === 'bubbles') {
      drawCircles(ctx, width, height);
    } else if (mode === 'waveform') {
      drawWaveform(ctx, width, height);
    }
    animationFrame = requestAnimationFrame(animate);
  }

  function drawBackground(ctx, width, height) {
    let energy = 0;
    for(let i=0; i<64; i++) energy += frequencyData[i];
    energy /= 64;
    if (energy > 200) {
      const newTarget = colorPalettes[Math.floor(random(0, colorPalettes.length))];
      if (newTarget !== targetGradient) targetGradient = newTarget;
    }
    currentGradient.forEach((col, i) => col.forEach((c, j) => {
        currentGradient[i][j] = lerp(c, targetGradient[i][j], 0.005);
    }));
    gradientAngle += map(energy, 0, 255, 0.2, 2);
    const angle = gradientAngle * (Math.PI / 180);
    ctx.save();
    const x1=Math.cos(angle)*width, y1=Math.sin(angle)*height, x2=width-x1, y2=height-y1;
    const gradient = ctx.createLinearGradient(x1, y1, x2, y2);
    gradient.addColorStop(0, `rgb(${currentGradient[0]})`);
    gradient.addColorStop(0.5, `rgb(${currentGradient[1]})`);
    gradient.addColorStop(1, `rgb(${currentGradient[2]})`);
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, width, height);
    ctx.restore();
  }

  function drawCircles(ctx, width, height) {
    const linAverages = getLinearAverages(frequencyData, linNum);
    globalRotation += 0.3;
    ctx.save();
    ctx.translate(width / 2, height / 2);

    // CHANGED: STEP 1 - Draw the orbiting circles FIRST
    for (let i = 1; i < linNum; i++) { // Start loop at 1 to skip the center
        const circle = circles[i];
        const avg = linAverages[i];
        ctx.save();
        ctx.rotate(globalRotation * circle.rotationSpeed * (Math.PI / 180));
        ctx.strokeStyle = 'white';
        ctx.lineWidth = 3;
        ctx.fillStyle="rgba(255,255,255,0.1)"; ctx.fill();
        ctx.beginPath();
        ctx.arc(circle.x, circle.y, avg * 1.0, 0, 2 * Math.PI);
        ctx.stroke();
        ctx.restore();
    }

    // CHANGED: STEP 2 - Draw the central hub (black circle + album art) LAST
    const centralCircle = circles[0];
    const centralAvg = linAverages[0];
    const blackCircleRadius = centralAvg * 1.5; // The big, pulsing black circle

    // Draw the black circle base
    ctx.fillStyle = 'black';
    ctx.beginPath();
    ctx.arc(centralCircle.x, centralCircle.y, blackCircleRadius, 0, 2 * Math.PI);
    ctx.fill();
    ctx.strokeStyle = 'white'; // Keep the white outline
    ctx.lineWidth = 3;
    ctx.stroke();

    // Draw the album art on top
    if (albumArt && isArtLoaded) {
      const artRadius = BASE_ALBUM_ART_SIZE * albumArtPulseScale; // Use the new, more static size
      ctx.save();
      ctx.beginPath();
      ctx.arc(centralCircle.x, centralCircle.y, artRadius, 0, Math.PI * 2);
      ctx.clip(); // Create a circular clipping mask
      ctx.drawImage(albumArtImage, centralCircle.x - artRadius, centralCircle.y - artRadius, artRadius * 2, artRadius * 2);
      ctx.restore(); // Remove the clipping mask
    }
    
    ctx.restore(); // Restore context after all drawing is done
  }
  
  function drawWaveform(ctx, width, height) {
      // This function remains unchanged.
      let totalMag = 0, maxMag = 0, peakIdx = 0;
      for (let i = 0; i < timeDomainData.length; i++) {
          const mag = Math.abs(timeDomainData[i] - 128);
          totalMag += mag;
          if (mag > maxMag) { maxMag = mag; peakIdx = i; }
      }
      const avgMag = totalMag / timeDomainData.length;
      if (avgMag > lastAvgMagnitude * BEAT_DETECTION_THRESHOLD && avgMag > 5) {
          beatPeaks.push({ position: peakIdx / timeDomainData.length, intensity: 1.0 });
      }
      lastAvgMagnitude = avgMag;
      beatPeaks.forEach(p => p.intensity *= PEAK_DECAY_SPEED);
      beatPeaks = beatPeaks.filter(p => p.intensity > 0.01);
      let points_u = [], points_l = [];
      const baseAmp = height / 8;
      for (let i = 0; i < NUM_POINTS; i++) {
          const sIdx = Math.floor(i * (timeDomainData.length / NUM_POINTS));
          const rawMag = Math.abs((timeDomainData[sIdx] || 128) - 128);
          const targetMag = (rawMag / 128) * baseAmp;
          smoothedMagnitudes[i] += (targetMag - smoothedMagnitudes[i]) * SMOOTHING_FACTOR;
          const pos = i / (NUM_POINTS - 1);
          let targetBoost = 0;
          for (const peak of beatPeaks) {
              const dist = Math.abs(pos - peak.position);
              const boost = peak.intensity * Math.exp(-(dist * dist * PEAK_FALLOFF));
              if (boost > targetBoost) targetBoost = boost;
          }
          smoothedPeakBoosts[i] += (targetBoost - smoothedPeakBoosts[i]) * PEAK_RISE_SPEED;
          const finalMag = smoothedMagnitudes[i] + (smoothedPeakBoosts[i] * SUPER_PEAK_AMPLITUDE);
          const x_pos = map(i, 0, NUM_POINTS - 1, 0, width);
          points_u.push({ x: x_pos, y: height/2 - finalMag });
          points_l.push({ x: x_pos, y: height/2 + finalMag });
      }
      ctx.beginPath();
      ctx.moveTo(points_u[0].x, points_u[0].y);
      for (let i = 1; i < points_u.length - 1; i++) ctx.quadraticCurveTo(points_u[i].x, points_u[i].y, (points_u[i].x+points_u[i+1].x)/2, (points_u[i].y+points_u[i+1].y)/2);
      ctx.lineTo(points_l[points_l.length-1].x, points_l[points_l.length-1].y);
      for (let i=points_l.length-2; i>0; i--) ctx.quadraticCurveTo(points_l[i].x, points_l[i].y, (points_l[i].x+points_l[i-1].x)/2, (points_l[i].y+points_l[i-1].y)/2);
      ctx.closePath();
      ctx.fillStyle="rgba(255,255,255,0.1)"; ctx.fill();
      ctx.lineWidth=2; ctx.strokeStyle='rgba(255,255,255,0.9)'; ctx.stroke();
      //tx.beginPath(); ctx.lineWidth=3; ctx.strokeStyle='white';
      //ctx.moveTo(0, height/2); ctx.lineTo(width, height/2); ctx.stroke();
  }

  onMount(() => {
    ctx = canvasRef.getContext('2d');
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const source = audioCtx.createMediaElementSource(audioEl);
    analyser = audioCtx.createAnalyser();
    analyser.fftSize = 2048;
    source.connect(analyser);
    analyser.connect(audioCtx.destination);
    frequencyData = new Uint8Array(analyser.frequencyBinCount);
    timeDomainData = new Uint8Array(analyser.fftSize);
    currentGradient = colorPalettes[0];
    targetGradient = colorPalettes[1];
    for (let i = 0; i < linNum; i++) {
      circles.push({
        x: i === 0 ? 0 : random(-canvasRef.width, canvasRef.width),
        y: i === 0 ? 0 : random(-canvasRef.height, canvasRef.height),
        rotationSpeed: random(0.5, 1.5)
      });
    }
    animate();
  });

  onDestroy(() => {
    cancelAnimationFrame(animationFrame);
    if (audioCtx && audioCtx.state !== 'closed') audioCtx.close();
  });

</script>

<canvas 
  bind:this={canvasRef} 
  width={window.innerWidth} 
  height={window.innerHeight} 
  style="position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: -1;"
></canvas>

<!-- No <img> tag or styles are needed here anymore -->

<style>
  #album-art {
    position: fixed;
    top: 50%;
    left: 50%;
    width: 280px;
    height: 280px;
    border-radius: 50%;
    object-fit: cover;
    z-index: 2;
    box-shadow: 0 0 40px rgba(255, 255, 255, 0.2);
    transition: transform 0.2s ease;
    transform: translate(-50%, -50%) scale(1);
  }
</style>