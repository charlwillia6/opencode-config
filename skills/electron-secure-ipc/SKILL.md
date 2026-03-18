---
name: electron-secure-ipc
description: Secure Electron application development with IPC (Inter-Process Communication) patterns, context isolation, and security best practices for 2026.
---

# Electron Secure IPC Development (2026)

## Architecture Overview

```
src/
├── main/
│   ├── index.ts              # Main process entry
│   ├── ipc/
│   │   ├── handler.ts        # IPC message handlers
│   │   └── channel.ts        # IPC channel definitions
│   └── services/
│       └── databaseService.ts
├── renderer/
│   ├── index.tsx             # Renderer process entry
│   ├── components/
│   ├── hooks/
│   │   └── useIpc.ts         # Custom IPC hooks
│   └── utils/
│       └── ipc.ts            # IPC utility functions
└── shared/
    └── types.ts              # Shared TypeScript types
```

## Security Principles

### Context Isolation
Always use context isolation in Electron 2026:
```typescript
// main/index.ts
const createWindow = () => {
  const window = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      enableRemoteModule: false,
      webSecurity: true,
      sandbox: true
    }
  });
};
```

### Safe IPC Patterns
```typescript
// renderer/hooks/useIpc.ts
import { ipcRenderer } from 'electron';

// Define types in shared
export interface DeviceMessage {
  type: 'device-status' | 'device-data';
  payload: any;
}

// Use typesafe wrappers
export function useDeviceStatus(callback: (data: any) => void) {
  useEffect(() => {
    const handler = (_: any, data: any) => callback(data);
    
    ipcRenderer.on('device-status', handler);
    return () => {
      ipcRenderer.off('device-status', handler);
    };
  }, [callback]);
}
```

## IPC Channel Registry

### Channel Definitions
```typescript
// shared/ipcChannels.ts
export const IPC_CHANNELS = {
  // Device management
  DEVICE_LIST: 'device:list',
  DEVICE_GET: 'device:get',
  DEVICE_CREATE: 'device:create',
  DEVICE_UPDATE: 'device:update',
  DEVICE_DELETE: 'device:delete',
  
  // Settings
  SETTINGS_GET: 'settings:get',
  SETTINGS_SAVE: 'settings:save',
  
  // Database
  DB_QUERY: 'db:query',
  DB_INSERT: 'db:insert',
  
  // Updater
  UPDATE_CHECK: 'update:check',
  UPDATE_DOWNLOAD: 'update:download',
  
  // Notification
  NOTIFICATION_SHOW: 'notification:show',
  
  // File operations
  FILE_OPEN: 'file:open',
  FILE_SAVE: 'file:save',
  FILE_EXPORT: 'file:export'
} as const;

export type IpcChannel = typeof IPC_CHANNELS[keyof typeof IPC_CHANNELS];
```

### Main Process Handler
```typescript
// main/ipc/handler.ts
import { ipcMain, shell } from 'electron';
import { IPC_CHANNELS } from '../shared/ipcChannels';
import { DatabaseService } from '../services/databaseService';
import { SettingsService } from '../services/settingsService';

export class IpcHandler {
  private databaseService: DatabaseService;
  private settingsService: SettingsService;

  constructor() {
    this.databaseService = new DatabaseService();
    this.settingsService = new SettingsService();
  }

  registerAll() {
    // Device management
    this.registerHandler(IPC_CHANNELS.DEVICE_LIST, this.listDevices.bind(this));
    this.registerHandler(IPC_CHANNELS.DEVICE_GET, this.getDevice.bind(this));
    this.registerHandler(IPC_CHANNELS.DEVICE_CREATE, this.createDevice.bind(this));
    this.registerHandler(IPC_CHANNELS.DEVICE_UPDATE, this.updateDevice.bind(this));
    this.registerHandler(IPC_CHANNELS.DEVICE_DELETE, this.deleteDevice.bind(this));
    
    // Settings
    this.registerHandler(IPC_CHANNELS.SETTINGS_GET, this.getSettings.bind(this));
    this.registerHandler(IPC_CHANNELS.SETTINGS_SAVE, this.saveSettings.bind(this));
    
    // File operations
    this.registerHandler(IPC_CHANNELS.FILE_OPEN, this.openFile.bind(this));
    this.registerHandler(IPC_CHANNELS.FILE_EXPORT, this.exportFile.bind(this));
    
    // Updater
    this.registerHandler(IPC_CHANNELS.UPDATE_CHECK, this.checkForUpdates.bind(this));
  }

  registerHandler<T = any, R = any>(channel: string, handler: (event: Electron.IpcMainEvent, args: T) => R | Promise<R>) {
    ipcMain.handle(channel, async (event, args) => {
      try {
        const result = await handler(event, args);
        return result;
      } catch (error) {
        console.error(`IPC handler error for ${channel}:`, error);
        throw error;
      }
    });
  }

  // Device handlers
  async listDevices() {
    return await this.databaseService.getAllDevices();
  }

  async getDevice(id: string) {
    return await this.databaseService.getDevice(id);
  }

  async createDevice(data: any) {
    return await this.databaseService.createDevice(data);
  }

  async updateDevice(id: string, data: any) {
    return await this.databaseService.updateDevice(id, data);
  }

  async deleteDevice(id: string) {
    return await this.databaseService.deleteDevice(id);
  }

  // Settings handlers
  async getSettings() {
    return await this.settingsService.getAllSettings();
  }

  async saveSettings(settings: any) {
    return await this.settingsService.saveSettings(settings);
  }

  // File handlers
  async openFile(filters: any) {
    const result = await shell.openExternal('file://');
    return result;
  }

  async exportFile(data: any, filename: string) {
    // Implement file export logic
    return { success: true };
  }

  // Updater handlers
  async checkForUpdates() {
    // Implement update checking logic
    return { updateAvailable: false };
  }
}
```

