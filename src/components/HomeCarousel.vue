<template>
  <div class="relative w-full h-screen bg-filter">
    <div ref="container" class="w-full h-full touch-pan-y"></div>

    <div
      class="absolute inset-0 flex items-center justify-center pointer-events-none"
    >
      <div class="">
        <h1
          :class="[
            'text-6xl font-bold tracking-widest carousel-title',
            { changed: titleChanged, hovered: isAnyHovered },
          ]"
        >
          {{ activeImage.title }}
        </h1>

        <p
          :class="[
            'text-lg opacity-80 carousel-subtitle',
            { changed: titleChanged, hovered: isAnyHovered },
          ]"
        >
          {{ activeImage.Subtitle }}
        </p>
      </div>
    </div>

    <PreviousIcon
      @click="prev"
      class="absolute left-[5%] top-1/2 -translate-y-1/2 w-[7vh] cursor-pointer"
    />

    <NextIcon
      @click="next"
      class="absolute right-[5%] top-1/2 -translate-y-1/2 w-[7vh] cursor-pointer"
    />
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch, computed } from "vue";
import * as THREE from "three";
import Test from "@/images/Test.png";
import PreviousIcon from "@/elements/PreviousIcon.vue";
import NextIcon from "@/elements/NextIcon.vue";

const container = ref(null);
let scene = null;
let camera = null;
let renderer = null;
let cards = [];
const activeIndex = ref(2);
let isDragging = false;
let startX = 0;
let dragDelta = 0;
let scrollUnits = 0;
let velocity = 0;
let lastPointerX = 0;
let lastPointerTime = 0;
let lastTime = 0;
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
let hoveredMesh = null;
const isAnyHovered = ref(false);
const titleChanged = ref(false);

function mod(i, n) {
  return ((i % n) + n) % n;
}

const props = defineProps({
  images: {
    type: Array,
    default: () => [
      { src: Test, title: "Thailand", Subtitle: "Land of Smiles" },
      { src: Test, title: "United States", Subtitle: "Land of Freedom" },
      { src: Test, title: "Japan", Subtitle: "Land of the Rising Sun" },
      { src: Test, title: "France", Subtitle: "Land of Art and Culture" },
      { src: Test, title: "Brazil", Subtitle: "Land of Samba and Football" },
    ],
  },
  invertScroll: {
    type: Boolean,
    default: false,
  },
  dragMultiplier: {
    type: Number,
    default: 1.6,
  },
  swipeMultiplier: {
    type: Number,
    default: 1.3,
  },
  wheelMultiplier: {
    type: Number,
    default: 2.0,
  },
  cardGap: {
    type: Number,
    default: 3.5,
  },
  maxVisible: {
    type: Number,
    default: 5,
  },
});

const emit = defineEmits(["active-change"]);

function createCameraAndRenderer() {
  if (!container.value) return;
  const { clientWidth, clientHeight } = container.value;

  camera = new THREE.PerspectiveCamera(
    45,
    clientWidth / clientHeight,
    0.1,
    1000,
  );
  camera.position.z = 10;

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setPixelRatio(window.devicePixelRatio || 1);
  renderer.setSize(clientWidth, clientHeight);
  renderer.outputEncoding = THREE.sRGBEncoding;

  container.value.appendChild(renderer.domElement);
}

function init() {
  scene = new THREE.Scene();

  createCameraAndRenderer();

  const loader = new THREE.TextureLoader();
  cards = [];

  const imgs = props.images || [];

    imgs.forEach((img) => {
      const texture = loader.load(img.src);
      texture.encoding = THREE.sRGBEncoding;
  
      const geometry = new THREE.PlaneGeometry(3.6, 4.6);
      const material = new THREE.MeshBasicMaterial({
        map: texture,
        transparent: true,
      });
  
      const mesh = new THREE.Mesh(geometry, material);
      mesh.scale.set(1, 1, 1);
      mesh.userData = {
        targetX: 0,
        targetZ: 0,
        targetYRot: 0,
        baseScale: 1,
        hoverScale: 1.06,
        isHovered: false,
        basePositions: geometry.attributes.position.array.slice(),
        baseUVs: geometry.attributes.uv ? geometry.attributes.uv.array.slice() : null,
        cornerOffsetTarget: new Float32Array(geometry.attributes.position.array.length),
        cornerOffsetCurrent: new Float32Array(geometry.attributes.position.array.length),
        hoverRotTargetX: 0,
        hoverRotTargetY: 0,
        hoverPosOffsetX: 0,
        hoverPosOffsetY: 0,
        hoverPosOffsetZ: 0,
      };
    scene.add(mesh);
    cards.push(mesh);
  });

  updatePositions();
}

