<script setup>
import { ref, onMounted, watch, computed } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { STLExporter } from 'three/examples/jsm/exporters/STLExporter';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry';

const items = [
  {
    id: 1,
    title: "Modern Pendant Lamp",
    image: "https://images.unsplash.com/photo-1513506003901-1e6a229e2d15?w=500&h=500&fit=crop",
    description: "Sleek ceiling lamp with adjustable height",
    price: "$249.99"
  },
  {
    id: 2,
    title: "Metal Wall Letters",
    image: "https://images.unsplash.com/photo-1618220179428-22790b461013?w=500&h=500&fit=crop",
    description: "Custom metal letters for wall decor",
    price: "$89.99"
  },
  {
    id: 3,
    title: "Steel Bookshelf",
    image: "https://images.unsplash.com/photo-1594620302200-9a762244a156?w=500&h=500&fit=crop",
    description: "Industrial style floating shelves",
    price: "$179.99"
  },
  {
    id: 4,
    title: "Copper Wall Clock",
    image: "https://images.unsplash.com/photo-1563861826100-9cb868fdbe1c?w=500&h=500&fit=crop",
    description: "Minimalist copper wall clock",
    price: "$129.99"
  },
  {
    id: 5,
    title: "Metal Room Divider",
    image: "https://images.unsplash.com/photo-1618221469555-7f3ad97540d6?w=500&h=500&fit=crop",
    description: "Geometric metal room partition",
    price: "$299.99"
  }
];

const showGallery = ref(true);
const selectedItem = ref(null);
const webglContainer = ref(null);
const lampProperties = ref({
  cableLength: 4,
  shadeSize: 1,
  shadeHeight: 1.5,
  bulbSize: 0.2,
  mountSize: 0.2
});

const textProperties = ref({
  text: "METAL",
  font: "helvetiker",
  size: 1,
  height: 0.2,
  bevelEnabled: true,
  bevelThickness: 0.03,
  bevelSize: 0.02,
  bevelSegments: 5
});

const availableFonts = [
  { name: 'Helvetiker', value: 'helvetiker' },
  { name: 'Optimer', value: 'optimer' },
  { name: 'Gentilis', value: 'gentilis' }
];

let loadedFonts = {};

let scene, camera, renderer, cube, controls;

const currentPrice = computed(() => {
  // Base price
  let price = 7;

  // Cable length contribution (2-8m) -> $0-2
  const cablePrice = ((lampProperties.value.cableLength - 2) / 6) * 2;
  
  // Shade size contribution (0.5-2x) -> $0-2
  const shadePrice = ((lampProperties.value.shadeSize - 0.5) / 1.5) * 2;
  
  // Shade height contribution (1-3m) -> $0-1
  const heightPrice = ((lampProperties.value.shadeHeight - 1) / 2) * 1;
  
  // Mount size contribution (0.1-0.4m) -> $0-1
  const mountPrice = ((lampProperties.value.mountSize - 0.1) / 0.3) * 1;
  
  // Bulb size contribution (0.1-0.4m) -> $0-1
  const bulbPrice = ((lampProperties.value.bulbSize - 0.1) / 0.3) * 1;

  // Sum all contributions
  price += cablePrice + shadePrice + heightPrice + mountPrice + bulbPrice;

  // Ensure price stays within bounds
  return Math.min(14, Math.max(7, price)).toFixed(2);
});

const textPrice = computed(() => {
  if (!selectedItem.value?.id === 2) return 0;

  // Base price
  let price = 7;

  // Letter count contribution ($0.5 per letter)
  const letterPrice = textProperties.value.text.length * 0.5;
  
  // Size contribution (0.5-2) -> $0-3
  const sizePrice = ((textProperties.value.size - 0.5) / 1.5) * 3;
  
  // Depth contribution (0.1-0.5) -> $0-2
  const depthPrice = ((textProperties.value.height - 0.1) / 0.4) * 2;
  
  // Bevel contribution (0.01-0.05) -> $0-2
  const bevelPrice = ((textProperties.value.bevelSize - 0.01) / 0.04) * 2;

  // Sum all contributions
  price += letterPrice + sizePrice + depthPrice + bevelPrice;

  // Ensure price stays within bounds (7-20)
  return Math.min(20, Math.max(7, price)).toFixed(2);
});

