<template>
    <div class="babylon-map-container">
      <canvas ref="mapCanvas" class="map-canvas"></canvas>
      
      <!-- UI Controls -->
      <div class="map-ui-overlay">
        <!-- Layer Controls -->
        <div class="control-panel">
          <div class="panel-header" @click="toggleLayerPanel">
            <span class="panel-title">🌍 Layers</span>
            <span class="panel-toggle">{{ layerPanelOpen ? '−' : '+' }}</span>
          </div>
          <div v-show="layerPanelOpen" class="panel-content">
            <div 
              v-for="layer in layers" 
              :key="layer.id"
              :class="['layer-button', { active: selectedLayer === layer.id }]"
              @click="switchLayer(layer.id)"
            >
              <span class="layer-icon">{{ layer.icon }}</span>
              <span class="layer-name">{{ layer.name }}</span>
              <span class="layer-description">{{ layer.description }}</span>
            </div>
          </div>
        </div>
  
        <!-- Marker Categories -->
        <div class="control-panel">
          <div class="panel-header" @click="toggleCategoryPanel">
            <span class="panel-title">🎯 Filters</span>
            <span class="panel-toggle">{{ categoryPanelOpen ? '−' : '+' }}</span>
          </div>
          <div v-show="categoryPanelOpen" class="panel-content">
            <div class="category-search">
              <input 
                v-model="categorySearch" 
                type="text" 
                placeholder="Search categories..."
                class="search-input"
              />
            </div>
            <div class="category-actions">
              <button @click="showAllCategories" class="action-btn">Show All</button>
              <button @click="hideAllCategories" class="action-btn">Hide All</button>
            </div>
            <div class="category-list">
              <div 
                v-for="category in filteredCategories" 
                :key="category.name"
                :class="['category-button', { active: category.visible }]"
                @click="toggleCategory(category.name)"
              >
                <span class="category-icon">{{ category.icon }}</span>
                <span class="category-name">{{ category.name }}</span>
                <span class="category-count">{{ category.count }}</span>
              </div>
            </div>
          </div>
        </div>
  
        <!-- Sun Controls -->
        <div class="control-panel">
          <div class="panel-header" @click="toggleSunPanel">
            <span class="panel-title">☀️ Sun</span>
            <span class="panel-toggle">{{ sunPanelOpen ? '−' : '+' }}</span>
          </div>
          <div v-show="sunPanelOpen" class="panel-content">
            <div class="sun-control">
              <label class="control-label">Azimuth (0-360°)</label>
              <input 
                v-model.number="sunAzimuth" 
                type="range" 
                min="0" 
                max="360" 
                step="1"
                class="slider-input"
                @input="updateSunPosition"
              />
              <span class="value-display">{{ sunAzimuth }}°</span>
            </div>
            <div class="sun-control">
              <label class="control-label">Elevation (0-90°)</label>
              <input 
                v-model.number="sunElevation" 
                type="range" 
                min="0" 
                max="90" 
                step="1"
                class="slider-input"
                @input="updateSunPosition"
              />
              <span class="value-display">{{ sunElevation }}°</span>
            </div>
            <div class="sun-control">
              <label class="control-label">Intensity</label>
              <input 
                v-model.number="sunIntensity" 
                type="range" 
                min="0.1" 
                max="3.0" 
                step="0.1"
                class="slider-input"
                @input="updateSunPosition"
              />
              <span class="value-display">{{ sunIntensity }}</span>
            </div>
            <div class="sun-control">
              <label class="control-label">Color</label>
              <input 
                v-model="sunColor" 
                type="color" 
                class="color-input"
                @input="updateSunPosition"
              />
            </div>
            
            <!-- Preset Sun Positions -->
            <div class="sun-presets">
              <label class="control-label">Presets</label>
              <div class="preset-buttons">
                <button @click="setSunPreset('dawn')" class="preset-btn">🌅 Dawn</button>
                <button @click="setSunPreset('noon')" class="preset-btn">☀️ Noon</button>
                <button @click="setSunPreset('dusk')" class="preset-btn">🌆 Dusk</button>
                <button @click="setSunPreset('night')" class="preset-btn">🌙 Night</button>
              </div>
            </div>
          </div>
        </div>

        <!-- Position Indicator -->
        <div class="control-panel">
          <div class="panel-header">
            <span class="panel-title">📍 Position</span>
          </div>
          <div class="panel-content">
            <div class="coords-display">
              <div class="coord-item">
                <span class="coord-label">X:</span>
                <span class="coord-value">{{ Math.round(mouseCoords.x) }}</span>
              </div>
              <div class="coord-item">
                <span class="coord-label">Y:</span>
                <span class="coord-value">{{ Math.round(mouseCoords.y) }}</span>
              </div>
            </div>
            <div class="current-layer-info">
              <span class="layer-label">Current Layer:</span>
              <span class="layer-value">{{ getCurrentLayerInfo().name }}</span>
            </div>
          </div>
        </div>
      </div>
  
      <!-- Loading Overlay -->
      <div v-if="isLoading" class="loading-overlay">
        <div class="loading-spinner"></div>
        <div class="loading-text">Loading 3D Map...</div>
      </div>
      
      <!-- Mesh Loading Overlay -->
      <div v-if="isMeshLoading" class="mesh-loading-overlay">
        <div class="mesh-loading-content">
          <div class="mesh-loading-spinner"></div>
          <div class="mesh-loading-text">Loading Meshes...</div>
          <div class="mesh-loading-progress">
            <div class="progress-bar">
              <div 
                class="progress-fill" 
                :style="{ width: meshLoadingProgress + '%' }"
              ></div>
            </div>
            <div class="progress-text">
              {{ loadedMeshesCount }} / {{ totalMeshesToLoad }} meshes loaded
            </div>
          </div>
        </div>
      </div>
      
      <!-- Tooltip -->
      <div 
        v-if="tooltipVisible" 
        class="tooltip"
        :style="{ left: tooltipX + 'px', top: tooltipY + 'px' }"
      >
        {{ tooltipText }}
      </div>
    </div>
  </template>
  
  <script setup lang="ts">
  import { ref, onMounted, onUnmounted, watch, reactive, computed } from 'vue'
  import * as BABYLON from '@babylonjs/core'
  import '@babylonjs/loaders'
  import '@babylonjs/materials'
  
  // Import data files
  import locations from '../assets/map/Locations.json'
  import entitiesData from '../assets/map/worldEntities.json'
  import npcs from '../assets/map/NPCs.json'
  import npcDefinitions from '../assets/map/NPCDefs.json'
  import heightmapsData from '../assets/map/heightmaps.json'
  import meshesData from '../assets/map/meshes.json'
  
  // Toon shader GLSL code
  const toonVertexShader = `
    precision highp float;
    // Attributes
    attribute vec3 position;
    attribute vec3 normal;
    uniform mat4 worldViewProjection;
    uniform mat4 world;
    varying vec3 vNormalW;
    varying vec3 vPositionW;
    void main(void) {
      vec4 outPosition = worldViewProjection * vec4(position, 1.0);
      vPositionW = (world * vec4(position, 1.0)).xyz;
      vNormalW = normalize(mat3(world) * normal);
      gl_Position = outPosition;
    }
  `;

  const toonFragmentShader = `
    precision highp float;
    varying vec3 vNormalW;
    varying vec3 vPositionW;
    uniform vec3 lightDirection;
    uniform vec3 diffuseColor;
    uniform vec3 rimColor;
    uniform float rimPower;
    uniform float band1;
    uniform float band2;
    void main(void) {
      // Lighting
      float NdotL = max(dot(normalize(vNormalW), -lightDirection), 0.0);
      float shade = 0.0;
      if (NdotL > band2) {
        shade = 1.0;
      } else if (NdotL > band1) {
        shade = 0.6;
      } else {
        shade = 0.25;
      }
      vec3 color = diffuseColor * shade;
      // Rim lighting
      float rim = 1.0 - max(dot(normalize(vNormalW), normalize(vPositionW)), 0.0);
      rim = smoothstep(rimPower - 0.01, rimPower + 0.01, rim);
      color = mix(color, rimColor, rim * 0.7);
      gl_FragColor = vec4(color, 1.0);
    }
  `;

  // Register the shader with Babylon
  if (!BABYLON.Effect.ShadersStore["toonVertexShader"]) {
    BABYLON.Effect.ShadersStore["toonVertexShader"] = toonVertexShader;
    BABYLON.Effect.ShadersStore["toonFragmentShader"] = toonFragmentShader;
  }

  // Utility to create a toon ShaderMaterial
  const createToonMaterial = (scene: BABYLON.Scene, color: BABYLON.Color3) => {
    const mat = new BABYLON.ShaderMaterial("toonMat", scene, {
      vertex: "toon",
      fragment: "toon",
    }, {
      attributes: ["position", "normal"],
      uniforms: ["world", "worldViewProjection", "lightDirection", "diffuseColor", "rimColor", "rimPower", "band1", "band2"]
    });
    // Set defaults
    mat.setVector3("lightDirection", new BABYLON.Vector3(0.5, 1, 0.5).normalize());
    mat.setColor3("diffuseColor", color);
    mat.setColor3("rimColor", new BABYLON.Color3(1, 1, 1));
    mat.setFloat("rimPower", 0.7);
    mat.setFloat("band1", 0.3);
    mat.setFloat("band2", 0.7);
    mat.backFaceCulling = false;
    return mat;
  }

  // Function to add mesh and its children to shadow system
  const addMeshAndChildrenToShadows = (mesh: BABYLON.Mesh) => {
    if (shadowGenerator) {
      shadowGenerator.addShadowCaster(mesh, true)
      
      // Also add all child meshes
      const childMeshes = mesh.getChildMeshes()
      for (const childMesh of childMeshes) {
        if (childMesh instanceof BABYLON.Mesh) {
          shadowGenerator!.addShadowCaster(childMesh, true)
        }
      }
    }
  }
  
  // Types and interfaces
  interface Layer {
    id: string
    name: string
    icon: string
    color: string
    description: string
  }
  
  interface MarkerCategory {
    name: string
    icon: string
    visible: boolean
    count: number
  }
  
  interface MapFeature {
    coordinates: BABYLON.Vector3
    name: string
    icon: string
    category: string
    isLocation?: boolean
  }
  
  // BabylonJS engine and scene
  let engine: BABYLON.Engine
  let scene: BABYLON.Scene
  let camera: BABYLON.ArcRotateCamera
  let canvas: HTMLCanvasElement
  
  // Map state
  const selectedLayer = ref('Overworld')
  const isLoading = ref(true)
  const mouseCoords = reactive({ x: 0, y: 0 })
  
  // Mesh loading state
  const isMeshLoading = ref(false)
  const meshLoadingProgress = ref(0)
  const totalMeshesToLoad = ref(0)
  const loadedMeshesCount = ref(0)
  
  // Tooltip state
  const tooltipVisible = ref(false)
  const tooltipText = ref('')
  const tooltipX = ref(0)
  const tooltipY = ref(0)
  
  // UI Panel state
  const layerPanelOpen = ref(true)
  const categoryPanelOpen = ref(true)
  const categorySearch = ref('')
  
  // Sun lighting state
  const sunPanelOpen = ref(false)
  const sunAzimuth = ref(45) // 0-360 degrees
  const sunElevation = ref(45) // 0-90 degrees
  const sunIntensity = ref(1.2)
  const sunColor = ref('#FFE5B4') // Warm sunlight color
  
  // Sun light reference
  let sunLight: BABYLON.DirectionalLight | null = null
  let ambientLight: BABYLON.HemisphericLight | null = null
  let shadowGenerator: BABYLON.ShadowGenerator | null = null
  
  // Computed properties
  const filteredCategories = computed(() => {
    if (!categorySearch.value) return markerCategories.value
    return markerCategories.value.filter(category => 
      category.name.toLowerCase().includes(categorySearch.value.toLowerCase())
    )
  })
  
  // Panel toggle functions
  const toggleLayerPanel = () => {
    layerPanelOpen.value = !layerPanelOpen.value
  }
  
  const toggleCategoryPanel = () => {
    categoryPanelOpen.value = !categoryPanelOpen.value
  }
  
  // Sun panel toggle function
  const toggleSunPanel = () => {
    sunPanelOpen.value = !sunPanelOpen.value
  }
  
  // Update sun position based on azimuth and elevation
  const updateSunPosition = () => {
    if (!sunLight) return
    
    // Convert spherical coordinates to Cartesian
    const azimuthRad = BABYLON.Tools.ToRadians(sunAzimuth.value)
    const elevationRad = BABYLON.Tools.ToRadians(sunElevation.value)
    
    // Calculate sun position (distance from origin)
    const distance = 100
    const x = distance * Math.cos(elevationRad) * Math.sin(azimuthRad)
    const y = distance * Math.sin(elevationRad)
    const z = distance * Math.cos(elevationRad) * Math.cos(azimuthRad)
    
    // Update sun light position and direction
    sunLight.position = new BABYLON.Vector3(x, y, z)
    sunLight.direction = new BABYLON.Vector3(-x, -y, -z).normalize()
    
    // Update sun intensity
    sunLight.intensity = sunIntensity.value
    
    // Update sun color
    const color = BABYLON.Color3.FromHexString(sunColor.value)
    sunLight.diffuse = color
    sunLight.specular = color
    
    // Update ambient light based on sun position
    if (ambientLight) {
      // Adjust ambient intensity based on sun elevation
      const ambientIntensity = Math.max(0.2, 0.6 - (sunElevation.value / 90) * 0.4)
      ambientLight.intensity = ambientIntensity
      
      // Adjust sky color based on sun position
      const skyColor = new BABYLON.Color3(
        0.2 + (sunElevation.value / 90) * 0.3,
        0.4 + (sunElevation.value / 90) * 0.4,
        0.6 + (sunElevation.value / 90) * 0.2
      )
      ambientLight.diffuse = skyColor
    }
    
    // Update shadow generator if it exists
    if (shadowGenerator) {
      shadowGenerator.forceCompilation(() => {
        // Shadow compilation complete
      })
    }
  }
  
  // Set sun preset positions
  const setSunPreset = (preset: string) => {
    switch (preset) {
      case 'dawn':
        sunAzimuth.value = 90
        sunElevation.value = 15
        sunIntensity.value = 0.8
        sunColor.value = '#FFB366'
        break
      case 'noon':
        sunAzimuth.value = 180
        sunElevation.value = 75
        sunIntensity.value = 1.5
        sunColor.value = '#FFE5B4'
        break
      case 'dusk':
        sunAzimuth.value = 270
        sunElevation.value = 15
        sunIntensity.value = 0.6
        sunColor.value = '#FF8C66'
        break
      case 'night':
        sunAzimuth.value = 0
        sunElevation.value = 0
        sunIntensity.value = 0.1
        sunColor.value = '#4A90E2'
        break
    }
    updateSunPosition()
  }
  
  // Category action functions
  const showAllCategories = () => {
    markerCategories.value.forEach(category => {
      category.visible = true
    })
  }
  
  const hideAllCategories = () => {
    markerCategories.value.forEach(category => {
      category.visible = false
    })
  }
  
  // Layer definitions
  const layers = ref<Layer[]>([
    { 
      id: 'Overworld', 
      name: 'Overworld', 
      icon: '🌍', 
      color: '#F9F449',
      description: 'The main world layer'
    },
    { 
      id: 'Underworld', 
      name: 'Underworld', 
      icon: '🕳️', 
      color: '#e6e03a',
      description: 'Underground areas and caves'
    },
    { 
      id: 'Sky', 
      name: 'Sky', 
      icon: '☁️', 
      color: '#fbf65c',
      description: 'Floating islands and sky areas'
    }
  ])
  
  // Marker categories
  const markerCategories = ref<MarkerCategory[]>([
    { name: 'Locations', icon: '📍', visible: true, count: 0 },
    { name: 'Trees', icon: '🌳', visible: false, count: 0 },
    { name: 'Obelisks', icon: '🗿', visible: false, count: 0 },
    { name: 'Ores', icon: '🪨', visible: false, count: 0 },
    { name: 'Banks', icon: '💰', visible: false, count: 0 },
    { name: 'Fires', icon: '🔥', visible: false, count: 0 },
    { name: 'Anvils', icon: '🔨', visible: false, count: 0 },
    { name: 'Furnaces', icon: '🏭', visible: false, count: 0 },
    { name: 'Kilns', icon: '⚱️', visible: false, count: 0 },
    { name: 'Stoves', icon: '🍳', visible: false, count: 0 },
    { name: 'Fishing Spots', icon: '🎣', visible: false, count: 0 },
    { name: 'Harvestables', icon: '🌾', visible: false, count: 0 },
    { name: 'Shops', icon: '🏪', visible: false, count: 0 },
    { name: 'NPCs', icon: '👤', visible: false, count: 0 },
    { name: 'Attackable NPCs', icon: '😐', visible: false, count: 0 },
    { name: 'Semi-Aggressive NPCs', icon: '😠', visible: false, count: 0 },
    { name: 'Aggressive NPCs', icon: '😈', visible: false, count: 0 }
  ])
  
  // Data storage for markers
  const levelMarkers: Record<string, Record<string, MapFeature[]>> = {
    Overworld: {},
    Underworld: {},
    Sky: {},
  }
  
  // Marker meshes storage
  const markerMeshes: Record<string, BABYLON.Mesh[]> = {
    Overworld: [],
    Underworld: [],
    Sky: []
  }
  
  // Mesh cache for loaded meshes
  const meshCache = new Map<string, BABYLON.Mesh>()
  const loadingMeshes = new Set<string>()
  
  // IndexedDB setup for mesh caching
  const DB_NAME = 'BabylonMapDB'
  const DB_VERSION = 1
  const MESH_STORE = 'meshes'
  const MTL_STORE = 'materials'
  
  let db: IDBDatabase | null = null
  
  // Initialize IndexedDB
  const initIndexedDB = (): Promise<void> => {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(DB_NAME, DB_VERSION)
      
      request.onerror = () => {
        console.error('Failed to open IndexedDB:', request.error)
        reject(request.error)
      }
      
      request.onsuccess = () => {
        db = request.result
        console.log('IndexedDB initialized successfully')
        resolve()
      }
      
      request.onupgradeneeded = (event) => {
        const database = (event.target as IDBOpenDBRequest).result
        
        // Create stores if they don't exist
        if (!database.objectStoreNames.contains(MESH_STORE)) {
          database.createObjectStore(MESH_STORE, { keyPath: 'filename' })
        }
        
        if (!database.objectStoreNames.contains(MTL_STORE)) {
          database.createObjectStore(MTL_STORE, { keyPath: 'filename' })
        }
        
        console.log('IndexedDB stores created')
      }
    })
  }
  
  // Pre-load all mesh and MTL data into IndexedDB
  const preloadMeshData = async (): Promise<void> => {
    if (!db) {
      throw new Error('IndexedDB not initialized')
    }
    
    console.log('Pre-loading mesh data into IndexedDB...')
    
    // Check if data is already loaded
    const transaction = db.transaction([MESH_STORE], 'readonly')
    const store = transaction.objectStore(MESH_STORE)
    const countRequest = store.count()
    
    const existingCount = await new Promise<number>((resolve, reject) => {
      countRequest.onsuccess = () => resolve(countRequest.result)
      countRequest.onerror = () => reject(countRequest.error)
    })
    
    if (existingCount > 0) {
      console.log(`IndexedDB already contains ${existingCount} mesh files, skipping pre-load`)
      return
    }
    
    // Process all mesh entries
    for (const meshEntry of meshesData) {
      try {
        // Skip MTL files - they should only be in the materials store
        if (meshEntry.filename.endsWith('.mtl')) {
          continue
        }
        
        // Convert base64 to blob
        const decodedData = atob(meshEntry.data.replace('data:text/plain;base64,', ''))
        const blob = new Blob([decodedData], { type: 'text/plain' })
        
        // Store mesh blob
        await storeBlob(MESH_STORE, meshEntry.filename, blob)
        
        // If this is an OBJ file, check for corresponding MTL file
        if (meshEntry.filename.endsWith('.obj')) {
          const mtlFilename = meshEntry.filename.replace('.obj', '.mtl')
          const mtlEntry = meshesData.find((entry: any) => entry.filename === mtlFilename)
          
          if (mtlEntry) {
            const mtlDecodedData = atob(mtlEntry.data.replace('data:text/plain;base64,', ''))
            const mtlBlob = new Blob([mtlDecodedData], { type: 'text/plain' })
            await storeBlob(MTL_STORE, mtlFilename, mtlBlob)
          }
        }
      } catch (error) {
        console.error(`Error storing ${meshEntry.filename}:`, error)
      }
    }
    
    console.log('Mesh data pre-loading complete')
  }
  
  // Helper function to store blob in IndexedDB
  const storeBlob = async (storeName: string, filename: string, blob: Blob): Promise<void> => {
    return new Promise((resolve, reject) => {
      const transaction = db!.transaction([storeName], 'readwrite')
      const store = transaction.objectStore(storeName)
      
      const request = store.put({
        filename,
        blob,
        timestamp: Date.now()
      })
      
      request.onsuccess = () => resolve()
      request.onerror = () => reject(request.error)
    })
  }
  
  // Generic function to get blob from IndexedDB
  const getBlob = async (storeName: string, filename: string): Promise<Blob | null> => {
    if (!db) {
      throw new Error('IndexedDB not initialized')
    }
    
    return new Promise((resolve, reject) => {
      const transaction = db!.transaction([storeName], 'readonly')
      const store = transaction.objectStore(storeName)
      const request = store.get(filename)
      
      request.onsuccess = () => {
        if (request.result) {
          resolve(request.result.blob)
        } else {
          resolve(null)
        }
      }
      
      request.onerror = () => {
        reject(request.error)
      }
    })
  }
  
  // Get mesh blob from IndexedDB
  const getMeshBlob = async (filename: string): Promise<Blob | null> => {
    return getBlob(MESH_STORE, filename)
  }
  
  // Get MTL blob from IndexedDB
  const getMTLBlob = async (filename: string): Promise<Blob | null> => {
    return getBlob(MTL_STORE, filename)
  }
  
  // Custom MTL loader for IndexedDB (unused but kept for potential future use)
  const createIndexedDBMTLLoader = () => {
    return {
      load: async (url: string): Promise<string> => {
        if (url.startsWith('indexeddb://')) {
          const filename = url.replace('indexeddb://', '')
          const mtlBlob = await getMTLBlob(filename)
          if (mtlBlob) {
            return await mtlBlob.text()
          } else {
            throw new Error(`MTL file not found in IndexedDB: ${filename}`)
          }
        } else {
          throw new Error(`Unsupported URL scheme: ${url}`)
        }
      }
    }
  }
  
  const getCurrentLayerInfo = () => {
    return layers.value.find(layer => layer.id === selectedLayer.value) || layers.value[0]
  }
  
  // Tooltip functions
  const showTooltip = (text: string, event: any) => {
    tooltipText.value = text
    tooltipX.value = event.clientX + 10
    tooltipY.value = event.clientY - 10
    tooltipVisible.value = true
  }
  
  const hideTooltip = () => {
    tooltipVisible.value = false
  }
  
  // Helper function to convert game coordinates to 3D world coordinates
  const gameToWorldCoords = (x: number, y: number): BABYLON.Vector3 => {
    // Map from [0,1024] x [0,1024] to [-50,50] x [-50,50] for 3D world
    const worldX = (x / 1024) * 100 - 50
    const worldZ = (y / 1024) * 100 - 50
    return new BABYLON.Vector3(worldX, 0, worldZ)
  }
  
  // Helper function to convert world coordinates back to game coordinates
  const worldToGameCoords = (worldPos: BABYLON.Vector3): { x: number, y: number } => {
    const gameX = ((worldPos.x + 50) / 100) * 1024
    const gameY = ((worldPos.z + 50) / 100) * 1024
    return { x: Math.round(gameX - 512.5), y: Math.round(gameY - 512.5) }
  }
  
  // Function to parse MTL data and extract material information
  const parseMTLData = (mtlData: string): Record<string, any> => {
    const materials: Record<string, any> = {}
    const lines = mtlData.split('\n')
    let currentMaterial: string | null = null
    
    for (const line of lines) {
      const trimmedLine = line.trim()
      
      if (trimmedLine.startsWith('newmtl ')) {
        // New material definition
        currentMaterial = trimmedLine.substring(7).trim()
        materials[currentMaterial] = {}
      } else if (currentMaterial && trimmedLine.startsWith('Kd ')) {
        // Diffuse color
        const values = trimmedLine.substring(3).trim().split(' ').map(Number)
        if (values.length >= 3) {
          materials[currentMaterial].Kd = values.slice(0, 3)
        }
      } else if (currentMaterial && trimmedLine.startsWith('Ks ')) {
        // Specular color
        const values = trimmedLine.substring(3).trim().split(' ').map(Number)
        if (values.length >= 3) {
          materials[currentMaterial].Ks = values.slice(0, 3)
        }
      } else if (currentMaterial && trimmedLine.startsWith('Ka ')) {
        // Ambient color
        const values = trimmedLine.substring(3).trim().split(' ').map(Number)
        if (values.length >= 3) {
          materials[currentMaterial].Ka = values.slice(0, 3)
        }
      } else if (currentMaterial && trimmedLine.startsWith('map_Kd ')) {
        // Diffuse texture
        const textureName = trimmedLine.substring(7).trim()
        materials[currentMaterial].map_Kd = textureName
      } else if (currentMaterial && trimmedLine.startsWith('Ns ')) {
        // Specular exponent
        const value = parseFloat(trimmedLine.substring(3).trim())
        if (!isNaN(value)) {
          materials[currentMaterial].Ns = value
        }
      } else if (currentMaterial && trimmedLine.startsWith('d ')) {
        // Transparency
        const value = parseFloat(trimmedLine.substring(2).trim())
        if (!isNaN(value)) {
          materials[currentMaterial].d = value
        }
      }
    }
    
    return materials
  }
  
  // Function to create BabylonJS material from MTL data
  const createBabylonMaterial = (materialName: string, mtlData: any, scene: BABYLON.Scene): BABYLON.StandardMaterial => {
    const material = new BABYLON.StandardMaterial(materialName, scene)
    
    // Apply diffuse color
    if (mtlData.Kd) {
      material.diffuseColor = new BABYLON.Color3(mtlData.Kd[0], mtlData.Kd[1], mtlData.Kd[2])
    }
    
    // Apply specular color
    if (mtlData.Ks) {
      material.specularColor = new BABYLON.Color3(mtlData.Ks[0], mtlData.Ks[1], mtlData.Ks[2])
    }
    
    // Apply ambient color
    if (mtlData.Ka) {
      material.ambientColor = new BABYLON.Color3(mtlData.Ka[0], mtlData.Ka[1], mtlData.Ka[2])
    }
    
    // Apply specular power
    if (mtlData.Ns) {
      material.specularPower = mtlData.Ns
    }
    
    // Apply transparency
    if (mtlData.d !== undefined) {
      material.alpha = mtlData.d
    }
    
    // Configure material for proper lighting
    configureMaterialForLighting(material)
    
    return material
  }
  
  // Function to load mesh from IndexedDB blob
  const loadMeshFromBlob = async (meshName: string, entityData?: any): Promise<BABYLON.Mesh | null> => {
    try {
      // Get mesh blob from IndexedDB
      const meshBlob = await getMeshBlob(meshName)
      if (!meshBlob) {
        return null
      }
      
      // For OBJ files, check if corresponding MTL file exists
      let processedData = await meshBlob.text()
      let mtlData: string | null = null
      
      if (meshName.endsWith('.obj')) {
        const mtlFileName = meshName.replace('.obj', '.mtl')
        const mtlBlob = await getMTLBlob(mtlFileName)
        
        if (mtlBlob) {
          mtlData = await mtlBlob.text()
          // Remove MTL references from OBJ content
          processedData = processedData.replace(/^mtllib\s+.*$/gm, '')
        } else {
          // Remove MTL references if no MTL file exists
          processedData = processedData.replace(/^mtllib\s+.*$/gm, '')
        }
      }
      
      // Create a blob with the processed data
      const processedBlob = new Blob([processedData], { type: 'text/plain' })
      const url = URL.createObjectURL(processedBlob)
      
      // If we have MTL data, parse it and apply materials manually
      if (mtlData) {
        const materials = parseMTLData(mtlData)
        const objBlob = new Blob([processedData], { type: 'text/plain' })
        const objUrl = URL.createObjectURL(objBlob)
        
        try {
          const result = await BABYLON.SceneLoader.ImportMeshAsync('', '', objUrl, scene, undefined, '.obj')
          URL.revokeObjectURL(objUrl)
          
          if (result.meshes.length > 0) {
            const parentMesh = new BABYLON.Mesh(meshName, scene)
            const materialMap = new Map<string, BABYLON.StandardMaterial>()
            
            // Pre-create all materials from MTL data
            for (const [materialName, mtlData] of Object.entries(materials)) {
              const babylonMaterial = createBabylonMaterial(`${meshName}_${materialName}`, mtlData, scene)
              
              if (entityData) {
                applyEntityMaterialPropertiesToMaterial(babylonMaterial, entityData, materialName)
              }
              
              materialMap.set(materialName, babylonMaterial)
              materialMap.set(`${materialName}_0`, babylonMaterial)
            }
            
            for (const mesh of result.meshes) {
              if (mesh instanceof BABYLON.Mesh) {
                mesh.parent = null
                mesh.parent = parentMesh
                
                // Try to apply material based on mesh name
                let appliedMaterial = false
                
                if (mesh.name && materialMap.has(mesh.name)) {
                  const material = materialMap.get(mesh.name)
                  if (material) {
                    mesh.material = material
                    appliedMaterial = true
                  }
                } else {
                  // Extract material name from mesh name
                  let materialName = mesh.name
                  if (mesh.name && mesh.name.includes('.')) {
                    const parts = mesh.name.split('_')
                    if (parts.length > 1) {
                      materialName = parts.slice(1).join('_')
                      materialName = materialName.replace(/_\d+$/, '')
                    }
                  } else {
                    // Handle generic mesh names
                    if (materialMap.size === 1) {
                      const material = materialMap.values().next().value
                      if (material) {
                        mesh.material = material
                        appliedMaterial = true
                      }
                    } else {
                      // Try to match by removing numbers from material names
                      for (const [mtlMaterialName, material] of materialMap.entries()) {
                        const baseMaterialName = mtlMaterialName.replace(/_\d+$/, '')
                        if (mesh.name && mesh.name.toLowerCase().includes(baseMaterialName.toLowerCase())) {
                          mesh.material = material
                          appliedMaterial = true
                          break
                        }
                      }
                    }
                  }
                  
                  if (materialMap.has(materialName)) {
                    const material = materialMap.get(materialName)
                    if (material) {
                      mesh.material = material
                      appliedMaterial = true
                    }
                  } else {
                    // Check if mesh name contains any material name
                    for (const [mtlMaterialName, material] of materialMap.entries()) {
                      if (mesh.name && mesh.name.toLowerCase().includes(mtlMaterialName.toLowerCase())) {
                        mesh.material = material
                        appliedMaterial = true
                        break
                      }
                    }
                  }
                }
                
                // Use first available material as fallback
                if (!appliedMaterial && materialMap.size > 0) {
                  const firstMaterial = materialMap.values().next().value
                  if (firstMaterial) {
                    mesh.material = firstMaterial
                    appliedMaterial = true
                  }
                }
                
                // Final fallback: create default material
                if (!appliedMaterial) {
                  const defaultMaterial = new BABYLON.StandardMaterial(`${meshName}_default_material`, scene)
                  defaultMaterial.diffuseColor = new BABYLON.Color3(0.8, 0.8, 0.8)
                  defaultMaterial.specularColor = new BABYLON.Color3(0.2, 0.2, 0.2)
                  
                  if (entityData) {
                    applyEntityMaterialPropertiesToMaterial(defaultMaterial, entityData, 'default')
                  }
                  
                  // Configure material for proper lighting
                  configureMaterialForLighting(defaultMaterial)
                  
                  mesh.material = defaultMaterial
                }
              }
            }
            
            // Reset position and rotation of the parent mesh
            parentMesh.position = new BABYLON.Vector3(0, 0, 0)
            parentMesh.rotation = new BABYLON.Vector3(0, 0, 0)
            parentMesh.scaling = new BABYLON.Vector3(0.25, 0.25, 0.25)
            
            meshCache.set(meshName, parentMesh)
            return parentMesh
          }
        } catch (error) {
          console.error(`Error loading mesh with MTL ${meshName}:`, error)
          URL.revokeObjectURL(objUrl)
        }
      }
      
      // Fallback: Load the mesh file without MTL
      const result = await BABYLON.SceneLoader.ImportMeshAsync('', '', url, scene, undefined, '.obj')
      URL.revokeObjectURL(url)
      
      if (result.meshes.length > 0) {
        const parentMesh = new BABYLON.Mesh(meshName, scene)
        
        for (const mesh of result.meshes) {
          if (mesh instanceof BABYLON.Mesh) {
            mesh.parent = null
            mesh.parent = parentMesh
            
            if (!mesh.material) {
              const material = new BABYLON.StandardMaterial(`${meshName}_fallback_material`, scene)
              material.diffuseColor = new BABYLON.Color3(0.8, 0.8, 0.8)
              material.specularColor = new BABYLON.Color3(0.2, 0.2, 0.2)
              material.ambientColor = new BABYLON.Color3(0.4, 0.4, 0.4)
              
              // Configure material for proper lighting
              configureMaterialForLighting(material)
              
              mesh.material = material
            }
          }
        }
        
        parentMesh.position = new BABYLON.Vector3(0, 0, 0)
        parentMesh.rotation = new BABYLON.Vector3(0, 0, 0)
        parentMesh.scaling = new BABYLON.Vector3(0.25, 0.25, 0.25)
        
        meshCache.set(meshName, parentMesh)
        return parentMesh
      }
      
      return null
    } catch (error) {
      console.error(`Error loading mesh ${meshName}:`, error)
      return null
    }
  }
  
  // Function to get or load a mesh
  const getOrLoadMesh = async (meshName: string, entityData?: any): Promise<BABYLON.Mesh | null> => {
    // Check if mesh is already cached
    if (meshCache.has(meshName)) {
      const cachedMesh = meshCache.get(meshName)!
      const instance = cachedMesh.clone(meshName + '_instance')
      instance.position = new BABYLON.Vector3(0, 0, 0)
      instance.rotation = new BABYLON.Vector3(0, 0, 0)
      instance.scaling = new BABYLON.Vector3(0.25, 0.25, 0.25)
      
      // Create unique materials for each child mesh to avoid sharing
      const childMeshes = instance.getChildMeshes()
      for (const childMesh of childMeshes) {
        if (childMesh instanceof BABYLON.Mesh && childMesh.material) {
          const originalMaterial = childMesh.material as BABYLON.StandardMaterial
          const uniqueMaterial = originalMaterial.clone(`${originalMaterial.name}_instance_${Date.now()}`)
          
          // Ensure the cloned material preserves all original properties
          uniqueMaterial.diffuseColor = originalMaterial.diffuseColor.clone()
          uniqueMaterial.specularColor = originalMaterial.specularColor.clone()
          uniqueMaterial.ambientColor = originalMaterial.ambientColor.clone()
          uniqueMaterial.emissiveColor = originalMaterial.emissiveColor.clone()
          uniqueMaterial.alpha = originalMaterial.alpha
          
          childMesh.material = uniqueMaterial
        }
      }
      
      return instance
    }
    
    // Check if already loading to prevent infinite loops
    if (loadingMeshes.has(meshName)) {
      await new Promise(resolve => setTimeout(resolve, 100))
      return getOrLoadMesh(meshName)
    }
    
    // Mark as loading
    loadingMeshes.add(meshName)
    
    try {
      const mesh = await loadMeshFromBlob(meshName, entityData)
      if (mesh) {
        meshCache.set(meshName, mesh)
        const instance = mesh.clone(meshName + '_instance')
        instance.position = new BABYLON.Vector3(0, 0, 0)
        instance.rotation = new BABYLON.Vector3(0, 0, 0)
        instance.scaling = new BABYLON.Vector3(0.25, 0.25, 0.25)
        
        // Create unique materials for each child mesh
        const childMeshes = instance.getChildMeshes()
        for (const childMesh of childMeshes) {
          if (childMesh instanceof BABYLON.Mesh && childMesh.material) {
            const originalMaterial = childMesh.material as BABYLON.StandardMaterial
            const uniqueMaterial = originalMaterial.clone(`${originalMaterial.name}_instance_${Date.now()}`)
            
            uniqueMaterial.diffuseColor = originalMaterial.diffuseColor.clone()
            uniqueMaterial.specularColor = originalMaterial.specularColor.clone()
            uniqueMaterial.ambientColor = originalMaterial.ambientColor.clone()
            uniqueMaterial.emissiveColor = originalMaterial.emissiveColor.clone()
            uniqueMaterial.alpha = originalMaterial.alpha
            
            childMesh.material = uniqueMaterial
          }
        }
        
        return instance
      }
      
      return null
    } finally {
      loadingMeshes.delete(meshName)
    }
  }
  
  // Function to apply entity-specific material properties to a single material
  const applyEntityMaterialPropertiesToMaterial = (material: BABYLON.StandardMaterial, entityData: any, materialName?: string) => {
    // Apply material color (mat property)
    if (entityData.mat && entityData.mat !== 'null') {
      if (entityData.mat.endsWith('.png')) {
        // Ignore texture files
        return
      } else if (entityData.mat.includes(',')) {
        // RGB values (e.g., "1,0,0")
        const rgbValues = entityData.mat.split(',').map((v: string) => parseFloat(v.trim()))
        if (rgbValues.length === 3) {
          material.diffuseColor = new BABYLON.Color3(rgbValues[0], rgbValues[1], rgbValues[2])
        }
      } else {
        // Single color value - try to parse as a single number
        const colorValue = parseFloat(entityData.mat)
        if (!isNaN(colorValue)) {
          material.diffuseColor = new BABYLON.Color3(colorValue, colorValue, colorValue)
        }
      }
    }
    
    // Apply glossiness (gloss property)
    if (entityData.gloss && entityData.gloss !== 'null') {
      if (entityData.gloss.includes(',')) {
        const glossValues = entityData.gloss.split(',').map((v: string) => parseFloat(v.trim()))
        if (glossValues.length === 3) {
          material.specularColor = new BABYLON.Color3(glossValues[0], glossValues[1], glossValues[2])
        }
      }
    }
    
    // Apply transparency (alpha property)
    if (entityData.alpha !== undefined && entityData.alpha !== null) {
      const alpha = parseFloat(entityData.alpha)
      if (alpha >= 0 && alpha <= 1) {
        material.alpha = alpha
        if (alpha < 1) {
          material.useAlphaFromDiffuseTexture = false
        }
      }
    }
    
    // Apply lighting properties (light property)
    if (entityData.light !== undefined && entityData.light !== null) {
      const lightValue = parseFloat(entityData.light)
      if (!isNaN(lightValue)) {
        const lightIntensity = Math.max(0, Math.min(1, lightValue))
        material.emissiveColor = new BABYLON.Color3(lightIntensity, lightIntensity, lightIntensity)
      }
    }
  }
  
  // Create marker mesh
  // Handles all positioning and scaling properties from worldEntities.json:
  // - x, y, z: Base position coordinates
  // - xOff, yOff: Position offsets
  // - rot: Rotation in degrees
  // - l, w, h: Length, width, height scaling (applied with base scaler of 0.25)
  const createMarkerMesh = async (position: BABYLON.Vector3, icon: string, category: string, name: string, meshName?: string, entityData?: any): Promise<BABYLON.Mesh | null> => {
    
    let markerMesh: BABYLON.Mesh
    
    // For NPCs, always use cylinders
    if (category.includes('NPC')) {
      const body = BABYLON.MeshBuilder.CreateCylinder(`body-${name}`, { height: 1.5, diameter: 0.4 }, scene)
      const head = BABYLON.MeshBuilder.CreateSphere(`head-${name}`, { diameter: 0.5 }, scene)
      head.position.y = 1
      head.parent = body
      markerMesh = body
    }
    // For entities with mesh data, try to load the encoded mesh
    else if (meshName && meshName !== 'null') {
      try {
        const loadedMesh = await getOrLoadMesh(meshName, entityData)
        if (loadedMesh) {
          markerMesh = loadedMesh
          markerMesh.position = new BABYLON.Vector3(0, 0, 0)
          markerMesh.rotation = new BABYLON.Vector3(0, 0, 0)
          markerMesh.scaling = new BABYLON.Vector3(0.25, 0.25, 0.25)
          
          // Apply entity-specific properties if available
          if (entityData) {
            // Apply rotation if specified
            if (entityData.rot && entityData.rot !== '0') {
              const rotationDegrees = parseFloat(entityData.rot)
              markerMesh.rotation.y = BABYLON.Tools.ToRadians(rotationDegrees)
            }
            
            // Apply scaling properties (l, w, h) if specified
            const baseScaler = 0.25
            let scaleX = baseScaler, scaleY = baseScaler, scaleZ = baseScaler
            
            if (entityData.l && entityData.l !== 'null') {
              scaleX = parseFloat(entityData.l) * baseScaler
            }
            if (entityData.w && entityData.w !== 'null') {
              scaleY = parseFloat(entityData.w) * baseScaler
            }
            if (entityData.h && entityData.h !== 'null') {
              scaleZ = parseFloat(entityData.h) * baseScaler
            }
            markerMesh.scaling = new BABYLON.Vector3(scaleX, scaleY, scaleZ)
            
            // Apply entity-specific material properties to existing materials
            const allMeshes = [markerMesh, ...markerMesh.getChildMeshes()]
            for (const mesh of allMeshes) {
              if (mesh instanceof BABYLON.Mesh && mesh.material) {
                applyEntityMaterialPropertiesToMaterial(mesh.material as BABYLON.StandardMaterial, entityData, mesh.name)
              }
            }
          }
        } else {
          return null
        }
      } catch (error) {
        console.error(`Error loading mesh ${meshName} for ${name}:`, error)
        return null
      }
    }
    // For other categories, skip if no mesh data
    else {
      return null
    }
    
    // Apply positioning with offsets if specified
    let finalPosition = position.clone()
    if (entityData) {
      // Apply xOff offset if specified
      if (entityData.xOff && entityData.xOff !== 'null') {
        const xOffset = parseFloat(entityData.xOff)
        finalPosition.x += xOffset
        console.log(`Applied x offset ${xOffset} to ${name}`)
      }
      
      // Apply yOff offset if specified
      if (entityData.yOff && entityData.yOff !== 'null') {
        const yOffset = parseFloat(entityData.yOff)
        finalPosition.z += yOffset // Note: yOff affects Z in 3D space
        console.log(`Applied y offset ${yOffset} to ${name}`)
      }
    }
    
    markerMesh.position = finalPosition
    
    // Get the height at this position from the heightmap
    const ground = scene.getMeshByName('ground')
    if (ground) {
      // Try multiple raycast approaches for better accuracy
      let terrainHeight = null
      
      // First try: direct raycast from above
      const ray = new BABYLON.Ray(
        new BABYLON.Vector3(finalPosition.x, 50, finalPosition.z), // Start from high above
        new BABYLON.Vector3(0, -1, 0)
      )
      const hit = scene.pickWithRay(ray)
      if (hit && hit.pickedMesh === ground && hit.pickedPoint) {
        terrainHeight = hit.pickedPoint.y
      } else {
        // Second try: sample multiple points around the position
        const sampleRadius = 2
        const samplePoints = [
          { x: finalPosition.x, z: finalPosition.z },
          { x: finalPosition.x + sampleRadius, z: finalPosition.z },
          { x: finalPosition.x - sampleRadius, z: finalPosition.z },
          { x: finalPosition.x, z: finalPosition.z + sampleRadius },
          { x: finalPosition.x, z: finalPosition.z - sampleRadius }
        ]
        
        let totalHeight = 0
        let validSamples = 0
        
        for (const point of samplePoints) {
          const sampleRay = new BABYLON.Ray(
            new BABYLON.Vector3(point.x, 50, point.z),
            new BABYLON.Vector3(0, -1, 0)
          )
          const sampleHit = scene.pickWithRay(sampleRay)
          if (sampleHit && sampleHit.pickedMesh === ground && sampleHit.pickedPoint) {
            totalHeight += sampleHit.pickedPoint.y
            validSamples++
          }
        }
        
        if (validSamples > 0) {
          terrainHeight = totalHeight / validSamples
        }
      }
      
      // Set marker height based on terrain
      if (terrainHeight !== null) {
        // Calculate the mesh bounds to determine its height
        const boundingBox = markerMesh.getHierarchyBoundingVectors()
        const meshHeight = boundingBox.max.y - boundingBox.min.y
        const bottomOffset = meshHeight / 2
        
        // Position the mesh so its bottom touches the terrain
        markerMesh.position.y = terrainHeight + bottomOffset
      } else {
        // Calculate the mesh bounds to determine its height
        const boundingBox = markerMesh.getHierarchyBoundingVectors()
        const meshHeight = boundingBox.max.y - boundingBox.min.y
        const bottomOffset = meshHeight / 2
        
        // Position the mesh so its bottom touches the ground level
        markerMesh.position.y = bottomOffset
      }
    } else {
      // Calculate the mesh bounds to determine its height
      const boundingBox = markerMesh.getHierarchyBoundingVectors()
      const meshHeight = boundingBox.max.y - boundingBox.min.y
      const bottomOffset = meshHeight / 2
      
      // Position the mesh so its bottom touches the ground level
      markerMesh.position.y = bottomOffset
    }
    
    // Only apply the red material if the mesh doesn't already have a material
    let markerMaterial: BABYLON.StandardMaterial | null = null
    if (!markerMesh.material) {
      markerMaterial = new BABYLON.StandardMaterial(`material-${name}`, scene)
      markerMaterial.diffuseColor = new BABYLON.Color3(1, 0, 0)
      markerMaterial.emissiveColor = new BABYLON.Color3(0.3, 0, 0)
      
      // Configure material for proper lighting
      configureMaterialForLighting(markerMaterial)
      
      markerMesh.material = markerMaterial
    } else {
      if (markerMesh.material instanceof BABYLON.StandardMaterial) {
        markerMaterial = markerMesh.material
        // Ensure existing materials are also configured for lighting
        configureMaterialForLighting(markerMaterial)
      }
    }
    
  
    
    // Ensure marker is visible and not culled
    markerMesh.isPickable = true
    markerMesh.visibility = 1
    
    // Add hover effect with tooltip
    markerMesh.actionManager = new BABYLON.ActionManager(scene)
    // Store original scaling for hover effects
    const originalScaling = markerMesh.scaling.clone()
    
    markerMesh.actionManager.registerAction(
      new BABYLON.ExecuteCodeAction(
        BABYLON.ActionManager.OnPointerOverTrigger,
        () => {
          // Scale up by 20% for hover effect while preserving custom scaling
          markerMesh.scaling = originalScaling.scale(1.2)
          if (markerMaterial) {
            markerMaterial.emissiveColor = new BABYLON.Color3(0.3, 0.3, 0.3)
          }
          
          // Show tooltip at mouse position
          const canvas = scene.getEngine().getRenderingCanvas()
          if (canvas) {
            const rect = canvas.getBoundingClientRect()
            const mouseX = rect.left + scene.pointerX
            const mouseY = rect.top + scene.pointerY
            showTooltip(`${icon} ${name}`, { clientX: mouseX, clientY: mouseY })
          }
        }
      )
    )
    markerMesh.actionManager.registerAction(
      new BABYLON.ExecuteCodeAction(
        BABYLON.ActionManager.OnPointerOutTrigger,
        () => {
          // Restore original scaling
          markerMesh.scaling = originalScaling
          if (markerMaterial) {
            markerMaterial.emissiveColor = new BABYLON.Color3(0, 0, 0)
          }
          
          // Hide tooltip
          hideTooltip()
        }
      )
    )
    
    // Add click action
    markerMesh.actionManager.registerAction(
      new BABYLON.ExecuteCodeAction(
        BABYLON.ActionManager.OnPickTrigger,
        () => {
          console.log(`Clicked on ${name} (${category})`)
          // Fly to marker
          camera.radius = 8
          camera.target = finalPosition
        }
      )
    )
    
    // Create a parent mesh to group the main marker and backup sphere
    const markerGroup = new BABYLON.Mesh(`group-${name}`, scene)
    
    // Remove marker mesh and all its descendants from scene root
    const removeMeshAndDescendants = (mesh: BABYLON.Mesh) => {
      // First remove all children recursively
      const children = mesh.getChildMeshes()
      children.forEach(child => {
        if (child.parent === null) {
          // Only remove if it's at scene root
          child.parent = markerGroup
        }
      })
      // Then remove the main mesh
      if (mesh.parent === null) {
        mesh.parent = markerGroup
      }
    }
    
    removeMeshAndDescendants(markerMesh)
    
    // Add marker to shadow system if shadow generator exists
    if (shadowGenerator) {
      // Add the main marker group as shadow caster
      shadowGenerator.addShadowCaster(markerGroup, true)
      
      // Also add all child meshes individually
      const childMeshes = markerGroup.getChildMeshes()
      for (const childMesh of childMeshes) {
        if (childMesh instanceof BABYLON.Mesh) {
          shadowGenerator!.addShadowCaster(childMesh, true)
        }
      }
      
      // Force shadow generator to update
      shadowGenerator.forceCompilation(() => {
        console.log(`Added ${markerGroup.name} to shadow system with ${childMeshes.length} child meshes`)
      })
    } else {
      console.warn('Shadow generator not available for', markerGroup.name)
    }
    
    addMeshAndChildrenToShadows(markerGroup)
    
    if (!category.includes('NPC')) {
      // Use a color based on category (for demo, ores = blue, trees = green, etc)
      let baseColor = new BABYLON.Color3(0.2, 0.6, 1.0); // default blue
      if (category === 'Ores') baseColor = new BABYLON.Color3(0.2, 0.6, 1.0);
      else if (category === 'Trees') baseColor = new BABYLON.Color3(0.2, 0.8, 0.2);
      else if (category === 'Obelisks') baseColor = new BABYLON.Color3(0.7, 0.7, 0.7);
      else if (category === 'Banks') baseColor = new BABYLON.Color3(0.9, 0.8, 0.2);
      // ...add more as desired
      const toonMat = createToonMaterial(scene, baseColor);
      // Set light direction from sun if available
      if (sunLight) {
        toonMat.setVector3("lightDirection", sunLight.direction);
      }
      markerMesh.material = toonMat;
      markerMesh.getChildMeshes().forEach(child => {
        child.material = toonMat;
      });
    }
    
    return markerGroup
  }
  
  // Create ground plane for the map
  const createGround = () => {
    // Get heightmap data for current layer
    const heightmapData = getHeightmapData(selectedLayer.value)
    
    let ground: BABYLON.Mesh
    
    if (heightmapData) {
      // Create heightmap-based terrain
      const heightmapTexture = new BABYLON.Texture(heightmapData, scene)
      
      // Create a ground with heightmap displacement
      ground = BABYLON.MeshBuilder.CreateGroundFromHeightMap(
        'ground',
        heightmapData,
        {
          width: 100,
          height: 100,
          subdivisions: 100,
          minHeight: 0,
          maxHeight: 10,
          onReady: async () => {
            // Create a material that properly handles transparency
            const groundMaterial = new BABYLON.StandardMaterial('groundMaterial', scene)
            
            // Merge base texture and overlay
            const mergedTextureData = await mergeTextures(selectedLayer.value)
            if (mergedTextureData) {
              const texture = new BABYLON.Texture(mergedTextureData, scene)
              groundMaterial.diffuseTexture = texture
            }
            
            groundMaterial.specularColor = new BABYLON.Color3(0, 0, 0)
            groundMaterial.ambientColor = new BABYLON.Color3(0.8, 0.8, 0.8)
            ground.material = groundMaterial
            ground.receiveShadows = true
            
            // Configure ground material for proper lighting
            configureMaterialForLighting(groundMaterial)
          }
        },
        scene
      )
    } else {
      // Fallback to flat ground if no heightmap
      ground = BABYLON.MeshBuilder.CreateGround('ground', { width: 100, height: 100 }, scene)
      
      // Create ground material with texture and overlay
      const groundMaterial = new BABYLON.StandardMaterial('groundMaterial', scene)
      
      // Base texture
      const textureData = getMapTextureData(selectedLayer.value)
      if (textureData) {
        const texture = new BABYLON.Texture(textureData, scene)
        groundMaterial.diffuseTexture = texture
      }
      
      // Map overlay - blend with base texture
      const overlayData = getMapOverlayData(selectedLayer.value)
      if (overlayData) {
        const overlay = new BABYLON.Texture(overlayData, scene)
        groundMaterial.emissiveTexture = overlay
        groundMaterial.emissiveColor = new BABYLON.Color3(0.2, 0.2, 0.2)
      }
      
      groundMaterial.specularColor = new BABYLON.Color3(0, 0, 0)
      
      ground.material = groundMaterial
      ground.receiveShadows = true
      
      // Configure ground material for proper lighting
      configureMaterialForLighting(groundMaterial)
    }
    
    return ground
  }
  
  // Helper function to get map texture data based on layer
  const getMapTextureData = (layer: string): string | null => {
    const textureFilename = (() => {
      switch (layer) {
        case 'Overworld':
          return 'earthoverworldtexture.png'
        case 'Underworld':
          return 'earthundergroundtexture.png'
        case 'Sky':
          return 'earthskytexture.png'
        default:
          return 'earthoverworldtexture.png'
      }
    })()
    
    const textureEntry = (heightmapsData as any[]).find((entry: any) => entry.filename === textureFilename)
    return textureEntry ? textureEntry.data : null
  }
  
  // Helper function to get map overlay data based on layer
  const getMapOverlayData = (layer: string): string | null => {
    const overlayFilename = (() => {
      switch (layer) {
        case 'Overworld':
          return 'earthoverworldmap.png'
        case 'Underworld':
          return 'earthundergroundmap.png'
        case 'Sky':
          return 'earthskymap.png'
        default:
          return 'earthoverworldmap.png'
      }
    })()
    
    const overlayEntry = (heightmapsData as any[]).find((entry: any) => entry.filename === overlayFilename)
    return overlayEntry ? overlayEntry.data : null
  }
  
  // Helper function to get map path data based on layer
  const getMapPathData = (layer: string): string | null => {
    const pathFilename = (() => {
      switch (layer) {
        case 'Overworld':
          return 'earthoverworldpath.png'
        case 'Underworld':
          return 'earthundergroundpath.png'
        case 'Sky':
          return 'earthskypath.png'
        default:
          return 'earthoverworldpath.png'
      }
    })()
    
    const pathEntry = (heightmapsData as any[]).find((entry: any) => entry.filename === pathFilename)
    return pathEntry ? pathEntry.data : null
  }
  
  // Helper function to get map cement data based on layer
  const getMapCementData = (layer: string): string | null => {
    const cementFilename = (() => {
      switch (layer) {
        case 'Overworld':
          return 'earthoverworldcement.png'
        case 'Underworld':
          return 'earthundergroundcement.png'
        case 'Sky':
          return null // Sky doesn't have cement
        default:
          return 'earthoverworldcement.png'
      }
    })()
    
    if (!cementFilename) return null
    const cementEntry = (heightmapsData as any[]).find((entry: any) => entry.filename === cementFilename)
    return cementEntry ? cementEntry.data : null
  }
  
  // Helper function to get heightmap data based on layer
  const getHeightmapData = (layer: string): string | null => {
    const heightmapFilename = (() => {
      switch (layer) {
        case 'Overworld':
          return 'earthoverworldheightmap3.png'
        case 'Underworld':
          return 'earthundergroundheightmap.png'
        case 'Sky':
          return 'earthskyheightmap.png'
        default:
          return 'earthoverworldheightmap3.png'
      }
    })()
    
    const heightmapEntry = (heightmapsData as any[]).find((entry: any) => entry.filename === heightmapFilename)
    return heightmapEntry ? heightmapEntry.data : null
  }
  
  // Helper function to get all available heightmap data
  const getAllHeightmapData = () => {
    const heightmaps = ['earthoverworldheightmap3.png', 'earthundergroundheightmap.png', 'earthskyheightmap.png']
    return heightmaps.map(filename => {
      const entry = (heightmapsData as any[]).find((entry: any) => entry.filename === filename)
      return entry ? entry.data : null
    }).filter(data => data !== null)
  }
  
  // Helper function to get all available map overlay data
  const getAllMapOverlayData = () => {
    const maps = ['earthoverworldmap.png', 'earthundergroundmap.png', 'earthskymap.png']
    return maps.map(filename => {
      const entry = (heightmapsData as any[]).find((entry: any) => entry.filename === filename)
      return entry ? entry.data : null
    }).filter(data => data !== null)
  }
  
  // Helper function to get all available texture data
  const getAllTextureData = () => {
    const textures = ['earthoverworldtexture.png', 'earthundergroundtexture.png', 'earthskytexture.png']
    return textures.map(filename => {
      const entry = (heightmapsData as any[]).find((entry: any) => entry.filename === filename)
      return entry ? entry.data : null
    }).filter(data => data !== null)
  }
  
  // Helper function to merge base texture and overlay
  const mergeTextures = async (layer: string): Promise<string | null> => {
    const baseTextureData = getMapTextureData(layer)
    const overlayData = getMapOverlayData(layer)
    
    if (!baseTextureData || !overlayData) {
      return baseTextureData || overlayData
    }
    
    // Create canvas to merge images
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d')
    if (!ctx) return baseTextureData
    
    // Create images from base64 data
    const baseImg = new Image()
    const overlayImg = new Image()
    
    return new Promise<string>((resolve) => {
      let loadedCount = 0
      const checkComplete = () => {
        loadedCount++
        if (loadedCount === 2) {
          // Set canvas size to match the larger image
          const maxWidth = Math.max(baseImg.width, overlayImg.width)
          const maxHeight = Math.max(baseImg.height, overlayImg.height)
          canvas.width = maxWidth
          canvas.height = maxHeight
          
          // Draw base texture first (as background) - scale to fit canvas
          ctx.drawImage(baseImg, 0, 0, maxWidth, maxHeight)
          
          // Draw overlay on top - scale to fit canvas
          ctx.globalCompositeOperation = 'source-over'
          ctx.drawImage(overlayImg, 0, 0, maxWidth, maxHeight)
          
          // Convert back to base64
          const mergedData = canvas.toDataURL('image/png')
          resolve(mergedData)
        }
      }
      
      baseImg.onload = checkComplete
      overlayImg.onload = checkComplete
      
      baseImg.src = baseTextureData
      overlayImg.src = overlayData
    })
  }
  
  // Create sun-based lighting system
  const createLighting = () => {
    // Primary ambient light for overall illumination
    ambientLight = new BABYLON.HemisphericLight('ambientLight', new BABYLON.Vector3(0, 1, 0), scene)
    ambientLight.intensity = 0.6
    ambientLight.diffuse = new BABYLON.Color3(0.9, 0.9, 1.0) // Slightly blue-tinted for sky
    ambientLight.specular = new BABYLON.Color3(0.2, 0.2, 0.3)
    ambientLight.groundColor = new BABYLON.Color3(0.4, 0.4, 0.3) // Warm ground reflection
    
    // Create configurable sun light
    sunLight = new BABYLON.DirectionalLight('sunLight', new BABYLON.Vector3(-0.5, -1, -0.5), scene)
    sunLight.intensity = sunIntensity.value
    
    // Configure shadows for sun light
    sunLight.shadowMinZ = 0.1
    sunLight.shadowMaxZ = 100
    sunLight.shadowFrustumSize = 120
    sunLight.shadowOrthoScale = 0.1
    sunLight.autoUpdateExtends = true
    
    // Set initial sun position
    updateSunPosition()
    
    return { ambientLight, sunLight }
  }
  
  // Function to configure material for proper lighting
  const configureMaterialForLighting = (material: BABYLON.StandardMaterial) => {
    // Ensure material can receive shadows
    material.backFaceCulling = false
    material.useParallax = false
    material.useParallaxOcclusion = false
    
    // Enable proper lighting calculations
    material.useLightmapAsShadowmap = false
    material.useEmissiveAsIllumination = false
    
    // Set reasonable defaults for lighting
    if (!material.diffuseColor) {
      material.diffuseColor = new BABYLON.Color3(0.8, 0.8, 0.8)
    }
    if (!material.specularColor) {
      material.specularColor = new BABYLON.Color3(0.2, 0.2, 0.2)
    }
    if (!material.ambientColor) {
      material.ambientColor = new BABYLON.Color3(0.4, 0.4, 0.4)
    }
    
    // Ensure material responds to lighting changes
    material.markAsDirty(BABYLON.Constants.MATERIAL_AllDirtyFlag)
  }
  
  // Function to add mesh to shadow generator
  const addMeshToShadows = (mesh: BABYLON.Mesh) => {
    if (shadowGenerator) {
      shadowGenerator.addShadowCaster(mesh, true)
      
      // Also add all child meshes
      const childMeshes = mesh.getChildMeshes()
      for (const childMesh of childMeshes) {
        if (childMesh instanceof BABYLON.Mesh) {
          shadowGenerator.addShadowCaster(childMesh, true)
        }
      }
    }
  }
  
  // Load and process map data
  const loadMapData = () => {
    // Process locations
    locations.locations.forEach((location: any) => {
      const feature = {
        coordinates: gameToWorldCoords(location.x + 512.5, location.y + 512.5),
        name: location.name,
        icon: '📍',
        category: 'Locations',
        isLocation: true
      }
      
      switch (location.labelType) {
        case 0:
          if (!levelMarkers.Underworld.Locations) levelMarkers.Underworld.Locations = []
          levelMarkers.Underworld.Locations.push(feature)
          break
        case 1:
          if (!levelMarkers.Overworld.Locations) levelMarkers.Overworld.Locations = []
          levelMarkers.Overworld.Locations.push(feature)
          break
        case 2:
          if (!levelMarkers.Sky.Locations) levelMarkers.Sky.Locations = []
          levelMarkers.Sky.Locations.push(feature)
          break
      }
    })
  
    // Process entities
    const worldEntities = (entitiesData as any).worldEntities || []
    
    worldEntities.forEach((entity: any) => {
      const position = gameToWorldCoords(entity.x + 512.5, entity.z + 512.5)
      let category = 'Unknown'
      let icon = '❓'
      
      // Determine category and icon based on entity type
      if (entity.type.includes('tree') || entity.type.includes('cherryblossom')) {
        category = 'Trees'
        icon = '🌳'
      } else if (entity.type.includes('obelisk')) {
        category = 'Obelisks'
        icon = '🗿'
      } else if (entity.type.includes('rocks')) {
        category = 'Ores'
        icon = '🪨'
      } else if (entity.type.includes('bank')) {
        category = 'Banks'
        icon = '💰'
      } else if (entity.type.includes('fire')) {
        category = 'Fires'
        icon = '🔥'
      } else if (entity.type.includes('smithingsource')) {
        category = 'Anvils'
        icon = '🔨'
      } else if (entity.type.includes('smeltingsource')) {
        category = 'Furnaces'
        icon = '🏭'
      } else if (entity.type.includes('kiln')) {
        category = 'Kilns'
        icon = '⚱️'
      } else if (entity.type.includes('heatsource')) {
        category = 'Stoves'
        icon = '🍳'
      } else if (entity.type.includes('fishing')) {
        category = 'Fishing Spots'
        icon = '🎣'
      } else if (entity.type.includes('pumpkin') || entity.type.includes('corn') || 
                 entity.type.includes('potatoes') || entity.type.includes('onion') ||
                 entity.type.includes('flax') || entity.type.includes('carrot') ||
                 entity.type.includes('redmushroom') || entity.type.includes('plant') ||
                 entity.type.includes('strawberries') || entity.type.includes('watermelon')) {
        category = 'Harvestables'
        icon = '🌾'
      }
      
      const feature = {
        coordinates: position,
        name: entity.type.replace(/([a-z])([A-Z])/g, '$1 $2').replace(/^\w/, (c: string) => c.toUpperCase()),
        icon,
        category
      }
      
      const layerMap = { 0: 'Underworld', 1: 'Overworld', 2: 'Sky' }
      const layerName = layerMap[entity.lvl as keyof typeof layerMap]
      if (layerName) {
        if (!levelMarkers[layerName][category]) levelMarkers[layerName][category] = []
        levelMarkers[layerName][category].push(feature)
      }
    })
  
    // Process NPCs
    npcs.npcs.forEach((npc: any) => {
      const npcDef = npcDefinitions.npcDefs.find((def: any) => npc.npcdef_id === def._id) as any
      if (!npcDef) return
  
      const position = gameToWorldCoords(npc.x + 512.5, npc.y + 512.5)
      let category = 'NPCs'
      let icon = '👤'
      
      if (npcDef.shopdef_id) {
        category = 'Shops'
        icon = '🏪'
      } else if (npcDef.combat) {
        if (npcDef.combat.isAlwaysAggro) {
          category = 'Aggressive NPCs'
          icon = '😈'
        } else if (npcDef.combat.aggroRadius >= 1) {
          category = 'Semi-Aggressive NPCs'
          icon = '😠'
        } else {
          category = 'Attackable NPCs'
          icon = '😐'
        }
      }
      
      const feature = {
        coordinates: position,
        name: npcDef.name?.replace(/(?:^|\s)\S/g, (a: string) => a.toUpperCase()) || 'Unknown NPC',
        icon,
        category
      }
      
      const layerMap = { 0: 'Underworld', 1: 'Overworld', 2: 'Sky' }
      const layerName = layerMap[npc.mapLevel as keyof typeof layerMap]
      if (layerName) {
        if (!levelMarkers[layerName][category]) levelMarkers[layerName][category] = []
        levelMarkers[layerName][category].push(feature)
      }
    })
  }
  
  // Update marker counts
  const updateMarkerCounts = () => {
    markerCategories.value.forEach(category => {
      const currentLayerMarkers = levelMarkers[selectedLayer.value]
      const newCount = currentLayerMarkers[category.name]?.length || 0
      category.count = newCount
    })
  }
  
  // Render markers for current layer and visible categories
  let isRendering = false
  const renderMarkers = async (specificCategory?: string) => {
    if (isRendering) {
      return
    }
    
    isRendering = true
    
    // Start mesh loading state
    isMeshLoading.value = true
    loadedMeshesCount.value = 0
    meshLoadingProgress.value = 0
    
    // If a specific category is provided, only handle that category
    if (specificCategory) {
      // Remove existing markers for this category
      const categoryMeshes = markerMeshes[selectedLayer.value].filter(mesh => 
        mesh.name && mesh.name.includes(specificCategory)
      )
      categoryMeshes.forEach(mesh => mesh.dispose())
      markerMeshes[selectedLayer.value] = markerMeshes[selectedLayer.value].filter(mesh => 
        !mesh.name || !mesh.name.includes(specificCategory)
      )
      
      // Add new markers for this category if visible
      const category = markerCategories.value.find(cat => cat.name === specificCategory)
      if (category && category.visible && levelMarkers[selectedLayer.value][specificCategory]) {
        const features = levelMarkers[selectedLayer.value][specificCategory]
        totalMeshesToLoad.value = features.length
        
        for (let index = 0; index < features.length; index++) {
          const feature = levelMarkers[selectedLayer.value][specificCategory][index]
          try {
            // Get mesh name and entity data from entities data if this is an entity
            let meshName: string | undefined
            let entityData: any = null
            if (feature.category !== 'Locations' && !feature.category.includes('NPC')) {
              const worldEntities = (entitiesData as any).worldEntities || []
              const featureGameCoords = worldToGameCoords(feature.coordinates)
              
              const entity = worldEntities.find((e: any) => {
                const distance = Math.sqrt(
                  Math.pow(e.x - featureGameCoords.x, 2) + 
                  Math.pow(e.z - featureGameCoords.y, 2)
                )
                return distance < 0.1
              })
              if (entity && entity.mesh && entity.mesh !== 'null') {
                meshName = entity.mesh
                entityData = entity
              }
            }
            
            const markerMesh = await createMarkerMesh(
              feature.coordinates,
              feature.icon,
              feature.category,
              feature.name,
              meshName,
              entityData
            )
                          if (markerMesh) {
                markerMeshes[selectedLayer.value].push(markerMesh)
                addMeshAndChildrenToShadows(markerMesh)
              }
            
            // Update progress
            loadedMeshesCount.value++
            meshLoadingProgress.value = Math.round((loadedMeshesCount.value / totalMeshesToLoad.value) * 100)
          } catch (error) {
            console.error(`Error creating marker for ${feature.name}:`, error)
          }
        }
      }
    } else {
      // Clear all existing markers (for layer changes)
      markerMeshes[selectedLayer.value].forEach((mesh) => {
        mesh.dispose()
      })
      markerMeshes[selectedLayer.value] = []
      
      const currentLayerMarkers = levelMarkers[selectedLayer.value]
      
      // Calculate total meshes to load
      let totalMeshes = 0
      for (const category of markerCategories.value) {
        if (category.visible && currentLayerMarkers[category.name]) {
          totalMeshes += currentLayerMarkers[category.name].length
        }
      }
      totalMeshesToLoad.value = totalMeshes
      
      for (const category of markerCategories.value) {
        if (category.visible && currentLayerMarkers[category.name]) {
          for (let index = 0; index < currentLayerMarkers[category.name].length; index++) {
            const feature = currentLayerMarkers[category.name][index]
            try {
              // Get mesh name and entity data from entities data if this is an entity
              let meshName: string | undefined
              let entityData: any = null
              if (feature.category !== 'Locations' && !feature.category.includes('NPC')) {
                const worldEntities = (entitiesData as any).worldEntities || []
                const featureGameCoords = worldToGameCoords(feature.coordinates)
                
                const entity = worldEntities.find((e: any) => {
                  const distance = Math.sqrt(
                    Math.pow(e.x - featureGameCoords.x, 2) + 
                    Math.pow(e.z - featureGameCoords.y, 2)
                  )
                  return distance < 0.1
                })
                if (entity && entity.mesh && entity.mesh !== 'null') {
                  meshName = entity.mesh
                  entityData = entity
                }
              }
              
              const markerMesh = await createMarkerMesh(
                feature.coordinates,
                feature.icon,
                feature.category,
                feature.name,
                meshName,
                entityData
              )
              if (markerMesh) {
                markerMeshes[selectedLayer.value].push(markerMesh)
                addMeshAndChildrenToShadows(markerMesh)
              }
              
              // Update progress
              loadedMeshesCount.value++
              meshLoadingProgress.value = Math.round((loadedMeshesCount.value / totalMeshesToLoad.value) * 100)
            } catch (error) {
              console.error(`Error creating marker for ${feature.name}:`, error)
            }
          }
        }
      }
    }
    
    // Hide mesh loading overlay
    isMeshLoading.value = false
    isRendering = false
  }
  
  // Layer switching
  const switchLayer = async (layerId: string) => {
    selectedLayer.value = layerId
    updateMarkerCounts()
    await renderMarkers()
    
    // Recreate ground with new heightmap and texture
    const oldGround = scene.getMeshByName('ground')
    if (oldGround) {
      oldGround.dispose()
    }
    
    const newGround = createGround()
    
    // Update camera to better view the new terrain
    camera.radius = 80
    camera.alpha = 0
    camera.beta = Math.PI / 4
  }
  
  // Category toggling
  const toggleCategory = async (categoryName: string) => {
    const category = markerCategories.value.find(cat => cat.name === categoryName)
    if (category) {
      category.visible = !category.visible
      await renderMarkers(categoryName)
    }
  }
  
  // Initialize BabylonJS scene
  const initBabylonScene = async () => {
    canvas = document.querySelector('.map-canvas') as HTMLCanvasElement
    
    // Initialize IndexedDB first
    await initIndexedDB()
    
    // Pre-load mesh data into IndexedDB
    await preloadMeshData()
    
    // Create engine
    engine = new BABYLON.Engine(canvas, true, { preserveDrawingBuffer: true, antialias: true })
    
    // Create scene
    scene = new BABYLON.Scene(engine)
    scene.clearColor = new BABYLON.Color4(0.2, 0.4, 0.6, 1)
    
    // Create camera
    camera = new BABYLON.ArcRotateCamera('camera', 0, Math.PI / 4, 80, BABYLON.Vector3.Zero(), scene)
    camera.attachControl(canvas, true)
    camera.lowerRadiusLimit = 5
    camera.upperRadiusLimit = 150
    camera.wheelDeltaPercentage = 0.01
    camera.minZ = 0.1
    camera.maxZ = 1000
    
    // Create ground
    const ground = createGround()
    
    // Create lighting
    const { ambientLight, sunLight } = createLighting()
    
    // Enable shadows with improved quality
    shadowGenerator = new BABYLON.ShadowGenerator(2048, sunLight)
    shadowGenerator.useBlurExponentialShadowMap = true
    shadowGenerator.blurKernel = 64
    shadowGenerator.bias = 0.00001
    shadowGenerator.normalBias = 0.02
    
    // Add ground to shadow system
    const groundMesh = scene.getMeshByName('ground')
    if (groundMesh) {
      groundMesh.receiveShadows = true
    }
    
    // Load map data
    loadMapData()
    
    // Update marker counts and render initial markers
    updateMarkerCounts()
    await renderMarkers()
    
    // Add all existing meshes to shadow system
    const addAllMeshesToShadows = () => {
      if (shadowGenerator) {
        // Add all marker meshes to shadow system
        for (const layer in markerMeshes) {
          markerMeshes[layer].forEach(mesh => {
            shadowGenerator!.addShadowCaster(mesh, true)
            const childMeshes = mesh.getChildMeshes()
            for (const childMesh of childMeshes) {
              if (childMesh instanceof BABYLON.Mesh) {
                shadowGenerator!.addShadowCaster(childMesh, true)
              }
            }
          })
        }
        
        // Force shadow compilation
        shadowGenerator.forceCompilation(() => {
          console.log('All meshes added to shadow system')
        })
      }
    }
    
    // Add meshes to shadows after a short delay to ensure they're all loaded
    setTimeout(addAllMeshesToShadows, 1000)
    
    // Mouse interaction for coordinates
    scene.onPointerMove = (evt) => {
      const pickResult = scene.pick(scene.pointerX, scene.pointerY)
      if (pickResult.hit && pickResult.pickedMesh === groundMesh && pickResult.pickedPoint) {
        const worldPos = pickResult.pickedPoint
        const gameCoords = worldToGameCoords(worldPos)
        mouseCoords.x = gameCoords.x
        mouseCoords.y = gameCoords.y
      }
    }
    
    // Start rendering loop
    engine.runRenderLoop(() => {
      scene.render()
    })
    
    // Handle window resize
    window.addEventListener('resize', () => {
      engine.resize()
    })
    
    // Set loading to false
    setTimeout(() => {
      isLoading.value = false
    }, 1000)
  }
  
  onMounted(async () => {
    await initBabylonScene()
  })
  
  onUnmounted(() => {
    if (engine) {
      engine.dispose()
    }
  })
  
  // Watch for layer changes
  watch(selectedLayer, async (newLayer, oldLayer) => {
    if (newLayer === oldLayer) {
      return
    }
    
    updateMarkerCounts()
    await renderMarkers()
    
    // Recreate ground with new heightmap and texture
    const oldGround = scene.getMeshByName('ground')
    if (oldGround) {
      oldGround.dispose()
    }
    
    createGround()
    
    // Update camera to better view the new terrain
    camera.radius = 80
    camera.alpha = 0
    camera.beta = Math.PI / 4
  })
  </script>
  
  <style scoped>
  .babylon-map-container {
    position: relative;
    width: 100%;
    height: 100vh;
    overflow: hidden;
    outline: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }
  
  .map-canvas {
    width: 100%;
    height: 100%;
    display: block;
    outline: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
    -webkit-tap-highlight-color: transparent;
  }
  
  .map-ui-overlay {
    position: absolute;
    top: 20px;
    right: 20px;
    z-index: 1000;
    display: flex;
    flex-direction: column;
    gap: 12px;
    width: 280px;
  }
  
  .control-panel {
    display: flex;
    flex-direction: column;
    background: rgba(0, 0, 0, 0.85);
    border-radius: 8px;
    backdrop-filter: blur(12px);
    border: 1px solid rgba(255, 255, 255, 0.1);
    margin-bottom: 8px;
    overflow: hidden;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
    max-width: 280px;
  }
  
     .panel-header {
     display: flex;
     align-items: center;
     justify-content: space-between;
     cursor: pointer;
     color: white;
     padding: 10px 12px;
     background: rgba(255, 255, 255, 0.05);
     border-bottom: 1px solid rgba(255, 255, 255, 0.1);
     transition: background-color 0.2s ease;
   }
   
   .panel-header:hover {
     background: rgba(255, 255, 255, 0.1);
   }
   
   .panel-title {
     font-size: 14px;
     font-weight: 600;
     color: #ffffff;
   }
   
   .panel-toggle {
     font-size: 16px;
     font-weight: bold;
     color: #4CAF50;
     transition: transform 0.2s ease;
   }
   
   .panel-content {
     display: flex;
     flex-direction: column;
     gap: 6px;
     padding: 10px 12px;
   }
  
     .layer-button {
     display: flex;
     align-items: center;
     gap: 8px;
     padding: 8px 10px;
     border-radius: 6px;
     cursor: pointer;
     transition: all 0.2s ease;
     color: white;
     border: 1px solid transparent;
   }
   
   .layer-button:hover {
     background: rgba(255, 255, 255, 0.1);
     border-color: rgba(255, 255, 255, 0.2);
   }
   
   .layer-button.active {
     background: rgba(76, 175, 80, 0.2);
     border-color: #4CAF50;
     font-weight: 600;
   }
  
     .layer-icon {
     font-size: 18px;
   }
   
   .layer-name {
     font-size: 13px;
     font-weight: 500;
   }
  
     .layer-description {
     font-size: 10px;
     opacity: 0.6;
     color: #cccccc;
     margin-top: 1px;
   }
  
     .category-search {
     display: flex;
     align-items: center;
     gap: 8px;
     margin-bottom: 8px;
   }
   
   .search-input {
     flex: 1;
     padding: 6px 10px;
     border-radius: 4px;
     border: 1px solid rgba(255, 255, 255, 0.2);
     background: rgba(255, 255, 255, 0.1);
     color: white;
     font-size: 11px;
     transition: all 0.2s ease;
   }
   
   .search-input:focus {
     outline: none;
     border-color: #4CAF50;
     background: rgba(255, 255, 255, 0.15);
   }
   
   .search-input::placeholder {
     color: rgba(255, 255, 255, 0.6);
   }
  
     .category-actions {
     display: flex;
     align-items: center;
     gap: 6px;
     margin-bottom: 8px;
   }
   
   .action-btn {
     flex: 1;
     padding: 5px 8px;
     border-radius: 4px;
     border: 1px solid rgba(255, 255, 255, 0.2);
     background: rgba(255, 255, 255, 0.1);
     cursor: pointer;
     transition: all 0.2s ease;
     color: white;
     font-size: 10px;
     font-weight: 500;
   }
   
   .action-btn:hover {
     background: rgba(255, 255, 255, 0.2);
     border-color: rgba(255, 255, 255, 0.3);
   }
  
     .category-list {
     display: flex;
     flex-direction: column;
     gap: 3px;
     max-height: 180px;
     overflow-y: auto;
     padding-right: 4px;
   }
   
   .category-list::-webkit-scrollbar {
     width: 4px;
   }
   
   .category-list::-webkit-scrollbar-track {
     background: rgba(255, 255, 255, 0.1);
     border-radius: 2px;
   }
   
   .category-list::-webkit-scrollbar-thumb {
     background: rgba(255, 255, 255, 0.3);
     border-radius: 2px;
   }
   
   .category-list::-webkit-scrollbar-thumb:hover {
     background: rgba(255, 255, 255, 0.5);
   }
  
     .category-button {
     display: flex;
     align-items: center;
     gap: 6px;
     padding: 6px 8px;
     border-radius: 4px;
     cursor: pointer;
     transition: all 0.2s ease;
     color: white;
     font-size: 11px;
     border: 1px solid transparent;
   }
   
   .category-button:hover {
     background: rgba(255, 255, 255, 0.1);
     border-color: rgba(255, 255, 255, 0.2);
   }
   
   .category-button.active {
     background: rgba(76, 175, 80, 0.2);
     border-color: #4CAF50;
     font-weight: 600;
   }
  
     .category-icon {
     font-size: 14px;
   }
   
   .category-name {
     flex: 1;
     font-weight: 500;
   }
   
   .category-count {
     font-size: 10px;
     opacity: 0.8;
     color: #4CAF50;
     font-weight: 600;
   }
  
     .coords-display {
     display: flex;
     gap: 12px;
     margin-bottom: 6px;
   }
   
   .coord-item {
     display: flex;
     align-items: center;
     gap: 4px;
   }
   
   .coord-label {
     font-weight: 600;
     font-size: 11px;
     color: #cccccc;
   }
   
   .coord-value {
     color: #4CAF50;
     font-weight: 600;
     font-size: 11px;
   }
   
   .current-layer-info {
     display: flex;
     align-items: center;
     gap: 6px;
     font-size: 11px;
   }
   
   .layer-label {
     font-weight: 600;
     color: #cccccc;
   }
   
     .layer-value {
    color: #4CAF50;
    font-weight: 600;
  }
  
  .sun-control {
    display: flex;
    flex-direction: column;
    gap: 4px;
    margin-bottom: 8px;
  }
  
  .control-label {
    font-size: 11px;
    font-weight: 600;
    color: #cccccc;
  }
  
  .slider-input {
    width: 100%;
    height: 4px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 2px;
    outline: none;
    -webkit-appearance: none;
    appearance: none;
  }
  
  .slider-input::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 12px;
    height: 12px;
    background: #4CAF50;
    border-radius: 50%;
    cursor: pointer;
  }
  
  .slider-input::-moz-range-thumb {
    width: 12px;
    height: 12px;
    background: #4CAF50;
    border-radius: 50%;
    cursor: pointer;
    border: none;
  }
  
  .value-display {
    font-size: 10px;
    color: #4CAF50;
    font-weight: 600;
    text-align: right;
  }
  
  .color-input {
    width: 100%;
    height: 24px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 4px;
    background: transparent;
    cursor: pointer;
  }
  
  .color-input::-webkit-color-swatch-wrapper {
    padding: 0;
  }
  
  .color-input::-webkit-color-swatch {
    border: none;
    border-radius: 4px;
  }
  
  .sun-presets {
    margin-top: 8px;
  }
  
  .preset-buttons {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 4px;
    margin-top: 4px;
  }
  
  .preset-btn {
    padding: 4px 6px;
    border-radius: 4px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    background: rgba(255, 255, 255, 0.1);
    cursor: pointer;
    transition: all 0.2s ease;
    color: white;
    font-size: 10px;
    font-weight: 500;
  }
  
  .preset-btn:hover {
    background: rgba(255, 255, 255, 0.2);
    border-color: rgba(255, 255, 255, 0.3);
  }
  
  .loading-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.8);
    display: flex; 
    flex-direction: column;
    align-items: center;
    justify-content: center;
    z-index: 2000;
  }
  
  .loading-spinner {
    width: 40px;
    height: 40px;
    border: 4px solid rgba(255, 255, 255, 0.3);
    border-top: 4px solid #4CAF50;
    border-radius: 50%;
    animation: spin 1s linear infinite;
    margin-bottom: 16px;
  }
  
  .loading-text {
    color: white;
    font-size: 16px;
    font-weight: bold;
  }
  
  .mesh-loading-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 2000;
  }
  
  .mesh-loading-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 20px;
    background: rgba(0, 0, 0, 0.9);
    padding: 30px;
    border-radius: 12px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(12px);
    min-width: 300px;
  }
  
  .mesh-loading-spinner {
    width: 40px;
    height: 40px;
    border: 4px solid rgba(255, 255, 255, 0.3);
    border-top: 4px solid #4CAF50;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }
  
  .mesh-loading-text {
    color: white;
    font-size: 18px;
    font-weight: bold;
  }
  
  .mesh-loading-progress {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }
  
  .progress-bar {
    width: 100%;
    height: 8px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 4px;
    overflow: hidden;
  }
  
  .progress-fill {
    height: 100%;
    background: linear-gradient(90deg, #4CAF50, #45a049);
    border-radius: 4px;
    transition: width 0.3s ease;
  }
  
  .progress-text {
    color: white;
    font-size: 14px;
    text-align: center;
    font-weight: 500;
  }
  
  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  .tooltip {
    position: fixed;
    background: rgba(0, 0, 0, 0.9);
    color: white;
    padding: 8px 12px;
    border-radius: 4px;
    font-size: 14px;
    font-weight: bold;
    pointer-events: none;
    z-index: 10000;
    white-space: nowrap;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  }
  

  </style>