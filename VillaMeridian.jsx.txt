import React, { useEffect, useRef, useState, useCallback } from "react";
import * as THREE from "three";
import {
  Menu, X, Sun, Moon, Phone, ChevronRight, RotateCcw, Ruler,
  MapPin, Sofa, Waves, BedDouble, ChefHat, Flame, Mail, Calendar, Check,
  MessageCircle, Send, ArrowLeft, Sparkles, Loader2,
} from "lucide-react";

/* ------------------------------------------------------------------ */
/*  CONTENT                                                            */
/* ------------------------------------------------------------------ */

const HOTSPOTS = [
  {
    id: "living",
    icon: Sofa,
    label: "The Great Room",
    floor: "ground",
    room: "Great Room",
    stat: "1,420 sf",
    tags: ["Floor-to-ceiling glass", "Retractable walls", "White oak flooring"],
    desc: "Motorized glass walls slide fully open onto the terrace, framing an uninterrupted line of sight to the water.",
    pos: [2, 1.9, 4.6],
    camPos: [11, 6, 15],
    camTarget: [2, 2, 3.5],
  },
  {
    id: "pool",
    icon: Waves,
    label: "The Horizon Pool",
    floor: "ground",
    room: "Horizon Pool",
    stat: "1,800 sf deck",
    tags: ["60' vanishing edge", "Heated year-round", "Submerged sun ledge"],
    desc: "A sixty-foot vanishing edge blurs into the coastline, with a sun ledge just below the surface for shallow-water lounging.",
    pos: [-6, 0.5, 8],
    camPos: [-3, 4.5, 17],
    camTarget: [-6, 0.3, 8],
  },
  {
    id: "master",
    icon: BedDouble,
    label: "The Owner's Suite",
    floor: "upper",
    room: "Owner's Suite",
    stat: "960 sf",
    tags: ["Private terrace", "Dual walk-in closets", "Freestanding soaking tub"],
    desc: "Set on its own upper wing, the suite opens onto a private terrace with a soaking tub positioned to catch the sunrise.",
    pos: [-1.5, 4.9, 2.05],
    camPos: [-9, 7.5, 11],
    camTarget: [-1.5, 4.7, 1.5],
  },
  {
    id: "kitchen",
    icon: ChefHat,
    label: "The Culinary Studio",
    floor: "ground",
    room: "Culinary Studio",
    stat: "640 sf",
    tags: ["Bookmatched marble island", "Gaggenau suite", "Butler's pantry"],
    desc: "A twelve-foot bookmatched marble island anchors a kitchen built for entertaining.",
    pos: [-6.4, 1.9, -1],
    camPos: [-14, 6, 3],
    camTarget: [-6.4, 1.9, -1],
  },
  {
    id: "skylounge",
    icon: Flame,
    label: "The Rooftop Lounge",
    floor: "upper",
    room: "Rooftop Lounge",
    stat: "1,100 sf",
    tags: ["270° views", "Outdoor kitchen", "Sunken fire lounge"],
    desc: "A sunken fire lounge and outdoor kitchen turn the rooftop into the house's third living room.",
    pos: [-1.5, 6.7, -1],
    camPos: [6, 10, -11],
    camTarget: [-1.5, 6.5, -1],
  },
];

const FLOORS = {
  ground: {
    label: "Ground Floor",
    rooms: [
      { name: "Foyer", hotspot: null, x: 1, y: 1, w: 1, h: 1 },
      { name: "Great Room", hotspot: "living", x: 2, y: 1, w: 2, h: 1 },
      { name: "Dining", hotspot: null, x: 4, y: 1, w: 1, h: 1 },
      { name: "Culinary Studio", hotspot: "kitchen", x: 1, y: 2, w: 2, h: 1 },
      { name: "Guest Suite", hotspot: null, x: 4, y: 2, w: 1, h: 1 },
      { name: "Horizon Pool", hotspot: "pool", x: 1, y: 3, w: 4, h: 1 },
    ],
  },
  upper: {
    label: "Upper Floor",
    rooms: [
      { name: "Owner's Suite", hotspot: "master", x: 1, y: 1, w: 2, h: 1 },
      { name: "Bedroom Two", hotspot: null, x: 3, y: 1, w: 1, h: 1 },
      { name: "Bedroom Three", hotspot: null, x: 4, y: 1, w: 1, h: 1 },
      { name: "Study", hotspot: null, x: 1, y: 2, w: 1, h: 1 },
      { name: "Rooftop Lounge", hotspot: "skylounge", x: 2, y: 2, w: 3, h: 1 },
    ],
  },
};

const DEFAULT_CAM_POS = [20, 13, 24];
const DEFAULT_CAM_TARGET = [0, 3, 0];

/* ------------------------------------------------------------------ */
/*  THREE.JS HELPERS                                                   */
/* ------------------------------------------------------------------ */

function easeInOutCubic(t) {
  return t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;
}

/**
 * A minimal orbit-controls implementation (rotate / zoom / pan with damping),
 * built on the base "three" package only — no examples/jsm import required.
 */
