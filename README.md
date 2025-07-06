# 3D Map Viewer

A standalone 3D map viewer built with Vue 3, TypeScript, and Babylon.js. This project was extracted from a larger application to provide an isolated 3D map experience.

## Features

- **3D Terrain Rendering**: Interactive 3D terrain with heightmap-based displacement
- **Layer System**: Switch between Overworld, Underworld, and Sky layers
- **Marker Categories**: Toggle visibility of different marker types (Locations, Trees, NPCs, etc.)
- **Interactive Markers**: Click on markers to fly to their location
- **Mesh Loading**: Dynamic loading of 3D meshes with IndexedDB caching
- **Material System**: Advanced material handling with MTL file support
- **Responsive UI**: Modern overlay interface with layer and category controls

## Technology Stack

- **Vue 3** with Composition API
- **TypeScript** for type safety
- **Babylon.js** for 3D rendering
- **Vite** for fast development and building
- **IndexedDB** for mesh caching

## Getting Started

### Prerequisites

- Node.js (v16 or higher)
- Yarn package manager

### Installation

1. Clone or download this project
2. Install dependencies:
   ```bash
   yarn install
   ```

### Development

Start the development server:
```bash
yarn dev
```

The application will be available at `http://localhost:5173`

### Building for Production

Build the project for production:
```bash
yarn build
```

## Project Structure

```
src/
├── components/
│   └── BabylonMap.vue          # Main 3D map component
├── views/
│   └── MapView.vue             # Map view wrapper
├── assets/
│   ├── map/                    # Map data files
│   │   ├── Locations.json      # Location markers
│   │   ├── worldEntities.json  # World entity data
│   │   ├── NPCs.json          # NPC data
│   │   ├── NPCDefs.json       # NPC definitions
│   │   ├── heightmaps.json    # Terrain height data
│   │   └── meshes.json        # 3D mesh data
│   └── map.css                 # Map UI styles
└── main.ts                     # Application entry point
```

## Map Data

The map uses several JSON data files:

- **Locations.json**: Points of interest and landmarks
- **worldEntities.json**: Interactive objects and structures
- **NPCs.json**: Non-player character data
- **NPCDefs.json**: NPC type definitions
- **heightmaps.json**: Terrain elevation data
- **meshes.json**: 3D model data (OBJ/MTL files encoded as base64)

## Controls

- **Mouse**: Rotate camera around the map
- **Scroll**: Zoom in/out
- **Layer Buttons**: Switch between map layers
- **Category Toggles**: Show/hide different marker types
- **Marker Clicks**: Fly to marker location

## Customization

### Adding New Layers

Edit the `layers` array in `BabylonMap.vue` to add new map layers.

### Adding New Marker Categories

Add new categories to the `markerCategories` array in `BabylonMap.vue`.

### Modifying Map Data

Update the JSON data files in `src/assets/map/` to modify map content.

## Performance

- Mesh data is cached in IndexedDB for faster loading
- LOD (Level of Detail) system for performance optimization
- Efficient material sharing and instancing

## Browser Support

Requires a modern browser with WebGL support and IndexedDB capabilities.

## License

This project is part of a larger application. Please ensure you have the appropriate permissions to use this code.
