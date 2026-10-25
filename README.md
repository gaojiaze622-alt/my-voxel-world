
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>My Voxel World</title>
<style>
* { box-sizing: border-box; }
html, body {
  margin: 0; width: 100%; height: 100%;
  overflow: hidden; background: #87ceeb;
  font-family: Arial, sans-serif;
  touch-action: none;
}
canvas {
  display: block; width: 100%; height: 100%;
  touch-action: none;
}
#help {
  position: absolute; top: 10px; left: 10px;
  color: white; background: #0008; padding: 8px 10px;
  border-radius: 8px; font-size: 12px; pointer-events: none;
}
#controls {
  position: absolute; left: 12px; bottom: 20px;
  display: grid; grid-template-columns: 54px 54px 54px;
  grid-template-rows: 54px 54px 54px; gap: 5px;
}
button {
  border: 1px solid #ffffff99; border-radius: 12px;
  background: #18202bcc; color: white; font-size: 20px;
  touch-action: none; user-select: none;
  -webkit-user-select: none; -webkit-touch-callout: none;
}
button.active { background: #4c8c54ee; }
#actions {
  position: absolute; right: 12px; bottom: 25px;
  display: flex; flex-direction: column; gap: 10px;
}
#actions button { min-width: 86px; height: 48px; font-size: 15px; }
#jump { position: absolute; right: 18px; bottom: 145px; width: 65px; height: 48px; font-size: 15px; }
#status {
  position: absolute; top: 55px; left: 10px;
  color: white; background: #0008; padding: 6px 9px;
  border-radius: 8px; font-size: 12px;
}
</style>
</head>
<body>
<div id="help">Drag to look · Hold arrows to move</div>
<div id="status">Loading world…</div>

<div id="controls">
  <div></div><button data-move="forward">▲</button><div></div>
  <button data-move="left">◀</button><button data-move="back">▼</button><button data-move="right">▶</button>
  <div></div><button data-move="down">−</button><div></div>
</div>
<button id="jump">Jump</button>
<div id="actions">
  <button id="build">＋ Build</button>
  <button id="dig">− Dig</button>
</div>

<script type="module">
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.169.0/build/three.module.js';

const status = document.getElementById('status');
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);
scene.fog = new THREE.Fog(0x87ceeb, 35, 100);

const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 150);
camera.position.set(0, 2.2, 8);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 1.7));
renderer.setSize(innerWidth, innerHeight);
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);
renderer.domElement.style.position = 'absolute';
renderer.domElement.style.inset = '0';
renderer.domElement.style.zIndex = '0';

document.querySelectorAll('#controls, #actions, #jump, #help, #status').forEach(el => {
  el.style.zIndex = '2';
});

scene.add(new THREE.HemisphereLight(0xffffff, 0x66804b, 2));
const sun = new THREE.DirectionalLight(0xffffff, 2);
sun.position.set(15, 25, 10);
scene.add(sun);

const materials = {
  grass: new THREE.MeshLambertMaterial({ color: 0x55a747 }),
  dirt: new THREE.MeshLambertMaterial({ color: 0x8b5a35 }),
  wood: new THREE.MeshLambertMaterial({ color: 0x80502b }),
  leaves: new THREE.MeshLambertMaterial({ color: 0x27823a }),
  stone: new THREE.MeshLambertMaterial({ color: 0x888888 }),
  sand: new THREE.MeshLambertMaterial({ color: 0xd9c27b })
};
const cubeGeometry = new THREE.BoxGeometry(1, 1, 1);
const blocks = [];
const blockMap = new Map();
const keyOf = (x,y,z) => `${x},${y},${z}`;

function addBlock(x, y, z, type='grass') {
  const key = keyOf(x,y,z);
  if (blockMap.has(key)) return;
  const mesh = new THREE.Mesh(cubeGeometry, materials[type] || materials.grass);
  mesh.position.set(x,y,z);
  mesh.userData = { x,y,z,type };
  scene.add(mesh);
  blocks.push(mesh);
  blockMap.set(key, mesh);
}
function removeBlock(mesh) {
  if (!mesh || mesh.position.y < 0) return;
  scene.remove(mesh);
  blockMap.delete(keyOf(mesh.position.x, mesh.position.y, mesh.position.z));
  const i = blocks.indexOf(mesh);
  if (i >= 0) blocks.splice(i,1);
}

// Large, flat world: 41 x 41 blocks
for (let x=-20; x<=20; x++) {
  for (let z=-20; z<=20; z++) {
    addBlock(x,0,z,'grass');
    if (Math.random() < 0.025) addBlock(x,-1,z,'dirt');
  }
}

// Trees spread across the expanded world
for (let i=0; i<45; i++) {
  const x = Math.floor(Math.random()*35)-17;
  const z = Math.floor(Math.random()*35)-17;
  if (Math.abs(x)<3 && Math.abs(z)<3) continue;
  const h = 3 + Math.floor(Math.random()*2);
  for (let y=1; y<=h; y++) addBlock(x,y,z,'wood');
  for (let dx=-2; dx<=2; dx++) {
    for (let dz=-2; dz<=2; dz++) {
      for (let dy=0; dy<=2; dy++) {
        if (Math.abs(dx)+Math.abs(dz)+dy < 5 && Math.random() > 0.12)
          addBlock(x+dx,h+dy,z+dz,'leaves');
      }
    }
  }
}