function createOrbitControls(camera, domElement) {
  const controls = {
    target: new THREE.Vector3(),
    enabled: true,
    minDistance: 1,
    maxDistance: Infinity,
    maxPolarAngle: Math.PI - 0.05,
    minPolarAngle: 0.05,
  };

  const spherical = new THREE.Spherical();
  const sphericalDelta = new THREE.Spherical(0, 0, 0);
  const offset = new THREE.Vector3();
  const panOffset = new THREE.Vector3();
  let scale = 1;
  let isRotating = false;
  let isPanning = false;
  const rotateStart = new THREE.Vector2();
  const rotateEnd = new THREE.Vector2();
  const rotateDelta = new THREE.Vector2();
  const panStart = new THREE.Vector2();

  function pan(deltaX, deltaY) {
    const distance = camera.position.distanceTo(controls.target);
    const fovRad = (camera.fov * Math.PI) / 180;
    const targetHeight = 2 * Math.tan(fovRad / 2) * distance;
    const panX = new THREE.Vector3().setFromMatrixColumn(camera.matrix, 0);
    panX.multiplyScalar((-deltaX * targetHeight) / domElement.clientHeight);
    const panY = new THREE.Vector3().setFromMatrixColumn(camera.matrix, 1);
    panY.multiplyScalar((deltaY * targetHeight) / domElement.clientHeight);
    panOffset.add(panX).add(panY);
  }

  function onPointerDown(e) {
    if (!controls.enabled) return;
    if (e.button === 2 || e.button === 1) {
      isPanning = true;
      panStart.set(e.clientX, e.clientY);
    } else {
      isRotating = true;
      rotateStart.set(e.clientX, e.clientY);
    }
  }
  function onPointerMove(e) {
    if (!controls.enabled) return;
    if (isRotating) {
      rotateEnd.set(e.clientX, e.clientY);
      rotateDelta.subVectors(rotateEnd, rotateStart).multiplyScalar((2 * Math.PI) / domElement.clientHeight);
      sphericalDelta.theta -= rotateDelta.x;
      sphericalDelta.phi -= rotateDelta.y;
      rotateStart.copy(rotateEnd);
    } else if (isPanning) {
      const panEnd = new THREE.Vector2(e.clientX, e.clientY);
      const panDelta = panEnd.clone().sub(panStart);
      pan(panDelta.x, panDelta.y);
      panStart.copy(panEnd);
    }
  }
  function onPointerUp() {
    isRotating = false;
    isPanning = false;
  }
  function onWheel(e) {
    if (!controls.enabled) return;
    e.preventDefault();
    scale *= e.deltaY < 0 ? 0.92 : 1 / 0.92;
  }
  function onContextMenu(e) {
    e.preventDefault();
  }

  let pinchDist = 0;
  function onTouchStart(e) {
    if (e.touches.length === 1) {
      isRotating = true;
      rotateStart.set(e.touches[0].clientX, e.touches[0].clientY);
    } else if (e.touches.length === 2) {
      isRotating = false;
      const dx = e.touches[0].clientX - e.touches[1].clientX;
      const dy = e.touches[0].clientY - e.touches[1].clientY;
      pinchDist = Math.sqrt(dx * dx + dy * dy);
    }
  }
  function onTouchMove(e) {
    e.preventDefault();
    if (e.touches.length === 1 && isRotating) {
      rotateEnd.set(e.touches[0].clientX, e.touches[0].clientY);
      rotateDelta.subVectors(rotateEnd, rotateStart).multiplyScalar((2 * Math.PI) / domElement.clientHeight);
      sphericalDelta.theta -= rotateDelta.x;
      sphericalDelta.phi -= rotateDelta.y;
      rotateStart.copy(rotateEnd);
    } else if (e.touches.length === 2) {
      const dx = e.touches[0].clientX - e.touches[1].clientX;
      const dy = e.touches[0].clientY - e.touches[1].clientY;
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (pinchDist > 0) scale *= dist < pinchDist ? 1.05 : 0.95;
      pinchDist = dist;
    }
  }
  function onTouchEnd() {
    isRotating = false;
    pinchDist = 0;
  }

  domElement.addEventListener("pointerdown", onPointerDown);
  domElement.addEventListener("pointermove", onPointerMove);
  domElement.addEventListener("pointerup", onPointerUp);
  domElement.addEventListener("pointerleave", onPointerUp);
  domElement.addEventListener("wheel", onWheel, { passive: false });
  domElement.addEventListener("contextmenu", onContextMenu);
  domElement.addEventListener("touchstart", onTouchStart, { passive: false });
  domElement.addEventListener("touchmove", onTouchMove, { passive: false });
  domElement.addEventListener("touchend", onTouchEnd);

  offset.copy(camera.position).sub(controls.target);
  spherical.setFromVector3(offset);

  controls.update = function () {
    offset.copy(camera.position).sub(controls.target);
    spherical.setFromVector3(offset);
    spherical.theta += sphericalDelta.theta;
    spherical.phi += sphericalDelta.phi;
    spherical.phi = Math.max(controls.minPolarAngle, Math.min(controls.maxPolarAngle, spherical.phi));
    spherical.makeSafe();
    spherical.radius *= scale;
    spherical.radius = Math.max(controls.minDistance, Math.min(controls.maxDistance, spherical.radius));
    controls.target.add(panOffset);
    offset.setFromSpherical(spherical);
    camera.position.copy(controls.target).add(offset);
    camera.lookAt(controls.target);

    sphericalDelta.theta *= 0.9;
    sphericalDelta.phi *= 0.9;
    scale = 1 + (scale - 1) * 0.9;
    panOffset.set(0, 0, 0);
  };

  controls.dispose = function () {
    domElement.removeEventListener("pointerdown", onPointerDown);
    domElement.removeEventListener("pointermove", onPointerMove);
    domElement.removeEventListener("pointerup", onPointerUp);
    domElement.removeEventListener("pointerleave", onPointerUp);
    domElement.removeEventListener("wheel", onWheel);
    domElement.removeEventListener("contextmenu", onContextMenu);
    domElement.removeEventListener("touchstart", onTouchStart);
    domElement.removeEventListener("touchmove", onTouchMove);
    domElement.removeEventListener("touchend", onTouchEnd);
  };

  return controls;
}

