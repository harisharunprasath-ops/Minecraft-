#Minecraft-
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Minecraft-like 3D Mobile Game</title>
    <style>
      html, body {
        margin: 0;
        overflow: hidden;
        background: #87ceeb;
        font-family: sans-serif;
        touch-action: none;
        user-select: none;
      }

      canvas {
        display: block;
        width: 100vw;
        height: 100vh;
      }

      .crosshair {
        position: fixed;
        left: 50%;
        top: 50%;
        width: 22px;
        height: 22px;
        transform: translate(-50%, -50%);
        pointer-events: none;
      }

      .crosshair::before,
      .crosshair::after {
        content: "";
        position: absolute;
        background: rgba(255,255,255,0.9);
        box-shadow: 0 0 4px rgba(0,0,0,0.5);
      }

      .crosshair::before {
        left: 50%;
        top: 0;
        width: 2px;
        height: 100%;
        transform: translateX(-50%);
      }

      .crosshair::after {
        top: 50%;
        left: 0;
        width: 100%;
        height: 2px;
        transform: translateY(-50%);
      }

      #mobileControls {
        position: fixed;
        inset: 0;
        pointer-events: none;
        display: none;
      }

      .joystick {
        position: absolute;
        left: 28px;
        bottom: 28px;
        width: 120px;
        height: 120px;
        border-radius: 50%;
        background: rgba(255,255,255,0.12);
        border: 2px solid rgba(255,255,255,0.22);
        pointer-events: auto;
      }

      .joystick-knob {
        position: absolute;
        width: 52px;
        height: 52px;
        left: 34px;
        top: 34px;
        border-radius: 50%;
        background: rgba(255,255,255,0.75);
        box-shadow: 0 0 12px rgba(0,0,0,0.2);
      }

      .action-btn {
        position: absolute;
        right: 28px;
        width: 72px;
        height: 72px;
        border-radius: 50%;
        border: none;
        background: rgba(15, 23, 42, 0.6);
        color: white;
        font-weight: bold;
        box-shadow: 0 6px 20px rgba(0,0,0,0.25);
        pointer-events: auto;
      }

      #jumpBtn {
        bottom: 120px;
      }

      #breakBtn {
        bottom: 28px;
      }

      #placeBtn {
        bottom: 116px;
        right: 120px;
      }

      #startButton {
        position: fixed;
        left: 50%;
        top: 50%;
        transform: translate(-50%, -50%);
        padding: 16px 26px;
        border: none;
        border-radius: 12px;
        font-size: 20px;
        font-weight: bold;
        background: white;
        color: #0f172a;
        box-shadow: 0 10px 30px rgba(0,0,0,0.2);
      }
    </style>
  </head>
  <body>
    <div class="crosshair"></div>

    <div id="mobileControls">
      <div id="joystick" class="joystick">
        <div id="joystickKnob" class="joystick-knob"></div>
      </div>

      <button id="jumpBtn" class="action-btn">Jump</button>
      <button id="breakBtn" class="action-btn">Break</button>
      <button id="placeBtn" class="action-btn">Place</button>
    </div>

    <button id="startButton">Start Game</button>

    <script type="module">
      import * as THREE from "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js";

      const isMobile = /Mobi|Android|iPhone|iPad/i.test(navigator.userAgent);

      const scene = new THREE.Scene();
      scene.background = new THREE.Color(0x87ceeb);
      scene.fog = new THREE.Fog(0x87ceeb, 12, 80);

      const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
      camera.position.set(0, 10, 18);

      const renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      document.body.appendChild(renderer.domElement);

      const ambientLight = new THREE.AmbientLight(0xffffff, 0.85);
      scene.add(ambientLight);

      const sunLight = new THREE.DirectionalLight(0xffffff, 1.05);
      sunLight.position.set(10, 22, 8);
      scene.add(sunLight);

      camera.rotation.order = "YXZ";

      const world = new Map();
      const blockGeometry = new THREE.BoxGeometry(1, 1, 1);

      function keyOf(x, y, z) {
        return `${x},${y},${z}`;
      }

      function addBlock(x, y, z, color = 0x6ab04c) {
        const k = keyOf(x, y, z);
        if (world.has(k)) return;
        const material = new THREE.MeshLambertMaterial({ color });
        const mesh = new THREE.Mesh(blockGeometry, material);
        mesh.position.set(x + 0.5, y + 0.5, z + 0.5);
        mesh.userData = { x, y, z };
        scene.add(mesh);
        world.set(k, mesh);
      }

      function removeBlock(x, y, z) {
        const k = keyOf(x, y, z);
        const mesh = world.get(k);
        if (!mesh) return;
        scene.remove(mesh);
        world.delete(k);
      }

      function generateTerrain() {
        for (let x = -14; x <= 14; x++) {
          for (let z = -14; z <= 14; z++) {
            addBlock(x, 0, z, 0x5e9d55);

            if (Math.random() < 0.08) {
              addBlock(x, 1, z, 0x8d6e63);
            }
            if (Math.random() < 0.04) {
              addBlock(x, 2, z, 0x7cb342);
            }
          }
        }
      }

      generateTerrain();

      const player = {
        velocityY: 0,
        onGround: true,
        speed: 7,
        jumpStrength: 8,
      };

      const keys = {};
      const moveState = { x: 0, y: 0 };
      let lookX = 0;
      let lookY = 0;

      document.addEventListener("keydown", (e) => {
        keys[e.code] = true;
      });

      document.addEventListener("keyup", (e) => {
        keys[e.code] = false;
      });

      function getPlayerDirection() {
        const forward = new THREE.Vector3();
        camera.getWorldDirection(forward);
        forward.y = 0;
        forward.normalize();

        const right = new THREE.Vector3().crossVectors(forward, new THREE.Vector3(0, 1, 0)).normalize();
        return { forward, right };
      }

      function updateMovement(delta) {
        const moveVector = new THREE.Vector3();

        if (!isMobile) {
          if (keys["KeyW"]) moveVector.z -= 1;
          if (keys["KeyS"]) moveVector.z += 1;
          if (keys["KeyA"]) moveVector.x -= 1;
          if (keys["KeyD"]) moveVector.x += 1;
        } else {
          moveVector.x = moveState.x;
          moveVector.z = moveState.y;
        }

        if (moveVector.lengthSq() > 0) {
          moveVector.normalize();

          const { forward, right } = getPlayerDirection();
          const finalMove = new THREE.Vector3();

          finalMove.addScaledVector(forward, -moveVector.z);
          finalMove.addScaledVector(right, moveVector.x);

          finalMove.normalize().multiplyScalar(player.speed * delta);
          camera.position.add(finalMove);
        }

        if (!isMobile && keys["Space"] && player.onGround) {
          player.velocityY = player.jumpStrength;
          player.onGround = false;
        }

        if (isMobile && jumpPressed) {
          if (player.onGround) {
            player.velocityY = player.jumpStrength;
            player.onGround = false;
          }
          jumpPressed = false;
        }

        player.velocityY -= 18 * delta;
        camera.position.y += player.velocityY * delta;

        if (camera.position.y < 6) {
          camera.position.y = 6;
          player.velocityY = 0;
          player.onGround = true;
        }
      }

      function raycastBlock() {
        const raycaster = new THREE.Raycaster();
        raycaster.setFromCamera(new THREE.Vector2(0, 0), camera);
        const hits = raycaster.intersectObjects([...world.values()], false);
        return hits[0] || null;
      }

      document.addEventListener("mousedown", (event) => {
        if (isMobile) return;

        if (event.button === 0) {
          const hit = raycastBlock();
          if (!hit) return;
          const block = hit.object.userData;
          removeBlock(block.x, block.y, block.z);
        }

        if (event.button === 2) {
          const hit = raycastBlock();
          if (!hit) return;
          const normal = hit.face.normal.clone();
          const block = hit.object.userData;
          const x = block.x + Math.round(normal.x);
          const y = block.y + Math.round(normal.y);
          const z = block.z + Math.round(normal.z);

          if (y >= 0) addBlock(x, y, z, 0x7cb342);
        }
      });

      document.addEventListener("contextmenu", (e) => e.preventDefault());

      document.addEventListener("mousemove", (event) => {
        if (isMobile) return;

        if (document.pointerLockElement === document.body) {
          camera.rotation.y -= event.movementX * 0.002;
          camera.rotation.x -= event.movementY * 0.0018;
          camera.rotation.x = Math.max(-1.3, Math.min(1.3, camera.rotation.x));
        }
      });

      document.addEventListener("click", () => {
        if (!isMobile && document.pointerLockElement !== document.body) {
          document.body.requestPointerLock();
        }
      });

      document.getElementById("startButton").addEventListener("click", () => {
        if (isMobile) {
          document.getElementById("mobileControls").style.display = "block";
        } else {
          document.body.requestPointerLock();
        }
        document.getElementById("startButton").style.display = "none";
      });

      // mobile controls
      const joystick = document.getElementById("joystick");
      const joystickKnob = document.getElementById("joystickKnob");

      let joystickActive = false;
      let joystickOrigin = { x: 0, y: 0 };

      joystick.addEventListener("pointerdown", (e) => {
        joystickActive = true;
        joystickOrigin = { x: e.clientX, y: e.clientY };
        updateJoystick(e);
      });

      joystick.addEventListener("pointermove", (e) => {
        if (joystickActive) updateJoystick(e);
      });

      joystick.addEventListener("pointerup", () => {
        joystickActive = false;
        moveState.x = 0;
        moveState.y = 0;
        joystickKnob.style.left = "34px";
        joystickKnob.style.top = "34px";
      });

      function updateJoystick(e) {
        const rect = joystick.getBoundingClientRect();
        const cx = rect.left + rect.width / 2;
        const cy = rect.top + rect.height / 2;

        const dx = e.clientX - cx;
        const dy = e.clientY - cy;

        const max = 28;
        const length = Math.min(Math.hypot(dx, dy), max);

        const ang = Math.atan2(dy, dx);
        const x = Math.cos(ang) * length;
        const y = Math.sin(ang) * length;

        joystickKnob.style.left = `${34 + x}px`;
        joystickKnob.style.top = `${34 + y}px`;

        moveState.x = x / max;
        moveState.y = -y / max;
      }

      let jumpPressed = false;
      let lookDrag = false;

      document.getElementById("jumpBtn").addEventListener("pointerdown", () => {
        jumpPressed = true;
      });

      document.getElementById("breakBtn").addEventListener("pointerdown", () => {
        const hit = raycastBlock();
        if (!hit) return;
        const block = hit.object.userData;
        removeBlock(block.x, block.y, block.z);
      });

      document.getElementById("placeBtn").addEventListener("pointerdown", () => {
        const hit = raycastBlock();
        if (!hit) return;
        const normal = hit.face.normal.clone();
        const block = hit.object.userData;
        const x = block.x + Math.round(normal.x);
        const y = block.y + Math.round(normal.y);
        const z = block.z + Math.round(normal.z);
        if (y >= 0) addBlock(x, y, z, 0x7cb342);
      });

      const lookArea = document.body;

      lookArea.addEventListener("pointerdown", (e) => {
        if (!isMobile) return;
        if (e.clientX < window.innerWidth / 2) return;
        lookDrag = true;
      });

      lookArea.addEventListener("pointermove", (e) => {
        if (!isMobile || !lookDrag) return;
        camera.rotation.y -= e.movementX * 0.005;
        camera.rotation.x -= e.movementY * 0.004;
        camera.rotation.x = Math.max(-1.3, Math.min(1.3, camera.rotation.x));
      });

      lookArea.addEventListener("pointerup", () => {
        lookDrag = false;
      });

      const clock = new THREE.Clock();

      function animate() {
        const delta = clock.getDelta();
        updateMovement(delta);
        renderer.render(scene, camera);
        requestAnimationFrame(animate);
      }

      animate();
    </script>
  </body>
</html>