### Preload Script
```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';
import { IPC_CHANNELS } from './shared/ipcChannels';

contextBridge.exposeInMainWorld('electron', {
  // Device API
  devices: {
    list: () => ipcRenderer.invoke(IPC_CHANNELS.DEVICE_LIST),
    get: (id: string) => ipcRenderer.invoke(IPC_CHANNELS.DEVICE_GET, id),
    create: (data: any) => ipcRenderer.invoke(IPC_CHANNELS.DEVICE_CREATE, data),
    update: (id: string, data: any) => ipcRenderer.invoke(IPC_CHANNELS.DEVICE_UPDATE, id, data),
    delete: (id: string) => ipcRenderer.invoke(IPC_CHANNELS.DEVICE_DELETE, id),
  },
  
  // Settings API
  settings: {
    get: () => ipcRenderer.invoke(IPC_CHANNELS.SETTINGS_GET),
    save: (settings: any) => ipcRenderer.invoke(IPC_CHANNELS.SETTINGS_SAVE, settings),
  },
  
  // File API
  file: {
    export: (data: any, filename: string) => ipcRenderer.invoke(IPC_CHANNELS.FILE_EXPORT, data, filename),
  },
  
  // Notification API
  notification: {
    show: (title: string, body: string) => ipcRenderer.invoke(IPC_CHANNELS.NOTIFICATION_SHOW, { title, body }),
  },
  
  // Update API
  update: {
    check: () => ipcRenderer.invoke(IPC_CHANNELS.UPDATE_CHECK),
  },
});

// Handle events
ipcRenderer.on('device-updated', (_event, data) => {
  window.dispatchEvent(new CustomEvent('device-updated', { detail: data }));
});
```

## Renderer Process Usage

### React Component Integration
```typescript
// renderer/components/DeviceList.tsx
import { useEffect } from 'react';
import { useDevices } from '../hooks/useDevices';

export function DeviceList() {
  const { devices, loading, error } = useDevices();
  
  // Listen for device updates via custom events
  useEffect(() => {
    const handleDeviceUpdate = (event: CustomEvent) => {
      console.log('Device updated:', event.detail);
      // Update local state
    };
    
    window.addEventListener('device-updated', handleDeviceUpdate);
    return () => {
      window.removeEventListener('device-updated', handleDeviceUpdate);
    };
  }, []);
  
  if (loading) return <div>Loading devices...</div>;
  if (error) return <div>Error loading devices: {error.message}</div>;
  
  return (
    <div className="device-list">
      {devices.map(device => (
        <div key={device.id} className="device-item">
          <h3>{device.name}</h3>
          <p>{device.serialNumber}</p>
          <span className={`status ${device.status}`}>{device.status}</span>
        </div>
      ))}
    </div>
  );
}
```