function buildVilla(scene) {
  const refs = { windowMats: [], glowMats: [], pointLights: [], rings: [], night: {} };

  const matStone = new THREE.MeshStandardMaterial({ color: 0xece3d0, roughness: 0.92, metalness: 0.02 });
  const matStoneUpper = new THREE.MeshStandardMaterial({ color: 0xf2ebdd, roughness: 0.9, metalness: 0.02 });
  const matSlate = new THREE.MeshStandardMaterial({ color: 0x2b3240, roughness: 0.75, metalness: 0.1 });
  const matGold = new THREE.MeshStandardMaterial({ color: 0xc9a961, roughness: 0.35, metalness: 0.75 });
  const matWood = new THREE.MeshStandardMaterial({ color: 0x8a6b45, roughness: 0.65, metalness: 0.05 });
  const matGrass = new THREE.MeshStandardMaterial({ color: 0x8fa06a, roughness: 1 });
  const matHedge = new THREE.MeshStandardMaterial({ color: 0x5f7a4c, roughness: 0.95 });
  const matPath = new THREE.MeshStandardMaterial({ color: 0xcfc7ae, roughness: 0.95 });
  const matWater = new THREE.MeshPhysicalMaterial({
    color: 0x2f7f96, roughness: 0.08, metalness: 0.1, transparent: true, opacity: 0.9,
    emissive: 0x08343f, emissiveIntensity: 0.1,
  });

  function glassMat() {
    const m = new THREE.MeshPhysicalMaterial({
      color: 0x9fc4d8, transparent: true, opacity: 0.32, roughness: 0.05, metalness: 0.15,
      emissive: 0xffd9a0, emissiveIntensity: 0,
    });
    refs.windowMats.push(m);
    return m;
  }

  const villa = new THREE.Group();

  // Ground
  const ground = new THREE.Mesh(new THREE.PlaneGeometry(90, 90), matGrass);
  ground.rotation.x = -Math.PI / 2;
  ground.receiveShadow = true;
  scene.add(ground);

  // Driveway / forecourt
  const drive = new THREE.Mesh(new THREE.PlaneGeometry(6, 14), matPath);
  drive.rotation.x = -Math.PI / 2;
  drive.position.set(11, 0.01, -2);
  drive.receiveShadow = true;
  scene.add(drive);

  // Ground floor mass
  const groundBox = new THREE.Mesh(new THREE.BoxGeometry(14, 3.2, 9), matStone);
  groundBox.position.set(0, 1.6, 0);
  groundBox.castShadow = true;
  groundBox.receiveShadow = true;
  villa.add(groundBox);

  // Slate plinth base
  const plinth = new THREE.Mesh(new THREE.BoxGeometry(14.4, 0.4, 9.4), matSlate);
  plinth.position.set(0, 0.2, 0);
  plinth.receiveShadow = true;
  villa.add(plinth);

  // Upper floor mass (recessed)
  const upperBox = new THREE.Mesh(new THREE.BoxGeometry(9, 3, 6), matStoneUpper);
  upperBox.position.set(-1.5, 4.7, -1);
  upperBox.castShadow = true;
  upperBox.receiveShadow = true;
  villa.add(upperBox);

  // Roof slab
  const roof = new THREE.Mesh(new THREE.BoxGeometry(9.6, 0.3, 6.6), matSlate);
  roof.position.set(-1.5, 6.35, -1);
  roof.castShadow = true;
  roof.receiveShadow = true;
  villa.add(roof);

  // Ground floor terrace canopy (cantilever)
  const canopy = new THREE.Mesh(new THREE.BoxGeometry(6.2, 0.22, 4.2), matSlate);
  canopy.position.set(4.2, 3.35, 3.2);
  canopy.castShadow = true;
  villa.add(canopy);
  const canopyEdge = new THREE.Mesh(new THREE.BoxGeometry(6.2, 0.06, 0.06), matGold);
  canopyEdge.position.set(4.2, 3.22, 5.3);
  villa.add(canopyEdge);

  // Front glass wall (Great Room)
  const frontGlass = new THREE.Mesh(new THREE.BoxGeometry(9, 2.6, 0.06), glassMat());
  frontGlass.position.set(2, 1.7, 4.53);
  villa.add(frontGlass);
  for (let i = 0; i <= 6; i++) {
    const mullion = new THREE.Mesh(new THREE.BoxGeometry(0.06, 2.6, 0.1), matGold);
    mullion.position.set(2 - 4.5 + i * 1.5, 1.7, 4.55);
    villa.add(mullion);
  }

  // Side glass (Kitchen)
  const sideGlass = new THREE.Mesh(new THREE.BoxGeometry(0.06, 2.4, 5.5), glassMat());
  sideGlass.position.set(-7.03, 1.7, -1.2);
  villa.add(sideGlass);

  // Upper front glass (Master)
  const upperGlass = new THREE.Mesh(new THREE.BoxGeometry(5, 2.2, 0.06), glassMat());
  upperGlass.position.set(-1.5, 4.9, 2.03);
  villa.add(upperGlass);
  for (let i = 0; i <= 3; i++) {
    const mullion = new THREE.Mesh(new THREE.BoxGeometry(0.05, 2.2, 0.08), matGold);
    mullion.position.set(-1.5 - 2 + i * 1.34, 4.9, 2.05);
    villa.add(mullion);
  }

  // Entrance steps
  for (let i = 0; i < 3; i++) {
    const step = new THREE.Mesh(new THREE.BoxGeometry(2.6 - i * 0.2, 0.16, 1 - i * 0.15), matStone);
    step.position.set(7.4, 0.08 + i * 0.16, -1.5);
    step.receiveShadow = true;
    step.castShadow = true;
    villa.add(step);
  }

  // Interior warm glow lights (ground + upper), on only at night
  [[2, 2.2, 3], [-6, 2.2, -1], [-1.5, 5, 1.6]].forEach((p) => {
    const l = new THREE.PointLight(0xffc98a, 0, 9, 2);
    l.position.set(...p);
    villa.add(l);
    refs.pointLights.push(l);
  });

  scene.add(villa);

  /* ---- Pool ---- */
  const poolDeck = new THREE.Mesh(new THREE.PlaneGeometry(9, 13), matPath);
  poolDeck.rotation.x = -Math.PI / 2;
  poolDeck.position.set(-6, 0.02, 8);
  poolDeck.receiveShadow = true;
  scene.add(poolDeck);

  const poolWater = new THREE.Mesh(new THREE.PlaneGeometry(5, 10), matWater);
  poolWater.rotation.x = -Math.PI / 2;
  poolWater.position.set(-6, 0.06, 8);
  scene.add(poolWater);

  const poolLight = new THREE.PointLight(0x6fd6ff, 0, 8, 2);
  poolLight.position.set(-6, 0.3, 8);
  scene.add(poolLight);
  refs.pointLights.push(poolLight);

  /* ---- Rooftop pergola / fire lounge ---- */
  const pergolaGroup = new THREE.Group();
  pergolaGroup.position.set(-1.5, 6.5, -1);
  const colGeo = new THREE.CylinderGeometry(0.08, 0.08, 1.6, 10);
  [[-2, 0, -2.4], [2, 0, -2.4], [-2, 0, 2.4], [2, 0, 2.4]].forEach((p) => {
    const c = new THREE.Mesh(colGeo, matGold);
    c.position.set(p[0], 0.8, p[2]);
    c.castShadow = true;
    pergolaGroup.add(c);
  });
  for (let i = -2; i <= 2; i++) {
    const slat = new THREE.Mesh(new THREE.BoxGeometry(4.2, 0.05, 0.08), matWood);
    slat.position.set(0, 1.65, i * 1.0);
    slat.castShadow = true;
    pergolaGroup.add(slat);
  }
  const firePit = new THREE.Mesh(new THREE.CylinderGeometry(0.45, 0.5, 0.3, 20), matSlate);
  firePit.position.set(0, 0.15, 0);
  pergolaGroup.add(firePit);
  const flame = new THREE.Mesh(
    new THREE.ConeGeometry(0.18, 0.4, 8),
    new THREE.MeshStandardMaterial({ color: 0xff8a3d, emissive: 0xff6a1a, emissiveIntensity: 0 })
  );
  flame.position.set(0, 0.45, 0);
  pergolaGroup.add(flame);
  refs.glowMats.push(flame.material);
  // low seating
  [[-1.1, 0.75], [1.1, -0.75]].forEach((p) => {
    const seat = new THREE.Mesh(new THREE.BoxGeometry(1.6, 0.35, 0.6), matSlate);
    seat.position.set(p[0], 0.18, p[1]);
    seat.castShadow = true;
    pergolaGroup.add(seat);
  });
  scene.add(pergolaGroup);

  /* ---- Landscaping ---- */
  function makeTree(x, z, scale = 1) {
    const g = new THREE.Group();
    const trunk = new THREE.Mesh(new THREE.CylinderGeometry(0.12 * scale, 0.16 * scale, 1.6 * scale, 8), matWood);
    trunk.position.y = 0.8 * scale;
    trunk.castShadow = true;
    g.add(trunk);
    for (let i = 0; i < 3; i++) {
      const foliage = new THREE.Mesh(
        new THREE.IcosahedronGeometry(0.7 * scale, 0),
        matHedge
      );
      foliage.position.set((Math.random() - 0.5) * 0.4 * scale, 1.6 * scale + i * 0.4 * scale, (Math.random() - 0.5) * 0.4 * scale);
      foliage.castShadow = true;
      g.add(foliage);
    }
    g.position.set(x, 0, z);
    scene.add(g);
  }
  const treeSpots = [[-15, 5], [-16, -8], [12, -10], [15, 10], [-2, -14], [9, 14], [-13, 12]];
  treeSpots.forEach(([x, z], i) => makeTree(x, z, 0.9 + (i % 3) * 0.25));

  function makeHedgeRow(x, z, len, rot = 0) {
    const hedge = new THREE.Mesh(new THREE.BoxGeometry(len, 0.6, 0.6), matHedge);
    hedge.position.set(x, 0.3, z);
    hedge.rotation.y = rot;
    hedge.castShadow = true;
    hedge.receiveShadow = true;
    scene.add(hedge);
  }
  makeHedgeRow(-6, 3, 9);
  makeHedgeRow(-6, 13, 9);
  makeHedgeRow(-10.6, 8, 10, Math.PI / 2);

  // path lights
  const pathLightPositions = [[8.5, -5], [7, 2], [-2, 12], [-10, 8]];
  pathLightPositions.forEach((p) => {
    const pole = new THREE.Mesh(new THREE.CylinderGeometry(0.04, 0.04, 0.9, 8), matSlate);
    pole.position.set(p[0], 0.45, p[1]);
    scene.add(pole);
    const bulbMat = new THREE.MeshStandardMaterial({ color: 0xffe3b0, emissive: 0xffcf8a, emissiveIntensity: 0 });
    const bulb = new THREE.Mesh(new THREE.SphereGeometry(0.09, 10, 10), bulbMat);
    bulb.position.set(p[0], 0.92, p[1]);
    scene.add(bulb);
    refs.glowMats.push(bulbMat);
  });

  /* ---- Hotspot ring markers (decorative, pulse) ---- */
  HOTSPOTS.forEach((hs) => {
    const ring = new THREE.Mesh(
      new THREE.RingGeometry(0.22, 0.28, 32),
      new THREE.MeshBasicMaterial({ color: 0xc9a961, transparent: true, opacity: 0.85, side: THREE.DoubleSide })
    );
    ring.position.set(...hs.pos);
    ring.lookAt(new THREE.Vector3(hs.pos[0], hs.pos[1], hs.pos[2] + 5));
    ring.userData.phase = Math.random() * Math.PI * 2;
    scene.add(ring);
    refs.rings.push(ring);
  });

  /* ---- Stars ---- */
  const starGeo = new THREE.BufferGeometry();
  const starCount = 700;
  const starPos = new Float32Array(starCount * 3);
  for (let i = 0; i < starCount; i++) {
    const r = 60 + Math.random() * 40;
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.random() * Math.PI * 0.5;
    starPos[i * 3] = r * Math.sin(phi) * Math.cos(theta);
    starPos[i * 3 + 1] = Math.abs(r * Math.cos(phi)) + 6;
    starPos[i * 3 + 2] = r * Math.sin(phi) * Math.sin(theta);
  }
  starGeo.setAttribute("position", new THREE.BufferAttribute(starPos, 3));
  const stars = new THREE.Points(
    starGeo,
    new THREE.PointsMaterial({ color: 0xffffff, size: 0.18, transparent: true, opacity: 0 })
  );
  scene.add(stars);
  refs.stars = stars;

  return refs;
}