function updatePositions() {
  if (!cards.length) return;

  const n = cards.length;
  const position = activeIndex.value - scrollUnits;
  const spread = props.cardGap ?? 4.2;
  const depth = -2.2;
  const rotation = -0.6;
  const maxVis = Math.max(1, Math.min(n, props.maxVisible ?? 5));
  const half = Math.floor(maxVis / 2);
  const center = mod(Math.round(position), n);

  cards.forEach((card, i) => {
    const raw = i - position;
    const diff = mod(raw + n / 2, n) - n / 2;
    const idxDiff = mod(i - center + n / 2, n) - n / 2;

    if (Math.abs(idxDiff) <= half) {
      card.userData.targetX = diff * spread;
      card.userData.targetZ = Math.abs(diff) * depth;
      card.userData.targetYRot = diff * rotation;
      const opacity = THREE.MathUtils.clamp(1 - Math.abs(diff) * 0.1, 0.5, 1);
      card.material.opacity = opacity;
      card.material.transparent = true;
      card.visible = true;
    } else {
      const side = idxDiff > 0 ? 1 : -1;
      card.userData.targetX = side * (spread * (half + 1) + 6);
      card.userData.targetZ = -10;
      card.userData.targetYRot = 0;
      card.material.opacity = 0;
      card.material.transparent = true;
      card.visible = false;
    }
  });
}

watch(activeIndex, (v) => emit("active-change", v));

watch(activeIndex, () => {
  titleChanged.value = true;
  setTimeout(() => (titleChanged.value = false), 320);
});

function animate() {
  if (!renderer || !scene || !camera) return;
  requestAnimationFrame(animate);
  const now = performance.now();
  const dt = lastTime ? (now - lastTime) / 1000 : 0;
  lastTime = now;

  if (!isDragging) {
    scrollUnits += velocity * dt;
    const damping = 6;
    velocity -= velocity * Math.min(1, damping * dt);
    if (Math.abs(velocity) < 0.001) velocity = 0;
    if (cards.length) {
      const n = cards.length;
      const position = activeIndex.value - scrollUnits;
      const newActive = mod(Math.round(position), n);
      if (newActive !== activeIndex.value) {
        const prevPosition = position;
        let shiftedActive = newActive;
        while (shiftedActive - prevPosition > n / 2) shiftedActive -= n;
        while (shiftedActive - prevPosition < -n / 2) shiftedActive += n;
        activeIndex.value = newActive;
        scrollUnits = shiftedActive - prevPosition;
      }
    }
  }

  updatePositions();
  const ease = 0.1;
  cards.forEach((card) => {
    const tx = card.userData.targetX ?? card.position.x;
    const tz = card.userData.targetZ ?? card.position.z;
    const tr = card.userData.targetYRot ?? card.rotation.y;

    const isH = !!card.userData.isHovered;
    const base = card.userData.baseScale ?? 1;
    const hoverScale = 1.08;

    const hoverRotY = card.userData.hoverRotTargetY ?? 0;
    const hoverRotX = card.userData.hoverRotTargetX ?? 0;
    const hoverPosX = card.userData.hoverPosOffsetX ?? 0;
    const hoverPosY = card.userData.hoverPosOffsetY ?? 0;
    const hoverPosZ = card.userData.hoverPosOffsetZ ?? 0;

    const targetScale = isH ? base * hoverScale : base;
    const targetX = tx + (isH ? hoverPosX : 0);
    const targetZ = tz + (isH ? hoverPosZ : 0);
    const targetY = isH ? hoverPosY : 0;
    const targetRotY = tr + hoverRotY;
    const targetRotX = hoverRotX;

    card.position.x += (targetX - card.position.x) * ease;
    card.position.z += (targetZ - card.position.z) * (ease * 1.2);
    card.position.y += (targetY - card.position.y) * (ease * 1.5);

    card.rotation.y += (targetRotY - card.rotation.y) * ease;
    card.rotation.x += (targetRotX - card.rotation.x) * (ease * 1.5);

    card.scale.x += (targetScale - card.scale.x) * (ease * 2);
    card.scale.y += (targetScale - card.scale.y) * (ease * 2);

    try {
      const gd = card.geometry;
      const ud = card.userData;
      if (gd && ud && ud.basePositions) {
        const base = ud.basePositions;
        const cur = ud.cornerOffsetCurrent;
        const tgt = ud.cornerOffsetTarget;
        const vcount = base.length;
        for (let i = 0; i < vcount; i++) {
          cur[i] += (tgt[i] - cur[i]) * (ease * 2.5);
        }
        const pos = gd.attributes.position.array;
        for (let i = 0; i < vcount; i++) {
          pos[i] = base[i] + cur[i];
        }
        gd.attributes.position.needsUpdate = true;
        if (gd.attributes.normal) gd.computeVertexNormals && gd.computeVertexNormals();
      }
    } catch (e) {}
  });

  if (!isDragging && velocity === 0) {
    if (Math.abs(scrollUnits) > 0.001) scrollUnits *= 0.6;
    if (Math.abs(scrollUnits) < 0.001) scrollUnits = 0;
  }

  renderer.render(scene, camera);
}