### Custom Hook
```typescript
// renderer/hooks/useDevices.ts
import { useState, useEffect } from 'react';
import { Device } from '../types';

export function useDevices() {
  const [devices, setDevices] = useState<Device[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const loadDevices = async () => {
      try {
        const result = await (window as any).electron.devices.list();
        setDevices(result);
      } catch (err) {
        setError(err as Error);
      } finally {
        setLoading(false);
      }
    };

    loadDevices();
  }, []);

  return { devices, loading, error };
}
```

## Security Features

### Input Validation
```typescript
// main/ipc/validators.ts
import { z } from 'zod';

export const CreateDeviceSchema = z.object({
  name: z.string().min(1).max(100),
  serialNumber: z.string().min(5).max(50).regex(/^[A-Z0-9]+$/),
  status: z.enum(['online', 'offline', 'maintenance']).optional().default('offline'),
  metadata: z.record(z.unknown()).optional()
});

export type CreateDeviceInput = z.infer<typeof CreateDeviceSchema>;

// In handler
async createDevice(input: any) {
  const validated = CreateDeviceSchema.parse(input);
  // Process validated data...
}
```

### Request Origin Validation
```typescript
// main/ipc/security.ts
export function validateOrigin(event: Electron.IpcMainEvent): boolean {
  const sender = event.sender as Electron.WebContents;
  const url = sender.getURL();
  
  // Only allow specific origins
  const allowedOrigins = [
    'file://',
    'app://',
    'http://localhost:*',
  ];
  
  return allowedOrigins.some(origin => {
    const regex = new RegExp(origin.replace('*', '.*'));
    return regex.test(url);
  });
}

// Use in handlers
async listDevices(event: Electron.IpcMainEvent) {
  if (!validateOrigin(event)) {
    throw new Error('Forbidden: Invalid origin');
  }
  
  return await this.databaseService.getAllDevices();
}
```

## Error Handling

### Comprehensive Error Handling
```typescript
// main/ipc/errorHandler.ts
import { ipcMain } from 'electron';

export class IpcErrorHandler {
  static wrap<T = any, R = any>(
    channel: string,
    handler: (args: T) => R
  ) {
    return async (event: Electron.IpcMainInvocationEvent, args: T): Promise<R> => {
      try {
        return await handler(args);
      } catch (error) {
        console.error(`IPC ${channel} error:`, error);
        
        // Don't expose internal error details
        throw new Error('An unexpected error occurred');
      }
    };
  }
}

// Usage
ipcMain.handle(IPC_CHANNELS.DEVICE_CREATE, 
  IpcErrorHandler.wrap(CreateDeviceSchema.parse, async (data) => {
    return await db.createDevice(data);
  })
);
```

## Common Patterns & Anti-Patterns

### Do
✅ Use context isolation in BrowserWindow  
✅ Validate all IPC inputs with Zod/Joi  
✅ Use typed IPC handlers with proper TypeScript interfaces  
✅ Implement request origin validation  
✅ Use custom events for notifications rather than direct IPC  

### Don't
❌ Disable context isolation for "convenience"  
❌ Send raw browserWindow instances to renderer  
❌ Use remote module (deprecated and insecure)  
❌ Send large binary data through IPC  
❌ Ignore errors in IPC handlers  

## Debugging

### Developer Tools
```typescript
// main/index.ts
const createWindow = () => {
  const window = new BrowserWindow({
    // ...other options
  });
  
  if (process.env.NODE_ENV === 'development') {
    window.webContents.openDevTools();
  }
};
```

### IPC Monitoring
```typescript
// main/index.ts - Development only
if (process.env.NODE_ENV === 'development') {
  ipcMain.on('*', (event, channel, ...args) => {
    console.log(`[IPC IN] ${channel}:`, args);
  });
  
  ipcMain.handle('*', async (event, channel, ...args) => {
    console.log(`[IPC HANDLE] ${channel}`, args);
    // ... continue with handler
  });
}
```

## References

* [Electron Documentation](https://www.electronjs.org/docs)
* [Electron Security Checklist](https://www.electronjs.org/docs/latest/tutorial/security)
* [Context Isolation Guide](https://www.electronjs.org/docs/latest/tutorial/context-isolation)

## 2026 Best Practices

* **Context Isolation**: Always enabled in BrowserWindow
* **Sandboxed Renderers**: Use sandbox: true for security
* **TypeScript**: Use typed IPC with shared interfaces
* **Input Validation**: Validate ALL IPC inputs
* **Error Handling**: Never expose internal errors to renderer
