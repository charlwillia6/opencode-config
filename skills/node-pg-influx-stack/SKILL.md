---
name: node-pg-influx-stack
description: Node.js backend with PostgreSQL (relational) and InfluxDB (time-series) patterns for high-performance full-stack applications.
---

# Node.js Stack: PostgreSQL + InfluxDB

## Architecture

```
src/
├── app.ts                  # Express/Fastify app entry
├── features/              # Feature modules
│   ├── analytics/
│   └── devices/
├── infrastructure/
│   ├── db/
│   │   ├── pg/           # PostgreSQL pool & queries
│   │   └── influx/       # InfluxDB client
│   └── config/
├── shared/
│   ├── types/
│   └── middleware/
└── utils/
```

## Technology Stack (2026)

| Layer | Technology | Version |
|-------|-----------|---------|
| **Framework** | Express/Fastify | Express 5.x / Fastify 5.x |
| **Database** | PostgreSQL | 16.x |
| **Time-Series** | InfluxDB | 3.x |
| **ORM** | Prisma | 6.x |
| **Validation** | Zod | 3.24.x |
| **Testing** | Vitest | 2.x |

## PostgreSQL Configuration

### Database Pool Setup
```typescript
// infrastructure/db/pg/pool.ts
import { Pool } from 'pg';
import dotenv from 'dotenv';

dotenv.config();

const pool = new Pool({
  host: process.env.PG_HOST || 'localhost',
  port: parseInt(process.env.PG_PORT || '5432'),
  database: process.env.PG_DATABASE || 'app',
  user: process.env.PG_USER || 'postgres',
  password: process.env.PG_PASSWORD,
  max: parseInt(process.env.PG_MAX_CONNECTIONS || '20'),
  connectionTimeoutMillis: 5000,
  idleTimeoutMillis: 30000,
});

export async function query<T>(text: string, params?: any[]): Promise<T[]> {
  const start = Date.now();
  try {
    const res = await pool.query(text, params);
    const duration = Date.now() - start;
    console.log('Executed query', { text, duration, rows: res.rowCount });
    return res.rows as T[];
  } catch (error) {
    console.error('Query error', { text, error });
    throw error;
  }
}

export functionClient() {
  const client = await pool.connect();
  try {
    yield client;
  } finally {
    client.release();
  }
}
```

### Prisma Schema
```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  role      UserRole @default(USER)
  devices   Device[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("users")
}

model Device {
  id        String   @id @default(cuid())
  name      String
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  measurements Measurement[]
  createdAt DateTime @default(now()

  @@map("devices")
}

model Measurement {
  id        String   @id @default(cuid())
  deviceId  String
  device    Device   @relation(fields: [deviceId], references: [id])
  value     Float
  timestamp DateTime @index
  metadata  Json?

  @@map("measurements")
  @@index([deviceId, timestamp])
}

enum UserRole {
  USER
  ADMIN
  SUPERADMIN
}
```

## InfluxDB Configuration

### Database Client
```typescript
// infrastructure/db/influx/client.ts
import { InfluxDB } from '@influxdata/influxdb-client';
import { Bucket } from '@influxdata/influxdb-client-api';

const influxURL = process.env.INFLUXDB_URL || 'http://localhost:8086';
const influxToken = process.env.INFLUXDB_TOKEN;
const influxOrg = process.env.INFLUXDB_ORG || 'my-org';
const influxBucket = process.env.INFLUXDB_BUCKET || 'measurements';

const influxDB = new InfluxDB({ url: influxURL, token: influxToken });

export async function writePoint(bucket: string, measurement: string, fields: Record<string, any>, tags: Record<string, string> = {}) {
  const writeApi = influxDB.getWriteApi(influxOrg, bucket);
  const point = Point(measurement)
    .tag(tags)
    .fields(fields)
    .timestamp(new Date());
  
  writeApi.writePoint(point);
  await writeApi.flush();
  writeApi.close();
}

export async function queryRecentMeasurements(deviceId: string, minutes: number = 60) {
  const queryApi = influxDB.getQueryApi(influxOrg);
  const query = `
    from(bucket: "${influxBucket}")
      |> range(start: -${minutes}m)
      |> filter(fn: (r) => r._measurement == "measurements" and r.device_id == "${deviceId}")
      |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
      |> sort(columns: ["_time"], desc: true)
      |> limit(limit: 1000)
  `;
  
  const results = await queryApi.queryRows(query);
  return results.map(r => ({
    timestamp: r._time,
    value: r._value,
    ...Object.fromEntries(
      Object.entries(r).filter(([k]) => !k.startsWith('_'))
    ),
  }));
}
```