/* ------------------------------------------------------------------ */
/*  UI SUB-COMPONENTS                                                  */
/* ------------------------------------------------------------------ */

function SunArcToggle({ dayMode, onToggle }) {
  return (
    <button
      onClick={onToggle}
      className="group relative w-[168px] h-[74px] rounded-2xl overflow-hidden border border-white/15 backdrop-blur-xl shadow-[0_8px_30px_rgba(0,0,0,0.35)] transition-colors duration-700"
      style={{
        background: dayMode
          ? "linear-gradient(180deg,#EAF3F8 0%,#BFDBEA 65%,#dfe8dc 100%)"
          : "linear-gradient(180deg,#0B1220 0%,#1B2A4A 70%,#1e2438 100%)",
      }}
      aria-label="Toggle day and night mode"
    >
      <svg className="absolute inset-0 w-full h-full" viewBox="0 0 168 74">
        <path
          d="M 14 58 A 70 70 0 0 1 154 58"
          fill="none"
          stroke={dayMode ? "#C9A961" : "#5b6a8f"}
          strokeWidth="1"
          strokeDasharray="2 5"
          opacity="0.7"
        />
      </svg>
      <div
        className="absolute w-9 h-9 rounded-full flex items-center justify-center shadow-lg transition-all duration-700 ease-[cubic-bezier(.65,0,.35,1)]"
        style={{
          background: dayMode ? "radial-gradient(circle at 35% 30%,#fff6df,#E4CC8E)" : "radial-gradient(circle at 35% 30%,#e7ecff,#9aa4c9)",
          left: dayMode ? "116px" : "16px",
          top: dayMode ? "10px" : "34px",
        }}
      >
        {dayMode ? <Sun size={16} className="text-amber-700" /> : <Moon size={14} className="text-slate-700" />}
      </div>
      <span
        className="absolute bottom-1.5 left-0 right-0 text-center text-[10px] tracking-[0.2em] font-medium uppercase"
        style={{ color: dayMode ? "#5b4a2a" : "#c9d3f0" }}
      >
        {dayMode ? "Daylight" : "Nightfall"}
      </span>
    </button>
  );
}

function InfoPanel({ hotspot, onClose }) {
  if (!hotspot) return null;
  const Icon = hotspot.icon;
  return (
    <div className="absolute left-4 bottom-4 md:left-6 md:bottom-6 w-[calc(100%-2rem)] max-w-sm z-30 animate-[fadeUp_.45s_ease]">
      <div className="rounded-2xl border border-white/15 bg-slate-900/55 backdrop-blur-2xl shadow-2xl p-5 text-cream-50">
        <div className="flex items-start justify-between gap-4">
          <div className="flex items-center gap-3">
            <div className="w-10 h-10 rounded-full bg-[#C9A961]/20 border border-[#C9A961]/40 flex items-center justify-center shrink-0">
              <Icon size={18} className="text-[#E4CC8E]" />
            </div>
            <div>
              <p className="text-[10px] tracking-[0.2em] uppercase text-[#C9A961]">
                {hotspot.floor === "ground" ? "Ground Floor" : "Upper Floor"}
              </p>
              <h3 className="font-serif text-xl leading-tight text-white">{hotspot.label}</h3>
            </div>
          </div>
          <button
            onClick={onClose}
            className="text-white/50 hover:text-white transition-colors p-1 -mt-1 -mr-1"
            aria-label="Close panel"
          >
            <X size={18} />
          </button>
        </div>

        <p className="mt-3 text-sm leading-relaxed text-white/75">{hotspot.desc}</p>

        <div className="mt-4 flex items-center gap-2 text-sm text-white/85">
          <Ruler size={14} className="text-[#C9A961]" />
          <span>{hotspot.stat}</span>
        </div>

        <div className="mt-3 flex flex-wrap gap-1.5">
          {hotspot.tags.map((t) => (
            <span
              key={t}
              className="text-[11px] px-2.5 py-1 rounded-full border border-white/15 bg-white/5 text-white/70"
            >
              {t}
            </span>
          ))}
        </div>
      </div>
    </div>
  );
}

function FloorPlanPanel({ activeFloor, setActiveFloor, activeHotspot, onSelectRoom, expanded, setExpanded }) {
  const floor = FLOORS[activeFloor];
  return (
    <div className="absolute right-4 bottom-4 md:right-6 md:bottom-6 z-30">
      <div className="rounded-2xl border border-white/15 bg-slate-900/55 backdrop-blur-2xl shadow-2xl overflow-hidden w-[240px]">
        <div className="flex items-center justify-between px-4 pt-3">
          <div className="flex gap-1 bg-white/5 rounded-full p-0.5 border border-white/10">
            {Object.entries(FLOORS).map(([key, f]) => (
              <button
                key={key}
                onClick={() => setActiveFloor(key)}
                className={`px-2.5 py-1 rounded-full text-[11px] tracking-wide transition-all ${
                  activeFloor === key ? "bg-[#C9A961] text-slate-900 font-medium" : "text-white/60 hover:text-white"
                }`}
              >
                {key === "ground" ? "Ground" : "Upper"}
              </button>
            ))}
          </div>
          <button
            onClick={() => setExpanded((e) => !e)}
            className="text-white/50 hover:text-white text-[10px] uppercase tracking-widest"
          >
            {expanded ? "Hide" : "Map"}
          </button>
        </div>

        {expanded && (
          <div className="p-3">
            <div className="grid grid-cols-5 grid-rows-3 gap-1 h-32">
              {floor.rooms.map((r) => {
                const isActive = activeHotspot && activeHotspot.room === r.name;
                const clickable = !!r.hotspot;
                return (
                  <button
                    key={r.name}
                    disabled={!clickable}
                    onClick={() => clickable && onSelectRoom(r.hotspot)}
                    style={{ gridColumn: `${r.x} / span ${r.w}`, gridRow: `${r.y} / span ${r.h}` }}
                    className={`relative rounded-[4px] border text-[8.5px] leading-tight px-1 py-0.5 text-left transition-all ${
                      isActive
                        ? "bg-[#C9A961]/30 border-[#C9A961] text-white shadow-[0_0_10px_rgba(201,169,97,0.5)]"
                        : clickable
                        ? "bg-white/[0.06] border-white/15 text-white/70 hover:bg-white/10 cursor-pointer"
                        : "bg-white/[0.02] border-white/5 text-white/30 cursor-default"
                    }`}
                  >
                    {r.name}
                  </button>
                );
              })}
            </div>
            <p className="mt-2 text-[9px] text-white/35 tracking-wide">
              Gold rooms are part of the interactive tour
            </p>
          </div>
        )}
      </div>
    </div>
  );
}

