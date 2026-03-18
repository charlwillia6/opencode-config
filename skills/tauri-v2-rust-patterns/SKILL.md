---
name: tauri-v2-rust-patterns
description: Modern Tauri 2.0 application development with Rust. Includes command structure, state management, and pattern best practices.
---

# Tauri 2.0 + Rust Patterns (2026)

## Project Structure

```
app/
├── src-tauri/
│   ├── src/
│   │   ├── main.rs               # Application entry
│   │   ├── commands/             # Tauri commands
│   │   │   ├── mod.rs
│   │   │   ├── auth.rs
│   │   │   ├── device.rs
│   │   │   └── analytics.rs
│   │   ├── states/               # Application state
│   │   │   ├── mod.rs
│   │   │   └── device_manager.rs
│   │   ├── utils/                # Helper functions
│   │   │   ├── mod.rs
│   │   │   ├── file.rs
│   │   │   └── validation.rs
│   │   └── lib.rs                # Library exports
│   ├── build.rs                  # Build configuration
│   ├── Cargo.toml
│   └── tauri.conf.json5
└── src/                          # Frontend code
    ├── features/
    └── providers.tsx
```

## Tauri 2.0 Key Concepts

### Commands (No Longer Async by Default)
```rust
// src-tauri/src/commands/device.rs
use tauri::{Command, State};
use crate::states::device_manager::DeviceManager;

#[cmd]
pub fn get_devices(_app: AppHandle, state: State<DeviceManager>) -> Vec<Device> {
    state.get_devices()
}

#[cmd]
pub fn add_device(_app: AppHandle, state: State<DeviceManager>, name: String) -> Result<Device, String> {
    state.add_device(name)
}
```

### State Management
```rust
// src-tauri/src/states/device_manager.rs
use std::collections::HashMap;
use tauri::{State, Manager};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Device {
    pub id: String,
    pub name: String,
    pub status: DeviceStatus,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
pub enum DeviceStatus {
    Online,
    Offline,
    Connecting,
}

pub struct DeviceManager {
    devices: Mutex<HashMap<String, Device>>,
}

impl DeviceManager {
    pub fn get_devices(&self) -> Vec<Device> {
        self.devices.lock().unwrap().values().cloned().collect()
    }

    pub fn add_device(&self, name: String) -> Result<Device, String> {
        let id = nanoid::nanoid!();
        let device = Device {
            id,
            name,
            status: DeviceStatus::Offline,
        };
        self.devices.lock().unwrap().insert(id.clone(), device.clone());
        Ok(device)
    }
}

impl Default for DeviceManager {
    fn default() -> Self {
        Self {
            devices: Mutex::new(HashMap::new()),
        }
    }
}
```

## Frontend Integration

### TypeScript Type Generation
```typescript
// src-tauri/src/lib.rs
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_log::Builder::default().build())
        .setup(|app| {
            app.manage(DeviceManager::default());
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}

#[cfg_desktop]
pub fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_log::Builder::default().build())
        .setup(|app| {
            app.manage(DeviceManager::default());
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### React Component Integration
```typescript
// src/features/devices/components/DeviceList.tsx
import { useState, useEffect } from 'react';
import { invoke } from '@tauri-apps/api/core';
import { Device } from '@/types/device';

interface DeviceListProps {
  onDeviceClick: (device: Device) => void;
}

