---
name: gis-development-core
description: Core GIS development patterns with PostGIS, Leaflet/OpenLayers, coordinate systems, and geospatial processing. Includes 2026 best practices.
---

# GIS Development Core (2026)

## Geospatial Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Database** | PostgreSQL + PostGIS | Geospatial database |
| **GIS Library** | Leaflet / OpenLayers | Web mapping |
| **GeoJSON** | Standard | Data interchange |
| **GDAL/OGR** | Toolbelt | Geospatial processing |
| **Proj** | Coordinate system engine | CRS transformations |

## Project Structure

```
src/
├── backend/
│   ├── sql/
│   │   ├── schema.sql
│   │   └── migrations/
│   └── services/
│       └── geospatialService.ts
├── frontend/
│   ├── components/
│   │   └── Map/
│   │       ├── Map.tsx
│   │       ├── MapControls.tsx
│   │       └── layers/
│   ├── utils/
│   │   └── geoUtils.ts
│   └── types/
│       └── geojson.ts
└── shared/
    └── geoTypes.ts
```

## PostGIS Setup

### Database Schema
```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Enable topology for advanced features
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- Enable fuzzy string matching
CREATE EXTENSION IF NOT EXISTS postgis_tiger_geocoder;

-- Create table with geometry column
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address TEXT,
    location GEOMETRY(POINT, 4326),  -- WGS84 coordinate system
    created_at TIMESTAMP DEFAULT NOW()
);

-- Add spatial index for fast queries
CREATE INDEX idx_locations_location ON locations USING GIST (location);

-- Create table with polygon geometry
CREATE TABLE districts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    boundary GEOMETRY(POLYGON, 4326),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_districts_boundary ON districts USING GIST (boundary);

-- Create table with linestring geometry
CREATE TABLE routes (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    path GEOMETRY(LINESTRING, 4326),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Geospatial Queries
```typescript
// backend/services/geospatialService.ts
import { Pool } from 'pg';

export class GeospatialService {
  private pool: Pool;

  constructor(pool: Pool) {
    this.pool = pool;
  }

  // Create point from coordinates
  async createLocation(name: string, longitude: number, latitude: number): Promise<void> {
    await this.pool.query(
      `INSERT INTO locations (name, location) VALUES ($1, ST_SetSRID(ST_MakePoint($2, $3), 4326))`,
      [name, longitude, latitude]
    );
  }

  // Find locations within radius
  async findNearby(longitude: number, latitude: number, radiusMeters: number): Promise<any[]> {
    const result = await this.pool.query(
      `
      SELECT id, name, ST_AsGeoJSON(location) as location
      FROM locations
      WHERE ST_DWithin(
        location,
        ST_SetSRID(ST_MakePoint($1, $2), 4326),
        $3
      )
      ORDER BY ST_Distance(location, ST_SetSRID(ST_MakePoint($1, $2), 4326))
      `,
      [longitude, latitude, radiusMeters]
    );
    return result.rows;
  }

  // Find locations within polygon
  async findWithinBoundary(polygonCoords: [number, number][]): Promise<any[]> {
    const polygonWkt = `POLYGON((${polygonCoords.map(([lng, lat]) => `${lng} ${lat}`).join(', ')}))`;
    
    const result = await this.pool.query(
      `
      SELECT id, name, ST_AsGeoJSON(location) as location
      FROM locations
      WHERE ST_Within(
        location,
        ST_GeomFromText($1, 4326)
      )
      `,
      [polygonWkt]
    );
    return result.rows;
  }

  // Calculate distance between two points
  async calculateDistance(
    lng1: number, lat1: number, lng2: number, lat2: number
  ): Promise<number> {
    const result = await this.pool.query(
      `
      SELECT ST_Distance(
        ST_SetSRID(ST_MakePoint($1, $2), 4326),
        ST_SetSRID(ST_MakePoint($3, $4), 4326)
      ) as distance
      `,
      [lng1, lat1, lng2, lat2]
    );
    
    // Distance is in degrees, convert to meters
    return parseFloat(result.rows[0].distance) * 111319.9; // Approximate meters per degree
  }

