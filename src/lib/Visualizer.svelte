<script>
  import { onMount, onDestroy } from 'svelte';

  export let audioEl = null;
  export let mode = 'videoWaveform'; 

  let canvasRef;
  let analyser;
  let ctx;
  let animationFrame;
  let dataArray;
  let audioCtx; 

  // --- GENERAL CONFIGURATION ---
  const fftSize = 2048; 
  const NUM_POINTS = 200; // Fewer points for a smoother line
  const SMOOTHING_FACTOR = 0.1; // Smoothing for the base wave's animation
  
  // --- PEAK ANIMATION CONFIGURATION ---
  // Controls how high the beat-synced peaks jump.
  const SUPER_PEAK_AMPLITUDE = 500; 
  // Controls the rise speed of peaks. (0.0 to 1.0). Lower = slower rise.
  const PEAK_RISE_SPEED = 0.1; 
  // Controls the decay speed of peaks. (0.0 to 1.0). Higher = slower decay.
  const PEAK_DECAY_SPEED = 0.8;
  // How wide the super peaks are. Higher number = narrower peak.
  const PEAK_FALLOFF = 1000; 
  // Beat detection sensitivity. Higher number = needs a louder sound to trigger.
  const BEAT_DETECTION_THRESHOLD = 1.2; 

  // --- State Variables ---
  let smoothedMagnitudes = new Array(NUM_POINTS).fill(0);
  let smoothedPeakBoosts = new Array(NUM_POINTS).fill(0); // For the smooth rise
  let beatPeaks = []; // Stores active super peaks: { position, intensity }
  let lastAvgMagnitude = 0; // For beat detection

  function drawVideoWaveform(canvasCtx, width, height) {
    if (!analyser) return;

    analyser.getByteTimeDomainData(dataArray);

    // --- Beat Detection & Peak Creation ---
    let totalMagnitude = 0;
    let maxMagnitude = 0;
    let peakIndex = 0;

    for (let i = 0; i < dataArray.length; i++) {
        const mag = Math.abs(dataArray[i] - 128);
        totalMagnitude += mag;
        if (mag > maxMagnitude) {
            maxMagnitude = mag;
            peakIndex = i;
        }
    }
    const avgMagnitude = totalMagnitude / dataArray.length;

    if (avgMagnitude > lastAvgMagnitude * BEAT_DETECTION_THRESHOLD && avgMagnitude > 5) {
        const peakPosition = peakIndex / dataArray.length; // Position from 0.0 to 1.0
        beatPeaks.push({ position: peakPosition, intensity: 1.0 });
    }
    lastAvgMagnitude = avgMagnitude;

    // --- Canvas Drawing ---
    canvasCtx.fillStyle = 'black';
    canvasCtx.fillRect(0, 0, width, height);

    const centerY = height / 2;
    const baseAmplitude = height / 8; // Reduced base amplitude
    
    let points_upper = [];
    let points_lower = [];

    // --- Calculate Points for Waves ---
    for (let i = 0; i < NUM_POINTS; i++) {
        // --- Base wave smoothing ---
        const sampleIndex = Math.floor(i * (dataArray.length / NUM_POINTS));
        const rawMagnitude = Math.abs(dataArray[sampleIndex] - 128);
        const targetMagnitude = (rawMagnitude / 90) * baseAmplitude;
        smoothedMagnitudes[i] += (targetMagnitude - smoothedMagnitudes[i]) * SMOOTHING_FACTOR;
        
        // --- Super peak smoothing (for rise animation) ---
        const currentPointPosition = i / (NUM_POINTS - 1);
        let targetSuperPeakBoost = 0;
        for (const peak of beatPeaks) {
            const distance = Math.abs(currentPointPosition - peak.position);
            const boost = peak.intensity * Math.exp(-(distance * distance * PEAK_FALLOFF));
            if (boost > targetSuperPeakBoost) {
                targetSuperPeakBoost = boost;
            }
        }
        
        // Use interpolation to smoothly approach the target boost, creating the rise animation
        smoothedPeakBoosts[i] += (targetSuperPeakBoost - smoothedPeakBoosts[i]) * PEAK_RISE_SPEED;
        
        const finalMagnitude = smoothedMagnitudes[i] + (smoothedPeakBoosts[i] * SUPER_PEAK_AMPLITUDE);
        
        const x = (i / (NUM_POINTS - 1)) * width;
        points_upper.push({ x: x, y: centerY - finalMagnitude });
        points_lower.push({ x: x, y: centerY + finalMagnitude });
    }

    // --- Draw the Filled Shape and Outlines ---
    canvasCtx.beginPath();
    // 1. Move to the first point of the upper wave
    canvasCtx.moveTo(points_upper[0].x, points_upper[0].y);

    // 2. Draw the upper wave forwards
    for (let i = 1; i < points_upper.length - 1; i++) {
        const xc = (points_upper[i].x + points_upper[i + 1].x) / 2;
        const yc = (points_upper[i].y + points_upper[i + 1].y) / 2;
        canvasCtx.quadraticCurveTo(points_upper[i].x, points_upper[i].y, xc, yc);
    }
    canvasCtx.lineTo(points_upper[points_upper.length-1].x, points_upper[points_upper.length-1].y);

    // 3. Draw a line to the start of the lower wave (at the end)
    canvasCtx.lineTo(points_lower[points_lower.length-1].x, points_lower[points_lower.length-1].y);

    // 4. Draw the lower wave backwards
    for (let i = points_lower.length - 2; i > 0; i--) {
        const xc = (points_lower[i].x + points_lower[i - 1].x) / 2;
        const yc = (points_lower[i].y + points_lower[i - 1].y) / 2;
        canvasCtx.quadraticCurveTo(points_lower[i].x, points_lower[i].y, xc, yc);
    }
    canvasCtx.lineTo(points_lower[0].x, points_lower[0].y);

    // 5. Close the path to connect the end of the lower wave to the start of the upper wave
    canvasCtx.closePath();

    // 6. Fill the created shape
    canvasCtx.fillStyle = "rgba(255, 255, 255, 0.1)";
    canvasCtx.fill();

    // 7. Stroke the outline on top of the fill
    canvasCtx.lineWidth = 2;
    canvasCtx.strokeStyle = 'rgba(255, 255, 255, 0.9)';
    canvasCtx.stroke();
    
    // Draw the central, solid white line over everything else
    //canvasCtx.beginPath();
    //canvasCtx.lineWidth = 3;
    //canvasCtx.strokeStyle = 'white';
    //canvasCtx.moveTo(0, centerY);
    //canvasCtx.lineTo(width, centerY);
    //canvasCtx.stroke();
  }

  function animate() {
    // Update and decay super peaks' intensity
    beatPeaks.forEach(peak => peak.intensity *= PEAK_DECAY_SPEED);
    beatPeaks = beatPeaks.filter(peak => peak.intensity > 0.01);

    if (ctx) {
      drawVideoWaveform(ctx, canvasRef.width, canvasRef.height);
    }
    animationFrame = requestAnimationFrame(animate);
  }

  onMount(() => {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)(); 
    ctx = canvasRef.getContext('2d');

    if (audioEl) {
        const source = audioCtx.createMediaElementSource(audioEl);
        analyser = audioCtx.createAnalyser();
        analyser.fftSize = fftSize;
        analyser.smoothingTimeConstant = 0.8; 
        dataArray = new Uint8Array(analyser.fftSize); 

        source.connect(analyser);
        analyser.connect(audioCtx.destination);
    } else {
        console.warn("Audio element not provided to visualizer.");
    }

    const resumeContext = () => {
      if (audioCtx.state === 'suspended') {
        audioCtx.resume();
      }
      window.removeEventListener('click', resumeContext); 
      window.removeEventListener('keydown', resumeContext); 
    };
    window.addEventListener('click', resumeContext);
    window.addEventListener('keydown', resumeContext);

    animate();
  });

  onDestroy(() => {
    cancelAnimationFrame(animationFrame);
    if (audioCtx && audioCtx.state !== 'closed') {
      audioCtx.close();
    }
  });
</script>

<canvas
  bind:this={canvasRef}
  width={window.innerWidth}
  height={window.innerHeight}
  style="position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: 0; pointer-events: none;"
></canvas>