### Prisma Integration with InfluxDB
```typescript
// features/devices/api/deviceMeasurements.ts
import { query, Client } from '@/infrastructure/db/pg/pool';
import { writePoint, queryRecentMeasurements } from '@/infrastructure/db/influx/client';
import { z } from 'zod';

const MeasurementInput = z.object({
  deviceId: z.string(),
  value: z.number(),
  timestamp: z.date().optional(),
  metadata: z.record(z.unknown()).optional(),
});

export async function recordMeasurement(input: z.infer<typeof MeasurementInput>) {
  const validated = MeasurementInput.parse(input);
  
  // Write to PostgreSQL
  await query(
    'INSERT INTO measurements (device_id, value, timestamp, metadata) VALUES ($1, $2, $3, $4)',
    [validated.deviceId, validated.value, validated.timestamp || new Date(), validated.metadata || null]
  );
  
  // Write to InfluxDB for time-series queries
  writePoint(
    'measurements',
    'device_readings',
    { value: validated.value },
    { device_id: validated.deviceId }
  );
}

export async function getDeviceAnalytics(deviceId: string) {
  // Get time-series data from InfluxDB
  const recentMeasurements = await queryRecentMeasurements(deviceId, 60);
  
  // Get device info from PostgreSQL
  const [device] = await query<{ id: string; name: string }>(
    'SELECT id, name FROM devices WHERE id = $1',
    [deviceId]
  );
  
  // Compute statistics
  const values = recentMeasurements.map(m => m.value);
  const avg = values.reduce((a, b) => a + b, 0) / values.length;
  const min = Math.min(...values);
  const max = Math.max(...values);
  
  return {
    device,
    statistics: { avg, min, max, count: values.length },
    recent: recentMeasurements,
  };
}
```

## API Layer

### Express Router Pattern
```typescript
// features/devices/devicesRouter.ts
import { Router } from 'express';
import { getDeviceAnalytics, recordMeasurement } from './api/deviceMeasurements';

const router = Router();

router.get('/analytics/:deviceId', async (req, res) => {
  try {
    const analytics = await getDeviceAnalytics(req.params.deviceId);
    res.json(analytics);
  } catch (error) {
    res.status(500).json({ error: 'Failed to get analytics' });
  }
});

router.post('/measurements', async (req, res) => {
  try {
    await recordMeasurement(req.body);
    res.status(201).json({ success: true });
  } catch (error) {
    res.status(400).json({ error: 'Invalid measurement data' });
  }
});

export default router;
```

## Performance Optimization

### Query Optimization
```typescript
// Use indexes for time-series queries
CREATE INDEX idx_measurements_device_time 
ON measurements (device_id, timestamp DESC);

// Use prepared statements for parameterized queries
await query(
  'SELECT * FROM measurements WHERE device_id = $1 AND timestamp > $2',
  [deviceId, cutoffDate]
);
```

### Connection Pooling
```typescript
// environment variables
PG_MAX_CONNECTIONS=20
INFLUXDB_BATCH_SIZE=100  // Batch writes for efficiency
```

### Caching Strategy
```typescript
// Cache expensive analytics queries
import NodeCache from 'node-cache';

const analyticsCache = new NodeCache({ stdTTL: 300 }); // 5 minutes

export async function getCachedAnalytics(deviceId: string) {
  const cacheKey = `analytics:${deviceId}`;
  const cached = analyticsCache.get(cacheKey);
  
  if (cached) return cached;
  
  const analytics = await getDeviceAnalytics(deviceId);
  analyticsCache.set(cacheKey, analytics);
  
  return analytics;
}
```

## Testing Strategy

### Unit Tests
```typescript
// features/devices/__tests__/deviceMeasurements.test.ts
import { recordMeasurement, getDeviceAnalytics } from '../api/deviceMeasurements';
import * as pgPool from '@/infrastructure/db/pg/pool';
import * as influxClient from '@/infrastructure/db/influx/client';

jest.mock('@/infrastructure/db/pg/pool');
jest.mock('@/infrastructure/db/influx/client');

describe('recordMeasurement', () => {
  it('writes to both PostgreSQL and InfluxDB', async () => {
    const mockQuery = jest.spyOn(pgPool, 'query').mockResolvedValue([]);
    jest.spyOn(influxClient, 'writePoint').mockResolvedValue(undefined);
    
    await recordMeasurement({
      deviceId: 'device-123',
      value: 42.5,
      timestamp: new Date(),
    });
    
    expect(mockQuery).toHaveBeenCalled();
    expect(influxClient.writePoint).toHaveBeenCalled();
  });
});
```

### Integration Tests
```typescript
// features/devices/__tests__/devicesRouter.test.ts
import request from 'supertest';
import app from '@/app';

describe('GET /analytics/:deviceId', () => {
  it('returns device analytics', async () => {
    const response = await request(app).get('/analytics/device-123');
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('statistics');
    expect(response.body).toHaveProperty('recent');
  });
});
```

## Common Patterns & Anti-Patterns

### Do
✅ Use Prisma for complex relational queries  
✅ Use InfluxDB for time-series data  
✅ Implement connection pooling  
✅ Cache expensive analytics queries  
✅ Batch InfluxDB writes  

### Don't
❌ Mix time-series and relational in same table  
❌ Use synchronous database operations  
❌ Forget to set timeout on database connections  
❌ Forget to close InfluxDB write APIs  
❌ Store large binary data in PostgreSQL  

## References

* [PostgreSQL Documentation](https://www.postgresql.org/docs)
* [InfluxDB 3.x Official Docs](https://docs.influxdata.com/influxdb/v3)
* [Prisma Documentation](https://www.prisma.io/docs)
* [Express.js Guide](https://expressjs.com)
* [Fastify Documentation](https://www.fastify.io)

## 2026 Best Practices

* **Prisma 6**: Better performance with Rust query engine
* **InfluxDB 3**: Native time-series indexing
* **Vitest**: Fast testing with native ESM
* **ESM+TypeScript**: Use `"type": "module"` in package.json
* **Connection Pooling**: Always use connection pooling in production
