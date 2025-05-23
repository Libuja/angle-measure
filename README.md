# angle-measure<!DOCTYPE html>
<html>
<head>
  <title>Shoulder Angle Tracker</title>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/pose@0.4/pose.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils@0.3/camera_utils.js"></script>
  <style>
    html, body { margin: 0; padding: 0; overflow: hidden; font-family: sans-serif; background: black; color: white; }
    canvas, video { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; object-fit: cover; z-index: 0; }
    #angleDisplay {
      position: absolute;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      background: rgba(0,0,0,0.6);
      padding: 15px 30px;
      border-radius: 12px;
      font-size: 24px;
      z-index: 1;
    }
    #controls {
      position: absolute;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      z-index: 1;
    }
    button {
      padding: 10px 20px;
      font-size: 16px;
      margin: 0 10px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <div id="angleDisplay">Angle: <span id="angle">-</span>°</div>
  <div id="controls">
    <button onclick="startTracking()">Start Tracking</button>
    <button onclick="stopTracking()">Stop Tracking</button>
  </div>
  <video id="inputVideo" autoplay muted playsinline></video>
  <canvas id="outputCanvas"></canvas>

  <script>
    const video = document.getElementById('inputVideo');
    const canvas = document.getElementById('outputCanvas');
    const ctx = canvas.getContext('2d');
    const angleSpan = document.getElementById('angle');
    let tracking = false;
    let camera;

    function resizeCanvas() {
      canvas.width = window.innerWidth;
      canvas.height = window.innerHeight;
    }
    window.addEventListener('resize', resizeCanvas);
    resizeCanvas();

    async function initCamera() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'user' } });
        video.srcObject = stream;
      } catch (error) {
        alert("Camera access denied. Please allow camera permissions in your browser settings.");
        console.error("Camera error:", error);
      }
    }

    function calculateAngle(A, B, C) {
      const AB = [B.x - A.x, B.y - A.y];
      const CB = [B.x - C.x, B.y - C.y];
      const dot = AB[0] * CB[0] + AB[1] * CB[1];
      const magAB = Math.hypot(...AB);
      const magCB = Math.hypot(...CB);
      const angle = Math.acos(dot / (magAB * magCB)) * (180 / Math.PI);
      return Math.round(angle);
    }

    const pose = new Pose({ locateFile: file => `https://cdn.jsdelivr.net/npm/@mediapipe/pose@0.4/${file}` });
    pose.setOptions({
      modelComplexity: 1,
      smoothLandmarks: true,
      enableSegmentation: false,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5
    });

    pose.onResults(results => {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.drawImage(results.image, 0, 0, canvas.width, canvas.height);

      if (!tracking || !results.poseLandmarks) return;

      const lm = results.poseLandmarks;
      const leftVisible = lm[11].visibility > 0.5 && lm[13].visibility > 0.5 && lm[23].visibility > 0.5;
      const rightVisible = lm[12].visibility > 0.5 && lm[14].visibility > 0.5 && lm[24].visibility > 0.5;

      let A, B, C;
      if (rightVisible && (!leftVisible || lm[14].visibility > lm[13].visibility)) {
        A = lm[14]; B = lm[12]; C = lm[24];
      } else if (leftVisible) {
        A = lm[13]; B = lm[11]; C = lm[23];
      } else {
        angleSpan.textContent = '-';
        return;
      }

      const angle = calculateAngle(A, B, C);
      angleSpan.textContent = angle;

      ctx.strokeStyle = 'lime';
      ctx.lineWidth = 4;

      ctx.beginPath();
      ctx.moveTo(A.x * canvas.width, A.y * canvas.height);
      ctx.lineTo(B.x * canvas.width, B.y * canvas.height);
      ctx.stroke();

      ctx.beginPath();
      ctx.moveTo(C.x * canvas.width, C.y * canvas.height);
      ctx.lineTo(B.x * canvas.width, B.y * canvas.height);
      ctx.stroke();

      // Draw angle at the midpoint between A and C
      const midX = (A.x + C.x) / 2 * canvas.width;
      const midY = (A.y + C.y) / 2 * canvas.height;
      ctx.fillStyle = 'white';
      ctx.font = '24px sans-serif';
      ctx.fillText(`${angle}°`, midX, midY);
    });

    function startTracking() {
      tracking = true;
      if (!camera) {
        camera = new Camera(video, {
          onFrame: async () => {
            await pose.send({ image: video });
          },
          width: 1280,
          height: 720
        });
        camera.start();
      }
    }

    function stopTracking() {
      tracking = false;
    }

    initCamera();
  </script>
</body>
</html>
