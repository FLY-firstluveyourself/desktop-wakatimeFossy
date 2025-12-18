
# Desktop WakaTime Development Guide

## Architecture Overview

This is an **Electron + React + Vite** desktop app that monitors application usage and sends time tracking data to WakaTime. The architecture splits into three main layers:

- **Electron Main Process** ([electron/main.ts](electron/main.ts)): Manages system tray, windows, IPC handlers, and coordinates watchers
- **Electron Preload** ([electron/preload.ts](electron/preload.ts)): Exposes secure IPC API to renderer via `window.ipcRenderer`
- **React Renderer** ([src/](src/)): Two separate pages (Settings, Monitored Apps) built with React + Tailwind + Radix UI

### Key Data Flow

1. **Watcher** ([electron/watchers/watcher.ts](electron/watchers/watcher.ts)) polls active windows every 2 seconds
2. **AppsManager** ([electron/helpers/apps-manager.ts](electron/helpers/apps-manager.ts)) maintains list of installed apps (cached in `wakatime-apps.json`)
3. **MonitoringManager** ([electron/helpers/monitoring-manager.ts](electron/helpers/monitoring-manager.ts)) checks if app should be tracked (stored in `.wakatime.cfg`)
4. **Wakatime** class ([electron/watchers/wakatime.ts](electron/watchers/wakatime.ts)) sends heartbeats to WakaTime CLI
5. **ConfigFileReader** ([electron/helpers/config-file-reader.ts](electron/helpers/config-file-reader.ts)) reads/writes INI-style `.wakatime.cfg` file

## Critical Patterns

### Adding New Monitored Apps

Edit [electron/watchers/apps.ts](electron/watchers/apps.ts) `allApps` array. Each app needs:
- `id`: unique identifier
- `mac.bundleId` or `windows.exePath` + `DisplayName`
- Optional flags: `isBrowser`, `isElectronApp`, `isDefaultEnabled`

See existing entries like `figma` or `chrome` for examples.

### IPC Communication Pattern

**All** renderer ↔ main communication uses IPC keys defined in [electron/utils/constants.ts](electron/utils/constants.ts):

```typescript
// Add to IpcKeys object
export const IpcKeys = {
  myNewKey: "my_new_key",
  // ...
};

// In main.ts, register handler
ipcMain.on(IpcKeys.myNewKey, (event, arg) => { /* ... */ });

// In preload.ts, expose method
myNewMethod(arg: string) {
  ipcRenderer.send(IpcKeys.myNewKey, arg);
}

// In React component, use
window.ipcRenderer?.myNewMethod(value);
```

### Config File Management

User settings stored in `.wakatime.cfg` (INI format). Use `ConfigFileReader` or `ConfigFile` helpers:
- `ConfigFileReader.get(file, section, key)` - reads string
- `ConfigFileReader.getBool(file, section, key)` - reads boolean
- Never write directly to file, always use helpers

Monitoring state keys follow pattern: `is_${appPath}_monitored` in `[monitoring]` section.

### Platform-Specific Code

Use [electron/helpers/installed-apps/](electron/helpers/installed-apps/) for OS-specific app detection:
- `windows.ts`: Registry scanning via `winreg` package
- `mac.ts`: Bundle scanning and `system_profiler` parsing

Check `process.platform === "darwin"` for macOS, `"win32"` for Windows.

## Development Workflow

```bash
npm i                # Install dependencies
npm run dev          # Run in dev mode (Vite + Electron)
npm run build        # Build production app for current platform
npm run lint         # ESLint check
npm run format       # Prettier formatting
```

**Dev mode**: Vite dev server at `VITE_DEV_SERVER_URL`, hot reload enabled. Main process changes require restart.

**Build output**: Generated in `/release/wakatime-[Platform]-[Arch].[Extension]`

## TypeScript Path Aliases

Defined in [tsconfig.json](tsconfig.json):
- `~/...` → `src/...` (renderer code)
- `electron/...` → `electron/...` (main process code)

## External Dependencies

- **WakaTime CLI**: Auto-downloaded by `Dependencies.installDependencies()` to resources folder
- **@miniben90/x-win**: Gets active window info on Windows
- **electron-store**: Persistent key-value storage (not heavily used, prefer `.wakatime.cfg`)
- **node-global-key-listener**: Keyboard activity detection
- **Radix UI**: Headless component library (Button, Checkbox, Switch, etc.)

## Testing

Shell scripts use **bats** (Bash Automated Testing System):
```bash
brew install bats-core  # macOS setup
bats bin/tests/*.bats   # Run tests
```

See [bin/tests/](bin/tests/) for existing test examples.

## Branch Naming (Required)

PRs must use these prefixes for semver automation:
- `major/...` → breaking changes
- `feature/...` → new features (minor)
- `bugfix/...` → bug fixes (patch)
- `docs/...` or `misc/...` → no version bump

See [CONTRIBUTING.md](CONTRIBUTING.md) for full PR guidelines.

## Common Gotchas

- **Multiple windows**: Settings and Monitored Apps are separate `BrowserWindow` instances, not routes
- **Window icons**: Use `getWindowIcon()` helper (loads from `public/app-icon.png`)
- **Auto-updates**: Handled by `electron-updater` in Wakatime class, checks GitHub releases
- **Deep linking**: `wakatime://` protocol registered in [electron/main.ts](electron/main.ts), handles `settings` and `monitoredApps`
- **Singleton managers**: AppsManager, Logging, Store use `instance()` pattern for global state