// A few stone patches for variety
for (let i=0; i<35; i++) {
  const x = Math.floor(Math.random()*33)-16;
  const z = Math.floor(Math.random()*33)-16;
  if (Math.abs(x)>4 || Math.abs(z)>4) addBlock(x,1,z,'stone');
}

let yaw = 0;
let pitch = -0.12;
const held = new Set();
let lookPointer = null;
let lastX = 0, lastY = 0;
let lastTouchEnd = 0;
let playerY = 2.2;
let verticalVelocity = 0;
let onGround = true;
let selectedBlock = 'grass';

function updateCamera() {
  camera.rotation.order = 'YXZ';
  camera.rotation.y = yaw;
  camera.rotation.x = pitch;
  camera.position.y = playerY;
}
updateCamera();

function releaseAll() {
  held.clear();
  document.querySelectorAll('[data-move]').forEach(b => b.classList.remove('active'));
  lookPointer = null;
}
window.addEventListener('blur', releaseAll);
document.addEventListener('visibilitychange', () => {
  if (document.hidden) releaseAll();
});
window.addEventListener('pointerup', e => {
  held.delete(e.pointerId);
  releaseAll();
});
window.addEventListener('pointercancel', releaseAll);
window.addEventListener('contextmenu', e => e.preventDefault());

document.querySelectorAll('[data-move]').forEach(button => {
  const direction = button.dataset.move;
  button.addEventListener('pointerdown', e => {
    e.preventDefault();
    e.stopPropagation();
    held.add(direction);
    button.classList.add('active');
  });
  button.addEventListener('pointerup', e => {
    e.preventDefault();
    held.delete(direction);
    button.classList.remove('active');
  });
  button.addEventListener('pointercancel', () => {
    held.delete(direction);
    button.classList.remove('active');
  });
  button.addEventListener('lostpointercapture', () => {
    held.delete(direction);
    button.classList.remove('active');
  });
});

renderer.domElement.addEventListener('pointerdown', e => {
  if (e.pointerType === 'mouse' && e.button !== 0) return;
  e.preventDefault();
  lookPointer = e.pointerId;
  lastX = e.clientX;
  lastY = e.clientY;
  try { renderer.domElement.setPointerCapture(e.pointerId); } catch {}
});
renderer.domElement.addEventListener('pointermove', e => {
  if (lookPointer !== e.pointerId) return;
  e.preventDefault();
  const dx = e.clientX - lastX;
  const dy = e.clientY - lastY;
  lastX = e.clientX;
  lastY = e.clientY;
  yaw -= dx * 0.006;
  pitch -= dy * 0.005;
  pitch = Math.max(-1.35, Math.min(1.35, pitch));
});
renderer.domElement.addEventListener('pointerup', e => {
  if (lookPointer === e.pointerId) lookPointer = null;
});
renderer.domElement.addEventListener('pointercancel', () => lookPointer = null);
renderer.domElement.addEventListener('lostpointercapture', () => lookPointer = null);

document.getElementById('jump').addEventListener('pointerdown', e => {
  e.preventDefault();
  if (onGround) {
    verticalVelocity = 5.5;
    onGround = false;
  }
});
document.getElementById('build').addEventListener('click', () => {
  const dir = new THREE.Vector3();
  camera.getWorldDirection(dir);
  const target = camera.position.clone().add(dir.multiplyScalar(4));
  const x = Math.round(target.x);
  const y = Math.max(1, Math.round(target.y));
  const z = Math.round(target.z);
  addBlock(x,y,z,selectedBlock);
});
document.getElementById('dig').addEventListener('click', () => {
  const ray = new THREE.Raycaster();
  ray.setFromCamera(new THREE.Vector2(0,0), camera);
  const hits = ray.intersectObjects(blocks, false);
  if (hits.length && hits[0].distance < 7) removeBlock(hits[0].object);
});

const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  const dt = Math.min(clock.getDelta(), 0.04);

  // Movement happens only while a direction is held; no inertia.
  const forward = new THREE.Vector3(-Math.sin(yaw), 0, -Math.cos(yaw));
  const right = new THREE.Vector3(Math.cos(yaw), 0, -Math.sin(yaw));
  const move = new THREE.Vector3();
  if (held.has('forward')) move.add(forward);
  if (held.has('back')) move.sub(forward);
  if (held.has('right')) move.add(right);
  if (held.has('left')) move.sub(right);
  if (move.lengthSq() > 0) {
    move.normalize().multiplyScalar(5 * dt);
    camera.position.x += move.x;
    camera.position.z += move.z;
  }

  if (held.has('down')) {
    camera.position.y = Math.max(1.2, camera.position.y - 3 * dt);
    playerY = camera.position.y;
  } else if (!onGround) {
    verticalVelocity -= 12 * dt;
    playerY += verticalVelocity * dt;
    if (playerY <= 2.2) {
      playerY = 2.2;
      verticalVelocity = 0;
      onGround = true;
    }
    camera.position.y = playerY;
  } else {
    camera.position.y = playerY;
  }

  updateCamera();
  renderer.render(scene, camera);
}
animate();
status.textContent = 'World ready · Hold a direction to move';
window.addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
</script>
</body>
</html>