  // Get bounding box of all locations
  async getBoundingBox(): Promise<{ minLng: number; minLat: number; maxLng: number; maxLat: number } | null> {
    const result = await this.pool.query(
      `
      SELECT 
        ST_XMin(ST_Envelope(ST_Collect(location))) as min_lng,
        ST_YMin(ST_Envelope(ST_Collect(location))) as min_lat,
        ST_XMax(ST_Envelope(ST_Collect(location))) as max_lng,
        ST_YMax(ST_Envelope(ST_Collect(location))) as max_lat
      FROM locations
      `
    );
    
    if (!result.rows[0].min_lng) return null;
    
    return {
      minLng: parseFloat(result.rows[0].min_lng),
      minLat: parseFloat(result.rows[0].min_lat),
      maxLng: parseFloat(result.rows[0].max_lng),
      maxLat: parseFloat(result.rows[0].max_lat),
    };
  }

  // Find intersections with polygon
  async findIntersectingRoutes(polygonCoords: [number, number][]): Promise<any[]> {
    const polygonWkt = `POLYGON((${polygonCoords.map(([lng, lat]) => `${lng} ${lat}`).join(', ')}))`;
    
    const result = await this.pool.query(
      `
      SELECT id, name, ST_AsGeoJSON(path) as path
      FROM routes
      WHERE ST_Intersects(
        path,
        ST_GeomFromText($1, 4326)
      )
      `,
      [polygonWkt]
    );
    return result.rows;
  }

  // Get intersection points
  async getIntersectionPoints(lng1: number, lat1: number, lng2: number, lat2: number): Promise<any[]> {
    const result = await this.pool.query(
      `
      WITH line AS (
        SELECT ST_MakeLine(
          ST_SetSRID(ST_MakePoint($1, $2), 4326),
          ST_SetSRID(ST_MakePoint($3, $4), 4326)
        ) as geom
      )
      SELECT 
        ST_AsGeoJSON(ST_Intersection(l.geom, d.boundary)) as intersection
      FROM line l, districts d
      WHERE ST_Intersects(l.geom, d.boundary)
      `,
      [lng1, lat1, lng2, lat2]
    );
    return result.rows;
  }
}
```

## Web Mapping (Leaflet 2026)

### Map Component
```typescript
// frontend/components/Map/Map.tsx
import { useState, useEffect, useRef } from 'react';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

interface MapProps {
  center?: [number, number];
  zoom?: number;
  initialLocations?: any[];
  onLocationClick?: (location: any) => void;
}

export function Map({ center = [51.505, -0.09], zoom = 13, initialLocations = [], onLocationClick }: MapProps) {
  const mapRef = useRef<L.Map | null>(null);
  const markersRef = useRef<L.Marker[]>([]);

  useEffect(() => {
    // Initialize map
    mapRef.current = L.map('map-container').setView(center, zoom);

    // Add tile layer
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; OpenStreetMap contributors',
      maxZoom: 19,
    }).addTo(mapRef.current);

    return () => {
      mapRef.current?.remove();
      mapRef.current = null;
    };
  }, [center, zoom]);

  useEffect(() => {
    if (!mapRef.current) return;

    // Clear existing markers
    markersRef.current.forEach(marker => mapRef.current!.removeLayer(marker));
    markersRef.current = [];

    // Add new markers
    initialLocations.forEach(location => {
      if (location.location && location.location.coordinates) {
        const [lng, lat] = location.location.coordinates;
        const marker = L.marker([lat, lng]).addTo(mapRef.current!);
        
        marker.bindPopup(`
          <b>${location.name}</b><br>
          ${location.address || 'No address'}
        `);
        
        marker.on('click', () => {
          if (onLocationClick) onLocationClick(location);
        });

        markersRef.current.push(marker);
      }
    });
  }, [initialLocations, onLocationClick]);

  // Add map controls
  useEffect(() => {
    if (!mapRef.current) return;

    L.control.scale({ imperial: false, metric: true }).addTo(mapRef.current);
    L.control.zoom({ position: 'topright' }).addTo(mapRef.current);
  }, []);

  return <div id="map-container" style={{ height: '500px', width: '100%' }} />;
}
```

### Map Controls
```typescript
// frontend/components/Map/MapControls.tsx
import { useState } from 'react';

