---
title: "Elixir LiveView Single Binary"
date: 2025-10-06T21:55:18+02:00
draft: false
comments: true
cover: "img/elixir-liveview-single-binary/screenshot.png"
tags:
  - code
  - elixir
  - tutorial

seo:
  - phoenix liveview desktop app
  - tauri elixir
  - cross-platform elixir
  - phoenix standalone executable
  - burrito elixir
  - elixir desktop application
  - phoenix tauri integration
  - sidecar architecture
seo_description: "Learn how to build a cross-platform desktop application from Phoenix LiveView using Tauri and Burrito. This comprehensive guide covers packaging Phoenix as a standalone executable with embedded ERTS, creating native desktop apps for Linux, macOS, and Windows, and implementing a sidecar architecture for seamless integration."
---

We had two repos of Elixir, seventy-five LiveView components, five sheets of high-powered Phoenix channels, a saltshaker half-full of SQLite databases, and a whole multicolored collection of Burrito deployables, Tauri binaries, WebView wrappers, hot-reloaders... Also, a quart of WebSockets, a quart of GenServers, a case of Ecto queries, a pint of raw BEAM bytecode, and two dozen supervisors.

The only thing that really worried me was the WebView. There is nothing in the world more helpless and irresponsible and depraved than a developer in the depths of a cross-platform binge, and I knew we'd get into that rotten stuff pretty soon.


# or Building a Cross-Platform Desktop App from Phoenix LiveView with Tauri

This guide shows how to package a Phoenix LiveView application as a standalone cross-platform desktop application using Tauri and Burrito, based on the CyCameraCompanion implementation.

## Architecture Overview

The desktop app uses a **sidecar architecture**:
- **Tauri (Rust)**: Native window wrapper and process manager
- **Phoenix Backend**: Bundled as a standalone executable via Burrito
- **Communication**: Phoenix runs on localhost:4000, Tauri webview connects to it

## Prerequisites

- Elixir/Phoenix application (working locally)
- Rust and Cargo installed
- Node.js/npm installed
- For Windows builds from Linux:
  - `cargo-xwin` (version 0.19.2 or compatible)
  - Zig 0.13.0 (other versions may not work - this specific version is required)

## Step 1: Add Burrito for Standalone Phoenix Executables

Burrito wraps Phoenix releases into single executables with embedded ERTS (Erlang Runtime System).

### 1.1 Add Burrito Dependency

Add to `mix.exs`:

```elixir
defp deps do
  [
    # ... existing deps
    {:burrito, "~> 1.1.0"}
  ]
end
```

### 1.2 Configure Releases

Add release configuration in `mix.exs`:

```elixir
def releases() do
  erts_path = System.get_env("MIX_ERTS_PATH") || Mix.env() == :prod

  [
    cy_camera_companion: [
      include_executables_for: [:unix],
      include_erts: erts_path,
      quiet: true
    ],
    cy_camera_companion_desktop: [
      steps: [:assemble, &Burrito.wrap/1],
      burrito: [
        targets: [
          macos: [os: :darwin, cpu: :aarch64],
          linux: [os: :linux, cpu: :x86_64],
          windows: [os: :windows, cpu: :x86_64]
        ]
      ]
    ]
  ]
end
```

### 1.3 Configure Runtime for Production

Update `config/runtime.exs` for production:

```elixir
if config_env() == :prod do
  database_path =
    System.get_env("CY_COMPANION_DATABASE_PATH") ||
    Path.join([System.user_home() || ".", ".cy_camera_companion", "cy_camera_companion.db"])

  # Ensure database directory exists
  database_dir = Path.dirname(database_path)
  File.mkdir_p!(database_dir)

  config :cy_camera_companion, CyCameraCompanion.Repo,
    database: database_path,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "5")

  config :cy_camera_companion, CyCameraCompanionWeb.Endpoint,
    http: [
      ip: {0, 0, 0, 0},
      port: String.to_integer(System.get_env("CY_COMPANION_PORT") || "4000")
    ],
    secret_key_base: System.get_env("SECRET_KEY_BASE") || "your-fallback-key",
    server: true,
    check_origin: false
end
```

### 1.4 Skip Migrations in Burrito Context

Update `lib/cy_camera_companion/application.ex`:

```elixir
children = [
  # ... other children
  {Ecto.Migrator,
   repos: Application.fetch_env!(:cy_camera_companion, :ecto_repos),
   skip: skip_migrations?()},
  # ... other children
]

defp skip_migrations?() do
  System.get_env("BURRITO_TARGET") != nil or
  Application.get_env(:cy_camera_companion, :skip_migrations, false)
end
```

### 1.5 Create Release Module for Migrations

Create `lib/cy_camera_companion/release.ex`:

```elixir
defmodule CyCameraCompanion.Release do
  @moduledoc """
  Used for executing DB release tasks when run in production without Mix
  installed.
  """
  @app :cy_camera_companion

  def migrate do
    load_app()

    for repo <- repos() do
      {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :up, all: true))
    end
  end

  def rollback(repo, version) do
    load_app()
    {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :down, to: version))
  end

  defp repos do
    Application.fetch_env!(@app, :ecto_repos)
  end

  defp load_app do
    Application.load(@app)
  end
end
```

### 1.6 Update .gitignore

Add to `.gitignore`:

```
burrito_out/*
```

## Step 2: Initialize Tauri Project

### 2.1 Create Tauri Directory Structure

```bash
mkdir -p tauri/src-tauri/binaries
cd tauri
```

### 2.2 Initialize Tauri Project

```bash
npm create tauri-app@latest
# Select: src-tauri as the folder name
# Select: Vanilla (no framework)
# Skip frontend setup (we'll use our own)
```

### 2.3 Configure Tauri

Update `tauri/src-tauri/Cargo.toml`:

```toml
[package]
name = "camera_companion"
version = "0.1.0"
description = "CyanView - Camera Companion"
authors = ["you"]
edition = "2021"

[[bin]]
name = "camera-companion"
path = "src/main.rs"

[lib]
name = "tauri_lib"
crate-type = ["staticlib", "cdylib", "rlib"]

[build-dependencies]
tauri-build = { version = "2", features = [] }

[dependencies]
tauri = { version = "2", features = [] }
tauri-plugin-opener = "2"
tauri-plugin-shell = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["time"] }
reqwest = { version = "0.11", features = ["blocking"] }
```

### 2.4 Configure Tauri Settings