const updateLampProperties = () => {
  if (cube && selectedItem.value?.id === 1) {
    const lampGroup = cube;
    // Update cord length
    const cord = lampGroup.children[0];
    cord.scale.y = lampProperties.value.cableLength / 4;
    cord.position.y = lampProperties.value.cableLength / 2;

    // Update shade size
    const shade = lampGroup.children[2];
    shade.scale.set(
      lampProperties.value.shadeSize,
      lampProperties.value.shadeHeight / 1.5,
      lampProperties.value.shadeSize
    );
    shade.position.y = lampProperties.value.shadeHeight / 2;

    // Update bulb size
    const bulb = lampGroup.children[4];
    bulb.scale.set(
      lampProperties.value.bulbSize / 0.2,
      lampProperties.value.bulbSize / 0.2,
      lampProperties.value.bulbSize / 0.2
    );

    // Update mount size
    const mount = lampGroup.children[1];
    mount.scale.set(
      lampProperties.value.mountSize / 0.2,
      1,
      lampProperties.value.mountSize / 0.2
    );
  }
};

watch(lampProperties.value, updateLampProperties, { deep: true });

// Function to load fonts
const loadFont = async (fontName) => {
  if (loadedFonts[fontName]) return loadedFonts[fontName];
  
  const loader = new FontLoader();
  const font = await new Promise((resolve) => {
    loader.load(
      `https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/fonts/${fontName}_regular.typeface.json`,
      resolve
    );
  });
  loadedFonts[fontName] = font;
  return font;
};

const createTextMesh = async () => {
  const font = await loadFont(textProperties.value.font);
  const textGeometry = new TextGeometry(textProperties.value.text, {
    font: font,
    size: textProperties.value.size,
    height: textProperties.value.height,
    curveSegments: 12,
    bevelEnabled: textProperties.value.bevelEnabled,
    bevelThickness: textProperties.value.bevelThickness,
    bevelSize: textProperties.value.bevelSize,
    bevelSegments: textProperties.value.bevelSegments
  });

  const materials = [
    new THREE.MeshPhysicalMaterial({
      color: 0x888888,
      metalness: 0.9,
      roughness: 0.2,
      clearcoat: 0.5,
      clearcoatRoughness: 0.1
    }),
    new THREE.MeshPhysicalMaterial({
      color: 0x666666,
      metalness: 0.9,
      roughness: 0.3,
      clearcoat: 0.5,
      clearcoatRoughness: 0.1
    })
  ];

  const textMesh = new THREE.Mesh(textGeometry, materials);
  textGeometry.computeBoundingBox();
  
  // Center the text
  const centerOffset = new THREE.Vector3();
  textGeometry.boundingBox.getCenter(centerOffset).multiplyScalar(-1);
  textMesh.position.set(centerOffset.x, centerOffset.y, centerOffset.z);

  return textMesh;
};

const updateTextMesh = async () => {
  if (cube && selectedItem.value?.id === 2) {
    const newTextMesh = await createTextMesh();
    scene.remove(cube);
    scene.add(newTextMesh);
    cube = newTextMesh;
  }
};