interface MapControlsProps {
  onZoomToLocations: () => void;
  onDrawPolygon: (callback: (coords: [number, number][]) => void) => void;
  onClearMap: () => void;
}

export function MapControls({ onZoomToLocations, onDrawPolygon, onClearMap }: MapControlsProps) {
  const [isDrawing, setIsDrawing] = useState(false);

  const handleDrawPolygon = () => {
    setIsDrawing(true);
    
    // Trigger drawing mode
    onDrawPolygon((coords) => {
      console.log('Polygon coordinates:', coords);
      setIsDrawing(false);
    });
  };

  return (
    <div className="map-controls">
      <button onClick={onZoomToLocations} title="Zoom to locations">
        <span role="img" aria-label="zoom">🔍</span>
      </button>
      
      <button onClick={handleDrawPolygon} disabled={isDrawing} title="Draw polygon">
        <span role="img" aria-label="polygon">🔺</span>
      </button>
      
      <button onClick={onClearMap} title="Clear map">
        <span role="img" aria-label="clear">🗑️</span>
      </button>
    </div>
  );
}
```

### Coordinate Conversion Utilities
```typescript
// frontend/utils/geoUtils.ts
/**
 * Convert degrees to radians
 */
export function degreesToRadians(degrees: number): number {
  return degrees * (Math.PI / 180);
}

/**
 * Convert radians to degrees
 */
export function radiansToDegrees(radians: number): number {
  return radians * (180 / Math.PI);
}

/**
 * Calculate distance between two points using Haversine formula
 * Returns distance in meters
 */
export function calculateDistance(lat1: number, lon1: number, lat2: number, lon2: number): number {
  const R = 6371e3; // Earth's radius in meters
  
  const φ1 = degreesToRadians(lat1);
  const φ2 = degreesToRadians(lat2);
  const Δφ = degreesToRadians(lat2 - lat1);
  const Δλ = degreesToRadians(lon2 - lon1);
  
  const a = Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
            Math.cos(φ1) * Math.cos(φ2) *
            Math.sin(Δλ / 2) * Math.sin(Δλ / 2);
  
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  
  return R * c;
}

/**
 * Calculate bearing between two points
 * Returns bearing in degrees (0-360)
 */
export function calculateBearing(lat1: number, lon1: number, lat2: number, lon2: number): number {
  const φ1 = degreesToRadians(lat1);
  const φ2 = degreesToRadians(lat2);
  const Δλ = degreesToRadians(lon2 - lon1);
  
  const y = Math.sin(Δλ) * Math.cos(φ2);
  const x = Math.cos(φ1) * Math.sin(φ2) -
            Math.sin(φ1) * Math.cos(φ2) * Math.cos(Δλ);
  
  const θ = Math.atan2(y, x);
  
  return (radiansToDegrees(θ) + 360) % 360;
}

/**
 * Get destination point given origin, bearing, and distance
 */
export function getDestinationPoint(lat: number, lng: number, bearing: number, distance: number): [number, number] {
  const R = 6371e3; // Earth's radius in meters
  const δ = distance / R; // Angular distance in radians
  const θ = degreesToRadians(bearing);
  const φ1 = degreesToRadians(lat);
  const λ1 = degreesToRadians(lng);
  
  const φ2 = Math.asin(Math.sin(φ1) * Math.cos(δ) +
                       Math.cos(φ1) * Math.sin(δ) * Math.cos(θ));
  
  const λ2 = λ1 + Math.atan2(Math.sin(θ) * Math.sin(δ) * Math.cos(φ1),
                             Math.cos(δ) - Math.sin(φ1) * Math.sin(φ2));
  
  return [
    radiansToDegrees(φ2),
    radiansToDegrees(λ2)
  ];
}