export function DeviceList({ onDeviceClick }: DeviceListProps) {
  const [devices, setDevices] = useState<Device[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDevices();
  }, []);

  async function loadDevices() {
    try {
      const result = await invoke<{ devices: Device[] }>('get_devices');
      setDevices(result.devices);
    } catch (error) {
      console.error('Failed to load devices:', error);
    } finally {
      setLoading(false);
    }
  }

  async function handleAddDevice() {
    const name = prompt('Enter device name:');
    if (name) {
      try {
        const device = await invoke<Device>('add_device', { name });
        setDevices((prev) => [...prev, device]);
      } catch (error) {
        console.error('Failed to add device:', error);
      }
    }
  }

  if (loading) {
    return <div>Loading devices...</div>;
  }

  return (
    <div className="device-list">
      <button onClick={handleAddDevice}>+ Add Device</button>
      <ul>
        {devices.map((device) => (
          <li key={device.id} onClick={() => onDeviceClick(device)}>
            <span>{device.name}</span>
            <span className={`status status-${device.status.toLowerCase()}`}>
              {device.status}
            </span>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## Build Configuration

### tauri.conf.json5
```json5
{
  "$schema": "https://schema.tauri.app/config/2.0",
  "productName": "MyApp",
  "version": "0.1.0",
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/256x256.png",
      "icons/512x512.png",
      "icons/1024x1024.png"
    ]
  },
  "plugins": {
    "log": true,
    "updater": true
  },
  "security": {
    "csp": "default-src 'self'; connect-src 'self' http://localhost:*; img-src 'self' data:"
  }
}
```

### Cargo.toml
```toml
[package]
name = "myapp"
version = "0.1.0"
description = "A Tauri application"
authors = ["Your Name"]
edition = "2021"

[lib]
name = "myapp_lib"
crate-type = ["staticlib", "cdylib"]

[dependencies]
tauri = { version = "2.0", features = ["api-all"] }
tauri-plugin-log = "2.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
thiserror = "2.0"
anyhow = "1.0"
tokio = { version = "1.35", features = ["full"] }
 nanoid = "2.0"
uuid = { version = "1.6", features = ["v4"] }

[target.'cfg(desktop)'.dependencies]
tauri-plugin-updater = "2.0"
tauri-plugin-autostart = "2.0"
tauri-plugin-shell = "2.0"

[features]
default = ["custom-protocol"]
custom-protocol = ["tauri/custom-protocol"]
```

## Security Best Practices

### Feature Control
```rust
// feat: devices allowlist
tauri::Builder::default()
    .plugin(tauri_plugin_log::Builder::default().build())
    .invoke_handler(tauri::generate_handler![
        commands::device::get_devices,
        commands::device::add_device,
        commands::device::update_device,
    ])
    .build(tauri::generate_context!())
    .expect("error while building tauri application");
```

### Event System
```rust
// Emit events to frontend
app.emit_all("device-updated", serde_json::json!({
    "id": device_id,
    "status": "online"
}))?;

// Listen to events in frontend
import { listen } from '@tauri-apps/api/event';

useEffect(() => {
  const unlisten = listen<string>('device-updated', (event) => {
    console.log('Device updated:', event.payload);
    // Update local state
  });
  
  return () => unlisten();
}, []);
```

## Build Scripts

### build.rs
```rust
fn main() {
    tauri_build::build()
}
```

### Development Commands
```bash
# Install dependencies
cargo tauri add log
cargo tauri add updater

# Run development
cargo tauri dev

# Build for production
cargo tauri build

# Run specific target
cargo tauri build --target aarch64-apple-darwin

# Run with specific profile
cargo tauri dev -- --release
```

## Common Patterns & Anti-Patterns

### Do
✅ Use State management for application-wide data  
✅ Group commands by domain (device.rs, auth.rs, etc.)  
✅ Use Tauri 2.0's new command macro `#[cmd]`  
✅ Emit events for real-time updates  
✅ Use Rust type system for validation  

### Don't
❌ Pass large binary data between Rust and JS  
❌ Use blocking operations in commands (use async)  
❌ Store secrets in config files  
❌ Skip validation in Rust commands  
❌ Mix UI logic with command logic  

## Migration from Tauri 1.x

### Key Changes
| Tauri 1.x | Tauri 2.0 |
|-----------|-----------|
| `tauri::async_fn!` | `#[cmd]` macro |
| `AppHandle` in handlers | State management |
| `Event` system | `app.emit_all()` |
| `Window` in commands | `Window` parameter |

### Example Migration
```rust
// Tauri 1.x
#[command]
async fn get_data(app: AppHandle) -> String {
    // ...
}

// Tauri 2.0
#[cmd]
fn get_data(state: State<MyState>) -> String {
    state.get_data()
}
```

## References

* [Tauri 2.0 Documentation](https://tauri.app)
* [Tauri Migration Guide](https://tauri.app/v2/guides/migration/)
* [Rust Book](https://doc.rust-lang.org/book/)
* [serde Documentation](https://serde.rs/)

## 2026 Best Practices

* **Tauri 2.0**: Use `#[cmd]` macro for commands
* **State Management**: Use `State<T>` for shared state
* **Cargo Features**: Split functionality with features
* **Async Commands**: Use Tokio for I/O operations
* **Type Safety**: Leverage Rust's type system for validation