const createLampMesh = () => {
  // Create lamp parts
  const lampGroup = new THREE.Group();

  // Cord
  const cordGeometry = new THREE.CylinderGeometry(0.02, 0.02, 4, 8);
  const cordMaterial = new THREE.MeshStandardMaterial({ color: 0x222222 });
  const cord = new THREE.Mesh(cordGeometry, cordMaterial);
  cord.position.y = 2;
  lampGroup.add(cord);

  // Ceiling mount
  const mountGeometry = new THREE.CylinderGeometry(0.2, 0.2, 0.1, 32);
  const mountMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x888888,
    metalness: 0.7,
    roughness: 0.3
  });
  const mount = new THREE.Mesh(mountGeometry, mountMaterial);
  mount.position.y = 4;
  lampGroup.add(mount);

  // Lamp shade
  const shadeGeometry = new THREE.ConeGeometry(1, 1.5, 32, 1, true);
  const shadeMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xcccccc,
    metalness: 0.8,
    roughness: 0.2,
    side: THREE.DoubleSide
  });
  const shade = new THREE.Mesh(shadeGeometry, shadeMaterial);
  shade.position.y = 0.5;
  lampGroup.add(shade);

  // Bulb holder
  const holderGeometry = new THREE.CylinderGeometry(0.1, 0.1, 0.3, 16);
  const holderMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x888888,
    metalness: 0.9,
    roughness: 0.1
  });
  const holder = new THREE.Mesh(holderGeometry, holderMaterial);
  holder.position.y = 0.2;
  lampGroup.add(holder);

  // Light bulb
  const bulbGeometry = new THREE.SphereGeometry(0.2, 16, 16);
  const bulbMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xffffee,
    emissive: 0xffffee,
    emissiveIntensity: 0.5,
    metalness: 0,
    roughness: 0.1
  });
  const bulb = new THREE.Mesh(bulbGeometry, bulbMaterial);
  bulb.position.y = 0;
  lampGroup.add(bulb);

  // Add point light
  const light = new THREE.PointLight(0xffffee, 2, 10);
  light.position.y = 0;
  lampGroup.add(light);

  return lampGroup;
};

const initThree = () => {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x2a2a2a);
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  renderer = new THREE.WebGLRenderer({ 
    antialias: true,
  });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  webglContainer.value.appendChild(renderer.domElement);

  // Create mesh based on selected item
  if (selectedItem.value.id === 1) {
    // Create lamp
    const lamp = createLampMesh();
    lamp.position.x = -2;
    scene.add(lamp);
    cube = lamp; // Store reference for animation
  } else if (selectedItem.value.id === 2) {
    // Create 3D text
    createTextMesh().then(textMesh => {
      scene.add(textMesh);
      cube = textMesh;
    });
  } else {
    // Create default cube for other items
    const geometry = new THREE.BoxGeometry(3, 3, 3);
    const material = new THREE.MeshStandardMaterial({
      color: 0x888888,
      metalness: 0.3,
      roughness: 0.4,
    });
    cube = new THREE.Mesh(geometry, material);
    cube.position.x = -2;
    scene.add(cube);
  }

  // Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
  directionalLight.position.set(5, 5, 5);
  scene.add(directionalLight);

  camera.position.z = 7;
  camera.position.y = 2;

  // Add OrbitControls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true; // Smooth animation
  controls.dampingFactor = 0.05;
  controls.rotateSpeed = 0.8;
  controls.panSpeed = 0.8;
  controls.zoomSpeed = 0.9;
  controls.minDistance = 3; // Minimum zoom
  controls.maxDistance = 20; // Maximum zoom
  controls.target.set(-2, 2, 0); // Look at the lamp's center
  controls.update();

  // Animation
  const animate = () => {
    requestAnimationFrame(animate);
    
    // Update controls
    controls.update();

    // No need for automatic rotation when using OrbitControls
    if (selectedItem.value.id === 1) {
      // Only keep gentle swaying for the lamp
      cube.rotation.x = Math.sin(Date.now() * 0.001) * 0.02;
      cube.rotation.z = Math.cos(Date.now() * 0.001) * 0.02;
    }

    renderer.render(scene, camera);
  };
  animate();
};

const handleItemClick = (item) => {
  if(item.id > 2) return;
  selectedItem.value = item;
  showGallery.value = false;
  setTimeout(() => {
    initThree();
  }, 500);
};

const handleBack = () => {
  showGallery.value = true;
  selectedItem.value = null;
  if (controls) {
    controls.dispose();
  }
  if (renderer) {
    webglContainer.value.removeChild(renderer.domElement);
    renderer.dispose();
  }
};