function onContainerPointerEnter() {
  isAnyHovered.value = false;
}

function onContainerPointerLeave() {
  isAnyHovered.value = false;
  hoveredMesh = null;
  cards.forEach((c) => (c.userData.isHovered = false));
}

function onMouseMove(e) {
  if (!container.value || !camera || !cards.length) return;
  const rect = container.value.getBoundingClientRect();
  mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(cards, false);
    if (intersects.length) {
      const obj = intersects[0].object;
    const idx = cards.indexOf(obj);
    const n = cards.length;
    const activeIdx = mod(activeIndex.value, n);
    // only allow hover for the active card
    if (idx === activeIdx) {
      if (hoveredMesh !== obj) {
        hoveredMesh = obj;
        cards.forEach((c, i) => (c.userData.isHovered = i === idx));
        isAnyHovered.value = true;
      }
    } else {
      hoveredMesh = null;
      cards.forEach((c) => (c.userData.isHovered = false));
      isAnyHovered.value = false;
    }
  } else {
    hoveredMesh = null;
    cards.forEach((c) => (c.userData.isHovered = false));
    isAnyHovered.value = false;
  }

  const pi = intersects && intersects.length ? intersects[0] : null;
  const pointerUV = pi && pi.uv ? { x: pi.uv.x, y: pi.uv.y } : null;

  if (pi && hoveredMesh) {
    try {
      const lp = pi.point.clone();
      hoveredMesh.worldToLocal(lp);
      const halfW = 3.6 / 2;
      const halfH = 4.6 / 2;
        const maxTilt = 0.12; 
        const rotY = (-lp.x / halfW) * maxTilt;
        const rotX = (lp.y / halfH) * maxTilt;
        const offsetX = (lp.x / halfW) * 0.12; 
        const offsetY = (lp.y / halfH) * 0.06;
        const offsetZ = Math.abs(lp.y / halfH) * 0.05;
      hoveredMesh.userData.hoverRotTargetX = rotX;
      hoveredMesh.userData.hoverRotTargetY = rotY;
      hoveredMesh.userData.hoverPosOffsetX = offsetX;
      hoveredMesh.userData.hoverPosOffsetY = offsetY;
      hoveredMesh.userData.hoverPosOffsetZ = offsetZ;
    } catch (e) {}
  } else {
    cards.forEach((c) => {
      c.userData.hoverRotTargetX = 0;
      c.userData.hoverRotTargetY = 0;
      c.userData.hoverPosOffsetX = 0;
      c.userData.hoverPosOffsetY = 0;
      c.userData.hoverPosOffsetZ = 0;
    });
  }

  cards.forEach((c) => {
    const gd = c.geometry;
    const ud = c.userData;
    if (!gd || !ud || !ud.basePositions || !ud.baseUVs || !ud.cornerOffsetTarget) return;
    const basePos = ud.basePositions;
    const baseUV = ud.baseUVs;
    const targetArr = ud.cornerOffsetTarget;

    if (ud.isHovered && pointerUV && hoveredMesh === c) {
      const pu = pointerUV.x;
      const pv = pointerUV.y;
      const vcount = basePos.length / 3;
      for (let vi = 0; vi < vcount; vi++) {
        const bi = vi * 3;
        const ui = vi * 2;
        const vu = baseUV[ui];
        const vv = baseUV[ui + 1];
        const du = vu - pu;
        const dv = vv - pv;
        const d = Math.hypot(du, dv);
        const influence = Math.max(0, 1 - d * 4);

        const sameU = Math.sign(vu - 0.5) === Math.sign(pu - 0.5);
        const sameV = Math.sign(vv - 0.5) === Math.sign(pv - 0.5);
        const isOpposite = !(sameU && sameV);

        targetArr[bi] = (vu - 0.5) * influence * 0.03;
        targetArr[bi + 1] = (vv - 0.5) * influence * 0.02;
        targetArr[bi + 2] = isOpposite ? -influence * 0.03 : influence * 0.10;
      }
    } else {
      for (let i = 0; i < targetArr.length; i++) targetArr[i] = 0;
    }
  });
}

function onResize() {
  if (!container.value || !camera || !renderer) return;
  const { clientWidth, clientHeight } = container.value;
  camera.aspect = clientWidth / clientHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(clientWidth, clientHeight);
}