function BookingModal({ open, onClose }) {
  const [sent, setSent] = useState(false);
  if (!open) return null;
  return (
    <div className="fixed inset-0 z-[60] flex items-center justify-center p-4 bg-black/60 backdrop-blur-sm animate-[fadeIn_.3s_ease]">
      <div className="w-full max-w-md rounded-2xl border border-white/15 bg-slate-900/90 backdrop-blur-2xl shadow-2xl p-6 relative">
        <button onClick={onClose} className="absolute top-4 right-4 text-white/50 hover:text-white">
          <X size={18} />
        </button>
        {!sent ? (
          <>
            <p className="text-[10px] tracking-[0.25em] uppercase text-[#C9A961] mb-1">Private Viewing</p>
            <h3 className="font-serif text-2xl text-white mb-4">Request Your Visit</h3>
            <form
              onSubmit={(e) => {
                e.preventDefault();
                setSent(true);
              }}
              className="space-y-3"
            >
              <input required placeholder="Full name" className="w-full bg-white/5 border border-white/15 rounded-lg px-3 py-2.5 text-sm text-white placeholder-white/35 outline-none focus:border-[#C9A961] transition-colors" />
              <input required type="email" placeholder="Email address" className="w-full bg-white/5 border border-white/15 rounded-lg px-3 py-2.5 text-sm text-white placeholder-white/35 outline-none focus:border-[#C9A961] transition-colors" />
              <div className="relative">
                <Calendar size={14} className="absolute left-3 top-1/2 -translate-y-1/2 text-white/40" />
                <input required type="date" className="w-full bg-white/5 border border-white/15 rounded-lg pl-9 pr-3 py-2.5 text-sm text-white/80 outline-none focus:border-[#C9A961] transition-colors" />
              </div>
              <button type="submit" className="w-full mt-2 bg-[#C9A961] hover:bg-[#dab876] text-slate-900 font-medium text-sm rounded-lg py-2.5 transition-colors">
                Submit Request
              </button>
            </form>
          </>
        ) : (
          <div className="py-6 text-center">
            <div className="w-12 h-12 mx-auto rounded-full bg-[#C9A961]/20 border border-[#C9A961]/50 flex items-center justify-center mb-4">
              <Check size={22} className="text-[#E4CC8E]" />
            </div>
            <h3 className="font-serif text-xl text-white mb-1">Request received</h3>
            <p className="text-sm text-white/60">A Meridian agent will confirm your viewing time within 24 hours.</p>
          </div>
        )}
      </div>
    </div>
  );
}

/* ------------------------------------------------------------------ */
/*  AI CONCIERGE CHAT PAGE                                             */
/* ------------------------------------------------------------------ */

const PROPERTY_SYSTEM_PROMPT = `You are the "Meridian Concierge," a warm, knowledgeable AI assistant embedded in the website for The Meridian Residence — a $18,750,000 luxury villa for sale in Paradise Cove, Malibu.

Property facts you know:
- 8,240 square feet, on a 1.3 acre cliffside lot, built 2024
- 5 bedrooms, 6.5 bathrooms
- The Great Room (1,420 sf): motorized floor-to-ceiling glass walls open fully onto the terrace
- The Horizon Pool: 60-foot vanishing edge, heated year-round, submerged sun ledge, 1,800 sf deck
- The Owner's Suite (960 sf): private terrace, dual walk-in closets, freestanding soaking tub, upper floor
- The Culinary Studio (640 sf): 12-foot bookmatched marble island, Gaggenau appliance suite, butler's pantry
- The Rooftop Lounge (1,100 sf): 270-degree ocean views, outdoor kitchen, sunken fire lounge
- Listing agent contact: +1 (310) 555-0148, viewings@meridian.house
- Private viewings can be booked through the "Book Private Viewing" button on the site

Answer questions about the property, its rooms, amenities, price, and the buying/viewing process in a concise, elegant, helpful tone — like a top-tier real estate concierge. Keep answers to a few sentences unless asked for more detail. If asked something you can't know (e.g. final negotiated price, HOA specifics not listed above), say so honestly and suggest contacting the listing agent. Never invent legal, financial, or structural details not given above.`;

const CHAT_SUGGESTIONS = [
  "What makes this pool special?",
  "Walk me through the price and size",
  "How do I book a private viewing?",
];