const exportDesign = () => {
  if (!cube) return;
  
  const exporter = new STLExporter();
  const result = exporter.parse(cube);
  
  // Create blob and download
  const blob = new Blob([result], { type: 'application/octet-stream' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = 'meta-design-cube.stl';
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
};

// Watch for text property changes
watch(textProperties, async () => {
  if (selectedItem.value?.id === 2) {
    await updateTextMesh();
  }
}, { deep: true });

// Clean up on component unmount
onMounted(() => {
  window.addEventListener('resize', () => {
    if (renderer && camera) {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
      if (controls) {
        controls.update();
      }
    }
  });
});
</script>

<template>
  <div class="container">
    <div class="title-container">
      <h1 class="metallic-title">META</h1>
      <h1 class="metallic-title design">DESIGN</h1>
    </div>
    
    <Transition name="fade">
      <div v-if="showGallery" class="gallery">
        <a v-for="item in items" 
           :key="item.id" 
           class="gallery-item"
           href="#"
           @click.prevent="handleItemClick(item)">
          <div class="image-container">
            <img :src="item.image" :alt="item.title">
            <div class="overlay">
              <span class="price">{{ item.price }}</span>
            </div>
          </div>
          <h3 class="item-title">{{ item.title }}</h3>
          <p class="item-description">{{ item.description }}</p>
        </a>
      </div>
    </Transition>

    <Transition name="fade">
      <div v-if="!showGallery" class="webgl-view">
        <div class="webgl-container" ref="webglContainer"></div>
        <div class="control-panel" v-if="selectedItem?.id === 1">
          <h3>Lamp Properties</h3>
          <div class="control-group">
            <label>Cable Length</label>
            <input 
              type="range" 
              v-model.number="lampProperties.cableLength" 
              min="2" 
              max="8" 
              step="0.1"
            >
            <span>{{ lampProperties.cableLength.toFixed(1) }}m</span>
          </div>
          <div class="control-group">
            <label>Shade Size</label>
            <input 
              type="range" 
              v-model.number="lampProperties.shadeSize" 
              min="0.5" 
              max="2" 
              step="0.1"
            >
            <span>{{ lampProperties.shadeSize.toFixed(1) }}x</span>
          </div>
          <div class="control-group">
            <label>Shade Height</label>
            <input 
              type="range" 
              v-model.number="lampProperties.shadeHeight" 
              min="1" 
              max="3" 
              step="0.1"
            >
            <span>{{ lampProperties.shadeHeight.toFixed(1) }}m</span>
          </div>
          <div class="control-group">
            <label>Bulb Size</label>
            <input 
              type="range" 
              v-model.number="lampProperties.bulbSize" 
              min="0.1" 
              max="0.4" 
              step="0.05"
            >
            <span>{{ lampProperties.bulbSize.toFixed(2) }}m</span>
          </div>
          <div class="control-group">
            <label>Mount Size</label>
            <input 
              type="range" 
              v-model.number="lampProperties.mountSize" 
              min="0.1" 
              max="0.4" 
              step="0.05"
            >
            <span>{{ lampProperties.mountSize.toFixed(2) }}m</span>
          </div>
          <div class="price-display">
            <span class="price-label">Total Price:</span>
            <span class="price-value">${{ currentPrice }}</span>
          </div>
          <button class="export-button" @click="exportDesign">
            <span class="export-icon">⬇</span>
            Export Design
          </button>
        </div>
        <div class="control-panel" v-else-if="selectedItem?.id === 2">
          <h3>3D Text Properties</h3>
          <div class="control-group">
            <label>Text</label>
            <input 
              type="text" 
              v-model="textProperties.text"
              class="text-input"
            >
          </div>
          <div class="control-group">
            <label>Font</label>
            <select v-model="textProperties.font" class="font-select">
              <option v-for="font in availableFonts" 
                      :key="font.value" 
                      :value="font.value">
                {{ font.name }}
              </option>
            </select>
          </div>
          <div class="control-group">
            <label>Size</label>
            <input 
              type="range" 
              v-model.number="textProperties.size" 
              min="0.5" 
              max="2" 
              step="0.1"
            >
            <span>{{ textProperties.size.toFixed(1) }}</span>
          </div>
          <div class="control-group">
            <label>Depth</label>
            <input 
              type="range" 
              v-model.number="textProperties.height" 
              min="0.1" 
              max="0.5" 
              step="0.05"
            >
            <span>{{ textProperties.height.toFixed(2) }}</span>
          </div>
          <div class="control-group">
            <label>Bevel Size</label>
            <input 
              type="range" 
              v-model.number="textProperties.bevelSize" 
              min="0.01" 
              max="0.05" 
              step="0.01"
            >
            <span>{{ textProperties.bevelSize.toFixed(2) }}</span>
          </div>
          <div class="price-display">
            <span class="price-label">Total Price:</span>
            <span class="price-value">${{ textPrice }}</span>
          </div>
          <button class="export-button" @click="exportDesign">
            <span class="export-icon">⬇</span>
            Export Design
          </button>
        </div>
        <div class="control-panel" v-else>
          <h3>Cube Dimensions</h3>
          <div class="control-group">
            <label>Width</label>
            <input 
              type="range" 
              v-model="cubeDimensions.width" 
              min="1" 
              max="5" 
              step="0.1"
            >
            <span>{{ cubeDimensions.width.toFixed(1) }}</span>
          </div>
          <div class="control-group">
            <label>Height</label>
            <input 
              type="range" 
              v-model="cubeDimensions.height" 
              min="1" 
              max="5" 
              step="0.1"
            >
            <span>{{ cubeDimensions.height.toFixed(1) }}</span>
          </div>
          <div class="control-group">
            <label>Depth</label>
            <input 
              type="range" 
              v-model="cubeDimensions.depth" 
              min="1" 
              max="5" 
              step="0.1"
            >
            <span>{{ cubeDimensions.depth.toFixed(1) }}</span>
          </div>
          <button class="export-button" @click="exportDesign">
            <span class="export-icon">⬇</span>
            Export Design
          </button>
        </div>
        <button class="back-button" @click="handleBack">Back to Gallery</button>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
.container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
  gap: 3rem;
}

.title-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

.metallic-title {
  font-size: 7rem;
  font-weight: 900;
  text-align: center;
  background: linear-gradient(
    145deg,
    #e0e0e0 0%,
    #ffffff 20%,
    #a0a0a0 40%,
    #ffffff 60%,
    #c0c0c0 80%,
    #e0e0e0 100%
  );
  background-clip: text;
  -webkit-background-clip: text;
  color: transparent;
  text-shadow: 
    4px 4px 8px rgba(0, 0, 0, 0.3),
    -2px -2px 4px rgba(255, 255, 255, 0.2),
    0 0 15px rgba(255, 255, 255, 0.1);
  letter-spacing: 0.2em;
  transform: perspective(1000px) rotateX(15deg);
  margin: 0;
  padding: 0;
  position: relative;
  -webkit-text-stroke: 1px rgba(120, 120, 120, 0.1);
}

.metallic-title.design {
  font-size: 5rem;
  background: linear-gradient(
    145deg,
    #c0c0c0 0%,
    #ffffff 30%,
    #757575 50%,
    #ffffff 70%,
    #c0c0c0 100%
  );
  background-clip: text;
  -webkit-background-clip: text;
}

.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 2rem;
  width: 100%;
  max-width: 1400px;
  padding: 0 1rem;
}