function onPointerDown(e) {
  if (!container.value) return;
  isDragging = true;
  startX = e.clientX;
  dragDelta = 0;
  lastPointerX = e.clientX;
  lastPointerTime = performance.now();
  velocity = 0;
  try {
    container.value.setPointerCapture?.(e.pointerId);
  } catch (e) {}
}

function onPointerMove(e) {
  if (!isDragging) return;
  const now = performance.now();
  const dt = Math.max(1, now - lastPointerTime);
  const dx = e.clientX - lastPointerX;
  const width = container.value?.clientWidth || window.innerWidth || 1;
  const dragMul = props.dragMultiplier ?? 1;
  velocity = (dx / width / (dt / 1000)) * dragMul;
  lastPointerX = e.clientX;
  lastPointerTime = now;

  dragDelta = e.clientX - startX;
  scrollUnits = (dragDelta / (container.value.clientWidth || 1)) * dragMul;
}

function onPointerUp(e) {
  if (!isDragging) return;
  isDragging = false;
  try {
    container.value.releasePointerCapture?.(e.pointerId);
  } catch (e) {}
  const width = container.value?.clientWidth || 0;
  const minSwipe = width * 0.02;
  if (Math.abs(dragDelta) < minSwipe) {
    scrollUnits = Math.round(scrollUnits);
    velocity = 0;
  } else {
    velocity *= props.swipeMultiplier ?? 1;
  }
}

const activeImage = computed(() => {
  const imgs = props.images || [];
  if (!imgs.length) return { title: "", Subtitle: "" };
  const n = imgs.length;
  const idx = mod(activeIndex.value, n);
  return imgs[idx];
});

function onWheel(e) {
  if (!container.value) return;
  e.preventDefault();
  const width = container.value.clientWidth || window.innerWidth || 1;
  const delta = e.deltaX !== 0 ? e.deltaX : e.deltaY;
  const direction = props.invertScroll ? -1 : 1;
  const units = (-delta / width) * direction * (props.wheelMultiplier ?? 1);
  scrollUnits += units;
  velocity = -units * 8;
  lastTime = performance.now();
}

function prev() {
  if (!cards.length) return;
  const n = cards.length;
  const prevIndex = mod(activeIndex.value - 1, n);
  activeIndex.value = prevIndex;
  scrollUnits = 0;
  velocity = 0;
}

function next() {
  if (!cards.length) return;
  const n = cards.length;
  const nextIndex = mod(activeIndex.value + 1, n);
  activeIndex.value = nextIndex;
  scrollUnits = 0;
  velocity = 0;
}

onMounted(() => {
  if (!container.value) return;
  init();

  container.value.addEventListener("pointerdown", onPointerDown);
  container.value.addEventListener("wheel", onWheel, { passive: false });
  container.value.addEventListener("mousemove", onMouseMove);
  container.value.addEventListener("pointerenter", onContainerPointerEnter);
  container.value.addEventListener("pointerleave", onContainerPointerLeave);
  window.addEventListener("pointermove", onPointerMove);
  window.addEventListener("pointerup", onPointerUp);
  window.addEventListener("pointercancel", onPointerUp);
  window.addEventListener("resize", onResize);

  animate();
});

onUnmounted(() => {
  window.removeEventListener("resize", onResize);
  window.removeEventListener("pointermove", onPointerMove);
  window.removeEventListener("pointerup", onPointerUp);
  window.removeEventListener("pointercancel", onPointerUp);

  if (container.value)
    container.value.removeEventListener("pointerdown", onPointerDown);
  if (container.value) container.value.removeEventListener("wheel", onWheel);
  if (container.value)
    container.value.removeEventListener("mousemove", onMouseMove);
  if (container.value)
    container.value.removeEventListener(
      "pointerenter",
      onContainerPointerEnter,
    );
  if (container.value)
    container.value.removeEventListener(
      "pointerleave",
      onContainerPointerLeave,
    );

  if (renderer) {
    renderer.forceContextLoss();
    renderer.domElement && renderer.domElement.remove();
  }

  cards.forEach((c) => {
    try {
      c.geometry && c.geometry.dispose();
      if (c.material) {
        if (c.material.map) c.material.map.dispose();
        c.material.dispose();
      }
      scene && scene.remove(c);
    } catch (e) {}
  });
  cards = [];
});
</script>

<style scoped>
.bg-filter {
  backdrop-filter: blur(16px);
}

.carousel-title,
.carousel-subtitle {
  transition:
    transform 1000ms ease,
    opacity 500ms ease;
}

.carousel-title.changed,
.carousel-subtitle.changed {
  transform: translateY(24px);
  opacity: 0;
}
</style>
