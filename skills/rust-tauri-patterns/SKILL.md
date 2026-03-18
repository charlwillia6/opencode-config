---
name: rust-tauri-patterns
description: Modern Rust development patterns with Tauri 2.0 integration. Includes CLI tools, state management, and best practices for desktop applications.
---

# Rust + Tauri 2.0 Patterns (2026)

## Project Structure

```
src-tauri/
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── commands/
│   │   ├── mod.rs
│   │   ├── device.rs
│   │   ├── settings.rs
│   │   └── analytics.rs
│   ├── states/
│   │   ├── mod.rs
│   │   └── device_manager.rs
│   ├── services/
│   │   ├── mod.rs
│   │   └── updatable.rs
│   └── utils/
│       ├── mod.rs
│       └── validation.rs
├── build.rs
├── Cargo.toml
└── tauri.conf.json5
```

## Tauri 2.0 Command Definitions

### Modern Command Pattern
```rust
// src-tauri/src/commands/device.rs
use tauri::{Manager, cmd};
use crate::states::device_manager::DeviceManager;

#[cmd]
pub fn get_devices(_app: tauri::AppHandle, state: tauri::State<DeviceManager>) -> Vec<Device> {
    state.get_devices()
}

#[cmd]
pub fn add_device(
    app: tauri::AppHandle,
    state: tauri::State<DeviceManager>,
    name: String
) -> Result<Device, String> {
    let device = state.add_device(name)?;
    
    // Emit event to all windows
    app.emit_all("device-added", &device)
        .map_err(|e| format!("Failed to emit event: {}", e))?;
    
    Ok(device)
}

#[cmd]
pub fn update_device_status(
    _app: tauri::AppHandle,
    state: tauri::State<DeviceManager>,
    id: String,
    status: DeviceStatus
) -> Result<Device, String> {
    state.update_status(&id, status)
}

#[cmd]
pub fn delete_device(
    app: tauri::AppHandle,
    state: tauri::State<DeviceManager>,
    id: String
) -> Result<(), String> {
    let device = state.get_device(&id)?;
    state.delete_device(&id)?;
    
    app.emit_all("device-deleted", &device)
        .map_err(|e| format!("Failed to emit event: {}", e))?;
    
    Ok(())
}
```

### Command Error Handling
```rust
// src-tauri/src/utils/validation.rs
use tauri::Manager;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DeviceError {
    #[error("Device not found: {0}")]
    NotFound(String),
    
    #[error("Invalid device name: {0}")]
    InvalidName(String),
    
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
}

impl From<DeviceError> for String {
    fn from(err: DeviceError) -> Self {
        err.to_string()
    }
}

pub fn validate_device_name(name: &str) -> Result<(), DeviceError> {
    if name.is_empty() {
        return Err(DeviceError::InvalidName("Name cannot be empty".to_string()));
    }
    
    if name.len() > 100 {
        return Err(DeviceError::InvalidName("Name exceeds maximum length of 100 characters".to_string()));
    }
    
    Ok(())
}
```

## State Management

### Application State
```rust
// src-tauri/src/states/device_manager.rs
use std::collections::HashMap;
use tauri::{Manager, State};
use serde::{Deserialize, Serialize};
use anyhow::Result;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Device {
    pub id: String,
    pub name: String,
    pub status: DeviceStatus,
    pub created_at: String,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
pub enum DeviceStatus {
    Connected,
    Disconnected,
    Syncing,
    Error,
}

pub struct DeviceManager {
    devices: std::sync::Mutex<HashMap<String, Device>>,
}

impl DeviceManager {
    pub fn get_devices(&self) -> Vec<Device> {
        self.devices.lock().unwrap().values().cloned().collect()
    }

    pub fn get_device(&self, id: &str) -> Option<Device> {
        self.devices.lock().unwrap().get(id).cloned()
    }

    pub fn add_device(&self, name: String) -> Result<Device> {
        validate_device_name(&name)?;
        
        let id = uuid::Uuid::new_v4().to_string();
        let device = Device {
            id: id.clone(),
            name,
            status: DeviceStatus::Disconnected,
            created_at: chrono::Utc::now().to_rfc3339(),
        };
        
        self.devices.lock().unwrap().insert(id, device.clone());
        Ok(device)
    }

    pub fn update_status(&self, id: &str, new_status: DeviceStatus) -> Result<Device> {
        let mut devices = self.devices.lock().unwrap();
        let device = devices.get_mut(id)
            .ok_or_else(|| anyhow::anyhow!("Device not found"))?;
        
        device.status = new_status;
        Ok(device.clone())
    }

    pub fn delete_device(&self, id: &str) -> Result<()> {
        let mut devices = self.devices.lock().unwrap();
        devices.remove(id)
            .ok_or_else(|| anyhow::anyhow!("Device not found"))?;
        Ok(())
    }
}

impl Default for DeviceManager {
    fn default() -> Self {
        Self {
            devices: std::sync::Mutex::new(HashMap::new()),
        }
    }
}
```