.gallery-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  transition: all 0.3s ease;
  text-decoration: none;
  cursor: pointer;
}

.gallery-item:hover {
  transform: translateY(-5px);
}

.gallery-item:hover .overlay {
  opacity: 1;
}

.image-container {
  position: relative;
  width: 100%;
  border-radius: 8px;
  overflow: hidden;
}

.gallery-item img {
  width: 100%;
  aspect-ratio: 1;
  object-fit: cover;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease;
}

.gallery-item:hover img {
  transform: scale(1.05);
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
    display: flex;
  justify-content: center;
  align-items: center;
  opacity: 0;
  transition: opacity 0.3s ease;
}

.price {
  color: white;
  font-size: 1.5rem;
  font-weight: bold;
  padding: 1rem 2rem;
  background: rgba(0, 0, 0, 0.7);
  border-radius: 4px;
  transform: translateY(20px);
  transition: transform 0.3s ease;
}

.gallery-item:hover .price {
  transform: translateY(0);
}

.item-title {
  font-size: 1.2rem;
  font-weight: 600;
  margin: 0;
  background: linear-gradient(145deg, #a0a0a0, #d0d0d0);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}

.item-description {
  font-size: 0.9rem;
  color: #666;
  text-align: center;
  margin: 0;
}

@media (max-width: 768px) {
  .metallic-title {
    font-size: 4rem;
  }
  
  .metallic-title.design {
    font-size: 3rem;
  }
  
  .gallery {
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  }

  .price {
    font-size: 1.2rem;
    padding: 0.8rem 1.6rem;
  }
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.webgl-view {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #1a1a1a;
}

.webgl-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
}

.back-button {
  position: fixed;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  z-index: 2;
  background: linear-gradient(145deg, #c0c0c0, #ffffff);
  border: none;
  padding: 1rem 2rem;
  border-radius: 4px;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.back-button:hover {
  transform: translateX(-50%) scale(1.05);
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.3);
}

.control-panel {
  position: fixed;
  right: 2rem;
  top: 50%;
  transform: translateY(-50%);
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  padding: 2rem;
  border-radius: 8px;
  z-index: 2;
  width: 300px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.control-panel h3 {
  color: white;
  margin: 0 0 1.5rem 0;
  font-size: 1.4rem;
  text-align: center;
  background: linear-gradient(145deg, #ffffff, #c0c0c0);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}

.control-group {
  margin-bottom: 1.5rem;
  display: grid;
  grid-template-columns: 60px 1fr 40px;
  align-items: center;
  gap: 1rem;
}

.control-group label {
  color: #ffffff;
  font-size: 0.9rem;
}

.control-group span {
  color: #ffffff;
  font-size: 0.9rem;
  text-align: right;
  min-width: 50px;
}

input[type="range"] {
  width: 100%;
  height: 4px;
  -webkit-appearance: none;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 2px;
  outline: none;
}

input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: linear-gradient(145deg, #ffffff, #c0c0c0);
  cursor: pointer;
  border: none;
}

input[type="range"]::-webkit-slider-thumb:hover {
  transform: scale(1.1);
}

@media (max-width: 768px) {
  .control-panel {
    right: 50%;
    bottom: 5rem;
    top: auto;
    transform: translateX(50%);
    width: 90%;
    max-width: 300px;
  }
}

.export-button {
  width: 100%;
  margin-top: 1rem;
  padding: 1rem;
  background: linear-gradient(145deg, #c0c0c0, #ffffff);
  border: none;
  border-radius: 4px;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

.export-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
}

.export-icon {
  font-size: 1.2rem;
}

.price-display {
  margin: 1.5rem 0;
  padding: 1rem;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.price-label {
  color: #ffffff;
  font-size: 1.1rem;
}

.price-value {
  color: #ffffff;
  font-size: 1.4rem;
  font-weight: bold;
  background: linear-gradient(145deg, #ffffff, #c0c0c0);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}

.text-input {
  width: 100%;
  padding: 0.5rem;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  color: white;
  font-size: 1rem;
  outline: none;
  transition: all 0.3s ease;
}

.text-input:focus {
  border-color: rgba(255, 255, 255, 0.4);
  background: rgba(255, 255, 255, 0.15);
}

.font-select {
  width: 100%;
  padding: 0.5rem;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  color: white;
  font-size: 1rem;
  outline: none;
  cursor: pointer;
  transition: all 0.3s ease;
}

.font-select:focus {
  border-color: rgba(255, 255, 255, 0.4);
  background: rgba(255, 255, 255, 0.15);
}

.font-select option {
  background: #2a2a2a;
  color: white;
}
</style>