/**
 * Convert WGS84 (EPSG:4326) to Web Mercator (EPSG:3857)
 */
export function wgs84ToMercator(lat: number, lng: number): [number, number] {
  const x = lng * 20037508.34 / 180;
  const y = Math.log(Math.tan((90 + lat) * Math.PI / 360)) / (Math.PI / 180);
  return [x, y];
}

/**
 * Convert Web Mercator (EPSG:3857) to WGS84 (EPSG:4326)
 */
export function mercatorToWgs84(x: number, y: number): [number, number] {
  const lng = (x / 20037508.34) * 180;
  const lat = (y / 20037508.34) * 180;
  return [
    180 / Math.PI * (2 * Math.atan(Math.exp(lat * Math.PI / 180)) - Math.PI / 2),
    lng
  ];
}

/**
 * Check if point is within polygon
 * Uses ray casting algorithm
 */
export function pointInPolygon(point: [number, number], polygon: [number, number][]): boolean {
  const [x, y] = point;
  let inside = false;
  
  for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
    const [xi, yi] = polygon[i];
    const [xj, yj] = polygon[j];
    
    const intersect = ((yi > y) !== (yj > y)) &&
                      (x < (xj - xi) * (y - yi) / (yj - yi) + xi);
    
    if (intersect) inside = !inside;
  }
  
  return inside;
}
```

## Common Patterns & Anti-Patterns

### Do
✅ Always specify SRID (4326 for WGS84)  
✅ Use spatial indexes (GIST) for geometry columns  
✅ Use `ST_DWithin` for radius queries instead of distance calculations  
✅ Use `ST_Intersects` for polygon relationships  
✅ Use Leaflet/OpenLayers for web mapping  
✅ Convert coordinates properly between CRS  

### Don't
❌ Store coordinates as separate lat/long columns  
❌ Forget to create spatial indexes  
❌ Use distance calculations in WHERE clauses without indexes  
❌ Mix different coordinate systems  
❌ Use Web Mercator for precise measurements  

## CRS (Coordinate Reference Systems)

### Common CRS
| EPSG | Name | Use Case |
|------|------|----------|
| 4326 | WGS84 | Geographic coordinates (lat/long) |
| 3857 | Web Mercator | Web mapping (Google Maps, OpenStreetMap) |
| 27700 | British National Grid | UK mapping |
| 32633 | WGS84 / UTM zone 33N | Europe |
| 32733 | WGS84 / UTM zone 33S | Southern hemisphere |

### Conversion Example
```typescript
// Use proj4 for complex conversions
import proj4 from 'proj4';

// Define CRS
proj4.defs('EPSG:27700', '+proj=tmerc +lat_0=49 +lon_0=-2 +k=0.999601 +x_0=400000 +y_0=-100000 +ellps=airy +towgs84=446.448,-125.157,542.060,0.1502,0.2470,0.8421,-20.4894 +units=m +no_defs');

// Convert between CRS
const wgs84 = [-2.3522, 51.4638]; // London
const bng = proj4('EPSG:4326', 'EPSG:27700', wgs84);
console.log(`British National Grid: ${bng}`); // [375000, 174000]
```

## References

* [PostGIS Documentation](https://postgis.net/documentation)
* [Leaflet Documentation](https://leafletjs.com)
* [OpenLayers Documentation](https://openlayers.org)
* [GDAL Documentation](https://gdal.org)
* [Proj Documentation](https://proj.org)

## 2026 Best Practices

* **PostGIS 3.4**: Use for advanced geospatial features
* **GeoJSON Standard**: Always use GeoJSON for web interchange
* **Spatial Indexes**: Always create GIST indexes on geometry columns
* **Leaflet 2.0**: Modern, lightweight mapping
* **GDAL 3.8**: Best for geospatial processing