function ConciergeChatPage({ onBack }) {
  const [messages, setMessages] = useState([
    {
      role: "assistant",
      content:
        "Welcome — I'm the Meridian Concierge. Ask me anything about the residence: rooms, amenities, pricing, or how to schedule a private viewing.",
    },
  ]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(false);
  const scrollRef = useRef(null);

  useEffect(() => {
    scrollRef.current?.scrollTo({ top: scrollRef.current.scrollHeight, behavior: "smooth" });
  }, [messages, loading]);

  const sendMessage = useCallback(
    async (text) => {
      const trimmed = text.trim();
      if (!trimmed || loading) return;
      const nextMessages = [...messages, { role: "user", content: trimmed }];
      setMessages(nextMessages);
      setInput("");
      setLoading(true);
      setError(false);
      try {
        const response = await fetch("https://api.anthropic.com/v1/messages", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            model: "claude-sonnet-4-6",
            max_tokens: 1000,
            system: PROPERTY_SYSTEM_PROMPT,
            messages: nextMessages.map((m) => ({ role: m.role, content: m.content })),
          }),
        });
        const data = await response.json();
        const text = (data.content || [])
          .map((block) => (block.type === "text" ? block.text : ""))
          .filter(Boolean)
          .join("\n")
          .trim();
        setMessages((prev) => [
          ...prev,
          { role: "assistant", content: text || "I wasn't able to find an answer to that — please try rephrasing." },
        ]);
      } catch (err) {
        setError(true);
        setMessages((prev) => [
          ...prev,
          { role: "assistant", content: "I'm having trouble connecting right now. Please try again in a moment." },
        ]);
      } finally {
        setLoading(false);
      }
    },
    [messages, loading]
  );

  return (
    <div
      className="fixed inset-0 z-[100] flex flex-col animate-[fadeIn_.35s_ease]"
      style={{ background: "linear-gradient(160deg,#0B1220 0%,#161d2c 55%,#1c2438 100%)", fontFamily: "'Inter', sans-serif" }}
    >
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Fraunces:ital,wght@0,400;0,500;0,600;1,500&family=Inter:wght@400;500;600;700&display=swap');
        .font-serif { font-family: 'Fraunces', serif; }
      `}</style>

      {/* Header */}
      <div className="flex items-center justify-between px-5 md:px-8 py-4 md:py-5 border-b border-white/10 shrink-0">
        <button
          onClick={onBack}
          className="flex items-center gap-2 text-white/70 hover:text-white text-sm transition-colors"
        >
          <ArrowLeft size={16} /> Back to Tour
        </button>
        <div className="flex items-center gap-2">
          <div className="w-7 h-7 rounded-full bg-[#C9A961]/20 border border-[#C9A961]/50 flex items-center justify-center">
            <Sparkles size={13} className="text-[#E4CC8E]" />
          </div>
          <span className="font-serif text-base tracking-[0.1em] text-white uppercase">Meridian Concierge</span>
        </div>
        <div className="w-[92px]" />
      </div>

      {/* Messages */}
      <div ref={scrollRef} className="flex-1 overflow-y-auto px-4 md:px-0">
        <div className="max-w-2xl mx-auto py-6 space-y-4">
          {messages.map((m, i) => (
            <div key={i} className={`flex ${m.role === "user" ? "justify-end" : "justify-start"}`}>
              {m.role === "assistant" && (
                <div className="w-8 h-8 rounded-full bg-[#C9A961]/20 border border-[#C9A961]/40 flex items-center justify-center mr-2.5 shrink-0 mt-0.5">
                  <Sparkles size={13} className="text-[#E4CC8E]" />
                </div>
              )}
              <div
                className={`max-w-[80%] rounded-2xl px-4 py-2.5 text-sm leading-relaxed ${
                  m.role === "user"
                    ? "bg-[#C9A961] text-slate-900 rounded-tr-sm"
                    : "bg-white/[0.06] border border-white/10 text-white/85 rounded-tl-sm backdrop-blur"
                }`}
              >
                {m.content}
              </div>
            </div>
          ))}
          {loading && (
            <div className="flex justify-start">
              <div className="w-8 h-8 rounded-full bg-[#C9A961]/20 border border-[#C9A961]/40 flex items-center justify-center mr-2.5 shrink-0">
                <Sparkles size={13} className="text-[#E4CC8E]" />
              </div>
              <div className="bg-white/[0.06] border border-white/10 rounded-2xl rounded-tl-sm px-4 py-3 flex items-center gap-1.5">
                <Loader2 size={14} className="animate-spin text-[#E4CC8E]" />
                <span className="text-xs text-white/50">Thinking…</span>
              </div>
            </div>
          )}

          {messages.length === 1 && !loading && (
            <div className="flex flex-wrap gap-2 pt-2">
              {CHAT_SUGGESTIONS.map((s) => (
                <button
                  key={s}
                  onClick={() => sendMessage(s)}
                  className="text-xs px-3 py-2 rounded-full border border-white/15 bg-white/[0.04] text-white/70 hover:bg-white/10 hover:text-white transition-colors"
                >
                  {s}
                </button>
              ))}
            </div>
          )}
        </div>
      </div>

      {/* Input */}
      <div className="border-t border-white/10 px-4 md:px-0 py-4 shrink-0">
        <form
          onSubmit={(e) => {
            e.preventDefault();
            sendMessage(input);
          }}
          className="max-w-2xl mx-auto flex items-center gap-2"
        >
          <input
            value={input}
            onChange={(e) => setInput(e.target.value)}
            placeholder="Ask about the residence…"
            className="flex-1 bg-white/[0.06] border border-white/15 rounded-full px-4 py-3 text-sm text-white placeholder-white/35 outline-none focus:border-[#C9A961] transition-colors"
          />
          <button
            type="submit"
            disabled={loading || !input.trim()}
            className="w-11 h-11 shrink-0 rounded-full bg-[#C9A961] hover:bg-[#dab876] disabled:opacity-40 disabled:cursor-not-allowed text-slate-900 flex items-center justify-center transition-colors"
            aria-label="Send message"
          >
            <Send size={16} />
          </button>
        </form>
        {error && (
          <p className="max-w-2xl mx-auto text-[11px] text-white/40 mt-2 text-center">
            Trouble reaching the concierge — you can also call +1 (310) 555-0148.
          </p>
        )}
      </div>
    </div>
  );
}

/* ------------------------------------------------------------------ */
/*  MAIN COMPONENT                                                     */
/* ------------------------------------------------------------------ */

export default function VillaMeridianExperience() {
  const mountRef = useRef(null);
  const markerRefs = useRef([]);
  const three = useRef({});
  const [dayMode, setDayMode] = useState(true);
  const [activeHotspot, setActiveHotspot] = useState(null);
  const [activeFloor, setActiveFloor] = useState("ground");
  const [mapExpanded, setMapExpanded] = useState(true);
  const [navOpen, setNavOpen] = useState(false);
  const [bookingOpen, setBookingOpen] = useState(false);
  const [view, setView] = useState("tour"); // "tour" | "chat"
  const [ready, setReady] = useState(false);

  /* ---------------- three.js setup (runs once) ---------------- */
  useEffect(() => {
    const mount = mountRef.current;
    const width = mount.clientWidth;
    const height = mount.clientHeight;

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(42, width / height, 0.1, 300);
    camera.position.set(...DEFAULT_CAM_POS);

    const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(width, height);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    renderer.outputEncoding = THREE.sRGBEncoding;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.05;
    mount.appendChild(renderer.domElement);

    const controls = createOrbitControls(camera, renderer.domElement);
    controls.minDistance = 6;
    controls.maxDistance = 55;
    controls.maxPolarAngle = Math.PI / 2 - 0.02;
    controls.target.set(...DEFAULT_CAM_TARGET);
    controls.update();

    // Lights
    const ambient = new THREE.AmbientLight(0xffffff, 0.55);
    scene.add(ambient);
    const hemi = new THREE.HemisphereLight(0xbfd9ea, 0x8fa06a, 0.5);
    scene.add(hemi);
    const sun = new THREE.DirectionalLight(0xfff2df, 1.4);
    sun.position.set(22, 28, 14);
    sun.castShadow = true;
    sun.shadow.mapSize.set(2048, 2048);
    sun.shadow.camera.left = -30;
    sun.shadow.camera.right = 30;
    sun.shadow.camera.top = 30;
    sun.shadow.camera.bottom = -30;
    sun.shadow.camera.far = 80;
    sun.shadow.bias = -0.0003;
    scene.add(sun);

    scene.background = new THREE.Color(0xcfe4ee);
    scene.fog = new THREE.Fog(0xcfe4ee, 40, 110);

    const villaRefs = buildVilla(scene);

    three.current = { scene, camera, renderer, controls, ambient, hemi, sun, villaRefs, animId: null, flightId: null };

    /* ---- resize ---- */
    const ro = new ResizeObserver(() => {
      const w = mount.clientWidth;
      const h = mount.clientHeight;
      if (w === 0 || h === 0) return;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
    });
    ro.observe(mount);

    /* ---- animate loop ---- */
    const clock = new THREE.Clock();
    function animate() {
      three.current.animId = requestAnimationFrame(animate);
      const t = clock.getElapsedTime();
      controls.update();
      villaRefs.rings.forEach((r) => {
        const s = 1 + 0.18 * Math.sin(t * 2.4 + r.userData.phase);
        r.scale.set(s, s, s);
      });
      renderer.render(scene, camera);
      updateMarkerPositions();
    }
    animate();

    function updateMarkerPositions() {
      const cam = three.current.camera;
      const rect = renderer.domElement.getBoundingClientRect();
      HOTSPOTS.forEach((hs, i) => {
        const el = markerRefs.current[i];
        if (!el) return;
        const v = new THREE.Vector3(...hs.pos).project(cam);
        const visible = v.z < 1;
        const x = (v.x * 0.5 + 0.5) * rect.width;
        const y = (-v.y * 0.5 + 0.5) * rect.height;
        el.style.display = visible ? "flex" : "none";
        el.style.transform = `translate(${x - 18}px, ${y - 18}px)`;
      });
    }

    setTimeout(() => setReady(true), 200);

    return () => {
      cancelAnimationFrame(three.current.animId);
      ro.disconnect();
      controls.dispose();
      renderer.dispose();
      if (mount.contains(renderer.domElement)) mount.removeChild(renderer.domElement);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  /* ---------------- day / night lighting transition ---------------- */
  useEffect(() => {
    const t3 = three.current;
    if (!t3.scene) return;
    const { scene, sun, ambient, hemi, villaRefs } = t3;

    const dayVals = {
      bg: new THREE.Color(0xcfe4ee),
      sunColor: new THREE.Color(0xfff2df),
      sunIntensity: 1.4,
      sunPos: new THREE.Vector3(22, 28, 14),
      ambient: 0.55,
      hemi: 0.5,
      fogColor: new THREE.Color(0xcfe4ee),
      windowEmissive: 0,
      glow: 0,
      pointLight: 0,
      starOpacity: 0,
    };
    const nightVals = {
      bg: new THREE.Color(0x0b1220),
      sunColor: new THREE.Color(0x7c8fc9),
      sunIntensity: 0.28,
      sunPos: new THREE.Vector3(-16, 18, -10),
      ambient: 0.12,
      hemi: 0.12,
      fogColor: new THREE.Color(0x0b1220),
      windowEmissive: 0.9,
      glow: 1,
      pointLight: 1.3,
      starOpacity: 0.9,
    };

    const target = dayMode ? dayVals : nightVals;
    const start = {
      bg: scene.background.clone(),
      sunColor: sun.color.clone(),
      sunIntensity: sun.intensity,
      sunPos: sun.position.clone(),
      ambient: ambient.intensity,
      hemi: hemi.intensity,
      fogColor: scene.fog.color.clone(),
      windowEmissive: villaRefs.windowMats[0]?.emissiveIntensity ?? 0,
      glow: villaRefs.glowMats[0]?.emissiveIntensity ?? 0,
      pointLight: villaRefs.pointLights[0]?.intensity ?? 0,
      starOpacity: villaRefs.stars.material.opacity,
    };

    const dur = 1200;
    const t0 = performance.now();
    function step(now) {
      const p = Math.min((now - t0) / dur, 1);
      const e = easeInOutCubic(p);
      scene.background.copy(start.bg).lerp(target.bg, e);
      scene.fog.color.copy(start.fogColor).lerp(target.fogColor, e);
      sun.color.copy(start.sunColor).lerp(target.sunColor, e);
      sun.intensity = THREE.MathUtils.lerp(start.sunIntensity, target.sunIntensity, e);
      sun.position.lerpVectors(start.sunPos, target.sunPos, e);
      ambient.intensity = THREE.MathUtils.lerp(start.ambient, target.ambient, e);
      hemi.intensity = THREE.MathUtils.lerp(start.hemi, target.hemi, e);
      villaRefs.windowMats.forEach((m) => (m.emissiveIntensity = THREE.MathUtils.lerp(start.windowEmissive, target.windowEmissive, e)));
      villaRefs.glowMats.forEach((m) => (m.emissiveIntensity = THREE.MathUtils.lerp(start.glow, target.glow, e)));
      villaRefs.pointLights.forEach((l) => (l.intensity = THREE.MathUtils.lerp(start.pointLight, target.pointLight, e)));
      villaRefs.stars.material.opacity = THREE.MathUtils.lerp(start.starOpacity, target.starOpacity, e);
      if (p < 1) requestAnimationFrame(step);
    }
    requestAnimationFrame(step);
  }, [dayMode]);

  /* ---------------- camera fly-to ---------------- */
  const flyTo = useCallback((toPos, toTarget, duration = 1400) => {
    const t3 = three.current;
    if (!t3.camera) return;
    const { camera, controls } = t3;
    controls.enabled = false;
    const fromPos = camera.position.clone();
    const fromTarget = controls.target.clone();
    const toPosV = new THREE.Vector3(...toPos);
    const toTargetV = new THREE.Vector3(...toTarget);
    const t0 = performance.now();
    if (t3.flightId) cancelAnimationFrame(t3.flightId);
    function step(now) {
      const p = Math.min((now - t0) / duration, 1);
      const e = easeInOutCubic(p);
      camera.position.lerpVectors(fromPos, toPosV, e);
      const curTarget = new THREE.Vector3().lerpVectors(fromTarget, toTargetV, e);
      controls.target.copy(curTarget);
      camera.lookAt(curTarget);
      if (p < 1) {
        t3.flightId = requestAnimationFrame(step);
      } else {
        controls.enabled = true;
      }
    }
    t3.flightId = requestAnimationFrame(step);
  }, []);

  const selectHotspot = useCallback(
    (hs) => {
      setActiveHotspot(hs);
      setActiveFloor(hs.floor);
      flyTo(hs.camPos, hs.camTarget);
      setNavOpen(false);
    },
    [flyTo]
  );

  const selectHotspotById = useCallback(
    (id) => {
      const hs = HOTSPOTS.find((h) => h.id === id);
      if (hs) selectHotspot(hs);
    },
    [selectHotspot]
  );

  const resetView = useCallback(() => {
    setActiveHotspot(null);
    flyTo(DEFAULT_CAM_POS, DEFAULT_CAM_TARGET, 1300);
  }, [flyTo]);

  return (
    <div className="relative w-full h-screen bg-slate-950 overflow-hidden select-none" style={{ fontFamily: "'Inter', sans-serif" }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Fraunces:ital,wght@0,400;0,500;0,600;1,500&family=Inter:wght@400;500;600;700&display=swap');
        .font-serif { font-family: 'Fraunces', serif; }
        @keyframes fadeUp { from { opacity:0; transform: translateY(14px);} to {opacity:1; transform:translateY(0);} }
        @keyframes fadeIn { from { opacity:0;} to {opacity:1;} }
        @keyframes pulseMarker { 0%,100% { box-shadow: 0 0 0 0 rgba(201,169,97,0.55);} 50% { box-shadow: 0 0 0 10px rgba(201,169,97,0);} }
        .marker-pulse { animation: pulseMarker 2.4s ease-in-out infinite; }
        ::selection { background: rgba(201,169,97,0.4); }
      `}</style>

      {/* 3D canvas mount */}
      <div ref={mountRef} className="absolute inset-0" />

      {/* vignette */}
      <div className="pointer-events-none absolute inset-0 z-10" style={{ boxShadow: "inset 0 0 160px rgba(0,0,0,0.45)" }} />

      {/* loading overlay */}
      <div
        className={`absolute inset-0 z-50 bg-slate-950 flex flex-col items-center justify-center transition-opacity duration-700 ${
          ready ? "opacity-0 pointer-events-none" : "opacity-100"
        }`}
      >
        <p className="font-serif italic text-2xl text-[#E4CC8E] tracking-wide">Meridian</p>
        <p className="mt-2 text-[11px] uppercase tracking-[0.3em] text-white/40">Preparing your private tour</p>
      </div>

      {/* Hotspot markers */}
      {HOTSPOTS.map((hs, i) => {
        const Icon = hs.icon;
        const isActive = activeHotspot?.id === hs.id;
        return (
          <button
            key={hs.id}
            ref={(el) => (markerRefs.current[i] = el)}
            onClick={() => selectHotspot(hs)}
            className={`absolute z-20 w-9 h-9 rounded-full flex items-center justify-center border transition-all duration-300 ${
              isActive
                ? "bg-[#C9A961] border-[#E4CC8E] scale-110"
                : "bg-slate-900/60 border-white/40 backdrop-blur hover:bg-[#C9A961]/80 hover:border-[#E4CC8E] marker-pulse"
            }`}
            style={{ top: 0, left: 0 }}
            title={hs.label}
          >
            <Icon size={15} className={isActive ? "text-slate-900" : "text-white"} />
          </button>
        );
      })}

      {/* Top nav */}
      <div className="absolute top-0 left-0 right-0 z-40">
        <div className="flex items-center justify-between px-5 md:px-8 py-4 md:py-5">
          <div className="flex items-center gap-2">
            <div className="w-8 h-8 rounded-full border border-[#C9A961]/60 flex items-center justify-center">
              <span className="font-serif italic text-[#E4CC8E] text-sm">M</span>
            </div>
            <span className="font-serif text-lg tracking-[0.15em] text-white uppercase">Meridian</span>
          </div>

          <div className="hidden md:flex items-center gap-8">
            {["Overview", "Residence", "Amenities", "Virtual Tour"].map((l) => (
              <button
                key={l}
                onClick={() => l === "Overview" && resetView()}
                className="text-[13px] tracking-wide text-white/75 hover:text-white transition-colors"
              >
                {l}
              </button>
            ))}
          </div>

          <div className="hidden md:flex items-center gap-3">
            <button
              onClick={() => setView("chat")}
              className="flex items-center gap-1.5 px-4 py-2.5 rounded-full border border-[#C9A961]/50 text-[#E4CC8E] hover:bg-[#C9A961]/10 text-[13px] font-medium tracking-wide transition-colors"
            >
              <MessageCircle size={14} /> Talk to AI Concierge
            </button>
            <button
              onClick={() => setBookingOpen(true)}
              className="px-5 py-2.5 rounded-full bg-[#C9A961] hover:bg-[#dab876] text-slate-900 text-[13px] font-medium tracking-wide transition-colors"
            >
              Book Private Viewing
            </button>
          </div>

          <button className="md:hidden text-white" onClick={() => setNavOpen((v) => !v)}>
            {navOpen ? <X size={22} /> : <Menu size={22} />}
          </button>
        </div>

        {navOpen && (
          <div className="md:hidden mx-4 mb-4 rounded-2xl border border-white/15 bg-slate-900/80 backdrop-blur-xl p-4 space-y-3 animate-[fadeUp_.3s_ease]">
            {HOTSPOTS.map((hs) => (
              <button key={hs.id} onClick={() => selectHotspot(hs)} className="flex items-center gap-2 text-white/80 text-sm w-full text-left py-1">
                <MapPin size={14} className="text-[#C9A961]" /> {hs.label}
              </button>
            ))}
            <button
              onClick={() => {
                setView("chat");
                setNavOpen(false);
              }}
              className="flex items-center gap-2 text-[#E4CC8E] text-sm w-full text-left py-1 border-t border-white/10 pt-3 mt-1"
            >
              <MessageCircle size={14} /> Talk to AI Concierge
            </button>
            <button
              onClick={() => {
                setBookingOpen(true);
                setNavOpen(false);
              }}
              className="w-full mt-2 px-4 py-2.5 rounded-full bg-[#C9A961] text-slate-900 text-sm font-medium"
            >
              Book Private Viewing
            </button>
          </div>
        )}
      </div>

      {/* Property title block (hero) */}
      {!activeHotspot && (
        <div className="absolute left-4 top-24 md:left-8 md:top-28 z-20 max-w-xs md:max-w-sm animate-[fadeUp_.6s_ease]">
          <p className="text-[10px] tracking-[0.3em] uppercase text-[#E4CC8E]/90">Paradise Cove &middot; Malibu</p>
          <h1 className="font-serif text-3xl md:text-4xl text-white mt-1 leading-[1.1]">
            The Meridian<br />Residence
          </h1>
          <p className="mt-3 text-sm text-white/65 leading-relaxed hidden md:block">
            A residence engineered around the horizon line. Rotate the model, or click a room to step inside.
          </p>
          <div className="mt-4 flex items-center gap-4 text-white/80 text-xs">
            <span className="font-serif text-lg text-[#E4CC8E]">$18.75M</span>
            <span className="w-px h-3 bg-white/25" />
            <span>8,240 sf</span>
            <span className="w-px h-3 bg-white/25" />
            <span>5 Bed &middot; 6.5 Bath</span>
          </div>
        </div>
      )}

      {/* Reset view / controls hint */}
      <div className="absolute top-24 right-4 md:top-28 md:right-8 z-20 flex flex-col items-end gap-3">
        <button
          onClick={resetView}
          className="flex items-center gap-1.5 text-[11px] tracking-wide text-white/70 hover:text-white bg-slate-900/50 border border-white/15 backdrop-blur px-3 py-2 rounded-full transition-colors"
        >
          <RotateCcw size={12} /> Reset View
        </button>
      </div>

      {/* Sun/Moon toggle */}
      <div className="absolute left-4 top-1/2 -translate-y-1/2 md:left-8 z-20 hidden sm:block">
        <SunArcToggle dayMode={dayMode} onToggle={() => setDayMode((d) => !d)} />
      </div>
      <div className="absolute bottom-24 left-4 z-20 sm:hidden">
        <SunArcToggle dayMode={dayMode} onToggle={() => setDayMode((d) => !d)} />
      </div>

      {/* Info + floor plan panels */}
      <InfoPanel hotspot={activeHotspot} onClose={() => setActiveHotspot(null)} />
      <FloorPlanPanel
        activeFloor={activeFloor}
        setActiveFloor={setActiveFloor}
        activeHotspot={activeHotspot}
        onSelectRoom={selectHotspotById}
        expanded={mapExpanded}
        setExpanded={setMapExpanded}
      />

      {/* Contact chip */}
      <div className="hidden lg:flex absolute bottom-6 left-1/2 -translate-x-1/2 z-20 items-center gap-2 text-white/50 text-[11px]">
        <Phone size={12} /> +1 (310) 555-0148 <span className="mx-1">&middot;</span>
        <Mail size={12} /> viewings@meridian.house
      </div>

      {/* Mobile floating AI concierge launcher */}
      <button
        onClick={() => setView("chat")}
        className="md:hidden absolute bottom-24 right-4 z-20 w-12 h-12 rounded-full bg-[#C9A961] shadow-[0_8px_24px_rgba(0,0,0,0.4)] flex items-center justify-center text-slate-900"
        aria-label="Talk to AI Concierge"
      >
        <MessageCircle size={20} />
      </button>

      <BookingModal open={bookingOpen} onClose={() => setBookingOpen(false)} />
      {view === "chat" && <ConciergeChatPage onBack={() => setView("tour")} />}
    </div>
  );
}