## Service Layer

### Background Services
```rust
// src-tauri/src/services/updatable.rs
use tauri::{AppHandle, Manager, Service};
use anyhow::Result;

pub struct UpdatableService;

impl Service for UpdatableService {
    fn Name(&self) -> &'static str {
        "updatable"
    }

    fn Start(&self, app: &AppHandle) -> Result<()> {
        let appclone = app.clone();
        
        // Start periodic update check
        std::thread::spawn(move || {
            loop {
                std::thread::sleep(std::time::Duration::from_secs(3600)); // Check hourly
                
                // Check for updates
                let update_available = check_for_updates();
                
                if update_available {
                    appclone.emit_all("update-available", true).ok();
                }
            }
        });

        Ok(())
    }

    fn Stop(&self, app: &AppHandle) -> Result<()> {
        // Cleanup logic
        Ok(())
    }
}

fn check_for_updates() -> bool {
    // Implement update checking logic
    false
}
```

## Application Entry Point

```rust
// src-tauri/src/main.rs
mod commands;
mod states;
mod services;
mod utils;

use tauri::Builder;
use states::device_manager::DeviceManager;
use services::updatable::UpdatableService;

fn main() {
    tauri::Builder::default()
        .setup(|app| {
            // Initialize state
            app.manage(DeviceManager::default());
            
            // Register services
            app.register_service(Box::new(UpdatableService));
            
            Ok(())
        })
        .plugin(tauri_plugin_log::Builder::default().build())
        .plugin(tauri_plugin_window_state::Builder::default().build())
        .invoke_handler(tauri::generate_handler![
            commands::device::get_devices,
            commands::device::add_device,
            commands::device::update_device_status,
            commands::device::delete_device,
            commands::settings::get_settings,
            commands::settings::save_settings,
            commands::analytics::record_event,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}

#[cfg(target_os = "macos")]
fn main() {
    tauri::Builder::default()
        .setup(|app| {
            app.manage(DeviceManager::default());
            Ok(())
        })
        .plugin(tauri_plugin_log::Builder::default().build())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

## Frontend Integration

### TypeScript Definitions
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

#[cfg(not(any(target_os = "android", target_os = "ios")))]
fn main() {
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

### React Component
```typescript
// src/features/devices/components/DeviceManager.tsx
import { useState, useEffect } from 'react';
import { invoke } from '@tauri-apps/api/core';
import { listen } from '@tauri-apps/api/event';

interface Device {
  id: string;
  name: string;
  status: 'Connected' | 'Disconnected' | 'Syncing' | 'Error';
}