Update `tauri/src-tauri/tauri.conf.json`:

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "CY Camera Companion",
  "version": "0.1.0",
  "identifier": "com.cyanview.camera-companion",
  "build": {
    "frontendDist": "../src"
  },
  "app": {
    "withGlobalTauri": true,
    "windows": [],
    "security": {
      "csp": null
    }
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "externalBin": [
      "cy_camera_companion_backend"
    ]
  },
  "plugins": {
    "shell": {
      "open": true
    }
  }
}
```

### 2.5 Create Tauri Main Logic

Update `tauri/src-tauri/src/lib.rs`:

```rust
use tauri::{WebviewUrl, WebviewWindowBuilder};

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .setup(|app| {
            let handle = app.handle().clone();

            // Start the Elixir backend as a sidecar
            tauri::async_runtime::spawn(async move {
                if let Err(e) = start_backend(&handle).await {
                    eprintln!("Failed to start backend: {}", e);
                }
            });

            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}

async fn start_backend(handle: &tauri::AppHandle) -> Result<(), Box<dyn std::error::Error>> {
    use tauri_plugin_shell::ShellExt;

    println!("Starting CY Camera Companion backend...");
    println!("Platform: {}", std::env::consts::OS);
    println!("Architecture: {}", std::env::consts::ARCH);

    // Get the sidecar command for the backend - Tauri will automatically select the right binary
    let sidecar_command = handle
        .shell()
        .sidecar("cy_camera_companion_backend")?
        .env("CY_COMPANION_PORT", "4000")
        .env("MIX_ENV", "prod")
        .env("BURRITO_TARGET", "tauri");

    // Spawn the sidecar process
    let (_rx, _child) = sidecar_command.spawn()?;

    println!("Backend process started, waiting for it to be ready...");

    // Wait for the backend to be ready
    let mut attempts = 0;
    let max_attempts = 60;

    loop {
        attempts += 1;
        if attempts > max_attempts {
            return Err(format!("Backend failed to start after {} attempts", max_attempts).into());
        }

        // Try to connect to the backend
        match reqwest::get("http://localhost:4000").await {
            Ok(response) if response.status().is_success() => {
                println!("Backend is ready after {} attempts!", attempts);
                break;
            }
            Ok(response) => {
                println!("Backend responded with status: {} (attempt {})", response.status(), attempts);
                tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
            }
            Err(e) => {
                if attempts % 5 == 0 {
                    println!("Waiting for backend... (attempt {}/{}): {}", attempts, max_attempts, e);
                }
                tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
            }
        }
    }

    // Open the main window pointing to localhost:4000
    WebviewWindowBuilder::new(
        handle,
        "main",
        WebviewUrl::External("http://localhost:4000".parse().unwrap())
    )
    .title("CY Camera Companion")
    .inner_size(1200.0, 800.0)
    .center()
    .build()?;

    println!("Main window opened successfully");

    Ok(())
}
```

### 2.6 Create Loading Page

Create `tauri/src/index.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>CY Camera Companion - Loading</title>
    <style>
        body {
            margin: 0;
            padding: 20px;
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            background: #1a1a1a;
            color: #fff;
        }
        .loading {
            text-align: center;
        }
        .spinner {
            border: 4px solid #333;
            border-top: 4px solid #0EA5E9;
            border-radius: 50%;
            width: 40px;
            height: 40px;
            animation: spin 1s linear infinite;
            margin: 20px auto;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body>
    <div class="loading">
        <h1>CY Camera Companion</h1>
        <div class="spinner"></div>
        <p>Starting backend service...</p>
        <p id="status">Initializing...</p>
    </div>

    <script>
        let attempts = 0;
        const maxAttempts = 30;

        function checkBackend() {
            attempts++;
            const status = document.getElementById('status');

            if (attempts > maxAttempts) {
                status.textContent = 'Backend failed to start. Please check logs.';
                return;
            }

            status.textContent = `Checking backend... (${attempts}/${maxAttempts})`;

            fetch('http://localhost:4000')
                .then(response => {
                    if (response.ok) {
                        status.textContent = 'Backend ready! Redirecting...';
                        window.location.href = 'http://localhost:4000';
                    } else {
                        setTimeout(checkBackend, 1000);
                    }
                })
                .catch(() => {
                    setTimeout(checkBackend, 1000);
                });
        }

        setTimeout(checkBackend, 3000);
    </script>
</body>
</html>
```

## Step 3: Build Process

### 3.1 Build Phoenix Release with Burrito

Build the standalone Phoenix executable for each platform:

```bash
# From the Phoenix app root directory
MIX_ENV=prod mix release cy_camera_companion_desktop
```

This creates binaries in `burrito_out/`:
- `cy_camera_companion_desktop_linux` (Linux x86_64)
- `cy_camera_companion_desktop_macos` (macOS ARM64)
- `cy_camera_companion_desktop_windows.exe` (Windows x86_64)

### 3.2 Link Binaries to Tauri

Create symlinks from Tauri directory to Burrito output binaries with platform-specific naming:

```bash
# Create symlinks (one-time setup)
cd tauri/src-tauri

# For Linux
ln -s ../../burrito_out/cy_camera_companion_desktop_linux \
   cy_camera_companion_backend-x86_64-unknown-linux-gnu

# For macOS
ln -s ../../burrito_out/cy_camera_companion_desktop_macos \
   cy_camera_companion_backend-aarch64-apple-darwin

# For Windows
ln -s ../../burrito_out/cy_camera_companion_desktop_windows.exe \
   cy_camera_companion_backend-x86_64-pc-windows-msvc.exe

cd ../..
```

**Important**: Tauri expects binaries named with the pattern: `{name}-{target-triple}` or `{name}-{target-triple}.exe` for Windows.

**Why symlinks?** Using symlinks instead of copying means you don't need to manually copy files after each Phoenix release build - the Tauri build will automatically use the latest Burrito output.

### 3.3 Build Tauri Application

#### For Native Platform

```bash
cd tauri
cargo tauri build
```

#### Cross-Compile for Windows from Linux

Install required tools:

```bash
# Install Zig 0.13.0 (IMPORTANT: other versions may not work)
# Download from https://ziglang.org/download/
# Or use version manager like zigup
wget https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz
tar -xf zig-linux-x86_64-0.13.0.tar.xz
sudo mv zig-linux-x86_64-0.13.0 /usr/local/zig-0.13.0
export PATH="/usr/local/zig-0.13.0:$PATH"

# Verify Zig version
zig version  # Should output: 0.13.0

# Install cargo-xwin
cargo install cargo-xwin
```

Build for Windows:

```bash
cd tauri
cargo tauri build --runner cargo-xwin --target x86_64-pc-windows-msvc
```

**Important**: Zig version 0.13.0 is required for cargo-xwin compatibility. Other versions (0.11.x, 0.12.x, 0.14.x) may cause build failures.

The built applications will be in:
- Linux: `tauri/src-tauri/target/release/bundle/`
- Windows: `tauri/src-tauri/target/x86_64-pc-windows-msvc/release/bundle/`
- macOS: `tauri/src-tauri/target/release/bundle/`

## Step 4: Key Implementation Details

### Database Persistence

The app stores the SQLite database in the user's home directory:
```elixir
Path.join([System.user_home() || ".", ".cy_camera_companion", "cy_camera_companion.db"])
```

### Migration Strategy

Migrations are skipped when running in Burrito/Tauri context (detected via `BURRITO_TARGET` env var). You can run migrations manually if needed via the Release module.

### Process Communication

1. Tauri launches the Phoenix backend as a sidecar process
2. Backend starts on `localhost:4000`
3. Tauri polls the endpoint until it responds successfully
4. Tauri opens a webview window pointing to `localhost:4000`
5. Phoenix LiveView handles all UI interactions

### Environment Variables

Key environment variables passed to the backend:
- `CY_COMPANION_PORT`: HTTP port (default: 4000)
- `MIX_ENV`: Set to "prod"
- `BURRITO_TARGET`: Signals running in bundled mode

## Platform-Specific Considerations

### Windows
- Binary must have `.exe` extension
- Use `cargo-xwin` for cross-compilation from Linux
- Target triple: `x86_64-pc-windows-msvc`

### macOS
- Target triple: `aarch64-apple-darwin` (Apple Silicon)
- Code signing may be required for distribution

### Linux
- Target triple: `x86_64-unknown-linux-gnu`
- May need to bundle system libraries

## Troubleshooting

### Backend Fails to Start
- Check logs in Tauri dev tools console
- Verify binary permissions (must be executable)
- Ensure all dependencies are bundled in Burrito release

### Window Opens Before Backend Ready
- Increase polling timeout in `lib.rs`
- Add more detailed logging to debug startup timing

### Database Issues
- Ensure database directory has write permissions
- Check that migrations are handled correctly
- Verify database path resolution in production

## Complete Build Commands Reference

```bash
# Step 1: Build Phoenix releases with Burrito
MIX_ENV=prod mix release cy_camera_companion_desktop

# Step 2: Create symlinks to binaries (one-time setup)
cd tauri/src-tauri
ln -s ../../burrito_out/cy_camera_companion_desktop_linux \
   cy_camera_companion_backend-x86_64-unknown-linux-gnu
ln -s ../../burrito_out/cy_camera_companion_desktop_macos \
   cy_camera_companion_backend-aarch64-apple-darwin
ln -s ../../burrito_out/cy_camera_companion_desktop_windows.exe \
   cy_camera_companion_backend-x86_64-pc-windows-msvc.exe
cd ../..

# Step 3: Build Tauri app
cd tauri

# Native build
cargo tauri build

# Windows cross-compile from Linux
cargo tauri build --runner cargo-xwin --target x86_64-pc-windows-msvc
```

## Result

You now have:
- ✅ Single-binary desktop applications for Linux, macOS, and Windows
- ✅ Native window with system integration
- ✅ Embedded Phoenix LiveView backend
- ✅ No external dependencies required
- ✅ Cross-platform builds from a single development machine

The final application is completely self-contained - users simply download and run the executable.

## Check out

- [www.cyanview.com](https://www.cyanview.com/)
- [CyanView - From Super Bowl to Olympics](https://youtu.be/CUzarY3ry2M?si=KYfJy8Ef1ZQL4vCU)
- [1703 Group](https://1703.lu/)