export function DeviceManager() {
  const [devices, setDevices] = useState<Device[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDevices();
    setupEventListeners();
  }, []);

  async function loadDevices() {
    try {
      const result = await invoke<Device[]>('get_devices');
      setDevices(result);
    } catch (error) {
      console.error('Failed to load devices:', error);
    } finally {
      setLoading(false);
    }
  }

  async function addDevice(name: string) {
    try {
      const device = await invoke<Device>('add_device', { name });
      setDevices(prev => [...prev, device]);
    } catch (error) {
      console.error('Failed to add device:', error);
    }
  }

  async function updateDeviceStatus(id: string, status: Device['status']) {
    try {
      await invoke('update_device_status', { id, status });
      setDevices(prev => 
        prev.map(d => d.id === id ? { ...d, status } : d)
      );
    } catch (error) {
      console.error('Failed to update device:', error);
    }
  }

  async function deleteDevice(id: string) {
    try {
      await invoke('delete_device', { id });
      setDevices(prev => prev.filter(d => d.id !== id));
    } catch (error) {
      console.error('Failed to delete device:', error);
    }
  }

  function setupEventListeners() {
    listen<Device>('device-added', (event) => {
      setDevices(prev => [...prev, event.payload]);
    });

    listen<Device>('device-deleted', (event) => {
      setDevices(prev => prev.filter(d => d.id !== event.payload.id));
    });
  }

  if (loading) {
    return <div>Loading devices...</div>;
  }

  return (
    <div className="device-manager">
      <h1>Device Manager</h1>
      
      <button onClick={() => addDevice('New Device')}>+ Add Device</button>
      
      <ul>
        {devices.map(device => (
          <li key={device.id}>
            <span>{device.name}</span>
            <select
              value={device.status}
              onChange={(e) => updateDeviceStatus(device.id, e.target.value as Device['status'])}
            >
              <option value="Connected">Connected</option>
              <option value="Disconnected">Disconnected</option>
              <option value="Syncing">Syncing</option>
              <option value="Error">Error</option>
            </select>
            <button onClick={() => deleteDevice(device.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## Error Handling Pattern

### Comprehensive Error Handling
```rust
// src-tauri/src/utils/error.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Invalid input: {0}")]
    InvalidInput(String),
    
    #[error("Operation failed: {0}")]
    OperationFailed(String),
    
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Tauri error: {0}")]
    Tauri(#[from] tauri::Error),
}

impl From<AppError> for String {
    fn from(err: AppError) -> Self {
        match err {
            AppError::InvalidInput(msg) => msg,
            AppError::OperationFailed(msg) => format!("Operation failed: {}", msg),
            AppError::Database(err) => format!("Database error: {}", err),
            AppError::Tauri(err) => format!("Application error: {}", err),
        }
    }
}

// Usage in commands
#[cmd]
pub fn do_something() -> Result<String, String> {
    something_that_can_fail()
        .map_err(|e| AppError::from(e).into())
}
```

## Testing

### Unit Tests
```rust
// src-tauri/src/tests/device_manager_tests.rs
use crate::states::device_manager::{DeviceManager, DeviceStatus};

#[test]
fn test_add_device() {
    let manager = DeviceManager::default();
    
    let device = manager.add_device("Test Device".to_string()).unwrap();
    
    assert_eq!(device.name, "Test Device");
    assert_eq!(device.status, DeviceStatus::Disconnected);
    assert!(!device.id.is_empty());
}

#[test]
fn test_update_device_status() {
    let manager = DeviceManager::default();
    let device = manager.add_device("Test Device".to_string()).unwrap();
    
    let updated = manager.update_status(&device.id, DeviceStatus::Connected).unwrap();
    
    assert_eq!(updated.status, DeviceStatus::Connected);
}

#[test]
fn test_delete_device() {
    let manager = DeviceManager::default();
    let device = manager.add_device("Test Device".to_string()).unwrap();
    
    manager.delete_device(&device.id).unwrap();
    
    assert!(manager.get_device(&device.id).is_none());
}
```

## Common Patterns & Anti-Patterns

### Do
✅ Use State<T> for sharing mutable state  
✅ Implement comprehensive error handling  
✅ Use tauri::cmd macro for clean command definitions  
✅ Emit events for real-time updates  
✅ Validate all command inputs  
✅ Use modules for organizing command logic  

### Don't
❌ Share browserWindow references between handlers  
❌ Use blocking I/O in commands (use async)  
❌ Forget to register services  
❌ Skip error handling in production  
❌ Hardcode values in commands  

## References

* [Tauri 2.0 Documentation](https://tauri.app)
* [Rust Book](https://doc.rust-lang.org/book/)
* [Tauri Patterns](https://tauri.app/v2/guides/architecture/)

## 2026 Best Practices

* **Tauri 2.0**: Use modern `#[cmd]` macro
* **State Management**: Use `State<T>` for application-wide state
* **Error Handling**: Use `thiserror` for structured errors
* **Command Organization**: Split commands by domain
* **Event System**: Use events for real-time UI updates
