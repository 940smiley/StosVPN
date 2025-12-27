## Purpose
Provide concise, actionable instructions to AI coding agents (Copilot-style) so they can be productive working on this repo quickly.

## Big picture
- This repo contains an iOS app `StosVPN` (SwiftUI) and a Network Extension target `TunnelProv` (Packet Tunnel Provider) that implements a local packet-manipulating VPN. Key files:
  - Tunnel implementation: TunnelProv/PacketTunnelProvider.swift
  - UI / manager: StosVPN/ContentView.swift and StosVPN/TunnelManager (in same file)
  - App entry: StosVPN/StosVPNApp.swift

## Architecture summary (what matters)
- Two process boundaries: the main app (`StosVPN`) and packet-tunnel extension (`TunnelProv`). They communicate indirectly via `NETunnelProviderManager` preferences and system APIs (NetworkExtension). See `Bundle.tunnelBundleID` in `ContentView.swift` for bundleID pattern.
- `TunnelProv/PacketTunnelProvider.swift` reads/writes raw packets via `packetFlow.readPackets` / `writePackets` and rewrites IPv4 headers (addresses). Treat that file as the single-source-of-truth for tunnel behavior.
- App-side `TunnelManager` configures `NETunnelProviderManager` objects, on-demand rules, and toggles the tunnel. Most feature logic lives in `StosVPN/ContentView.swift`.

## Build & run (developer workflows)
- Open the workspace in Xcode: open `StosVPN.xcodeproj/project.xcworkspace` and choose the scheme `StosVPN` or `TunnelProv` (shared schemes are in `xcshareddata/xcschemes`).
- To run the extension target for debugging, select the `TunnelProv` scheme in Xcode (the extension must be launched via the system or the app — use the `StosVPN` app scheme to drive config).
- CLI build example:
  - xcodebuild -workspace StosVPN.xcodeproj/project.xcworkspace -scheme StosVPN -configuration Debug
- Signing & entitlements: see `Build.xcconfig` and `CodeSigning.xcconfig.sample` (copy/override into `CodeSigning.xcconfig` with your Team/IDs). Entrust provisioning profiles for the app and the Packet Tunnel extension; entitlements exist at `StosVPN/StosVPN.entitlements` and `TunnelProv/TunnelProv.entitlements`.

## Project-specific patterns & conventions
- Tunnel bundle ID: derived from main bundle + `.TunnelProv` (see `Bundle.tunnelBundleID` in `ContentView.swift`). Use that when creating NETunnelProviderProtocol.providerBundleIdentifier.
- Default tunnel IPs are stored in `UserDefaults` keys: `TunnelDeviceIP`, `TunnelFakeIP`, `TunnelSubnetMask` (read in `TunnelManager`). Agents editing networking behavior should update both app and extension logic consistently.
- Logging: `VPNLogger` collects logs in-memory (in `ContentView.swift`). Use it for debugging UI-visible logs.
- Duplicate NETunnelProviderManager cleanup: `TunnelManager.cleanupDuplicateManagers` removes extra saved configs — preserve this logic when changing preference handling.

## Integration points & external deps
- NetworkExtension APIs: `NEPacketTunnelProvider`, `NETunnelProviderManager`, `NEIPv4Settings`, `NEPacketTunnelNetworkSettings` — consult Apple docs when changing behavior.
- Bridging header exists at `StosVPN-Bridging-Header.h` for any C/ObjC interop; currently empty but retain when adding native libs.
- Localization and assets live under `StosVPN/Assets.xcassets/Localization/*`. Follow existing keys when adding strings.

## When making changes, be conservative about:
- Packet mutation rules in `PacketTunnelProvider.swift` — small changes can break routing or cause crashes due to unsafe memory operations.
- Entitlements and code signing — CI/local builds will fail if provisioning is mismatched. Reference `CodeSigning.xcconfig.sample`.
- NETunnelProviderManager lifecycle — preserve load/save patterns and on-demand rules defined in `TunnelManager`.

## Examples to reference when coding
- Add new tunnel option: mirror how `createStosVPNConfiguration` in `ContentView.swift` builds a `NETunnelProviderProtocol` and saves it.
- Modify packet rewrite: update `setPackets()` in `TunnelProv/PacketTunnelProvider.swift` and update corresponding IP/UserDefaults keys in `ContentView.swift`.

## Questions for reviewers (what the agent should ask)
- Do you want additional unit tests or integration tests for tunnel setup behaviours? (no tests present)
- Should the bridging header be populated with any existing C libraries for performance-sensitive code?

---
If this file is incomplete or you want a different level of detail (more build scripts, CI commands, or test scaffolding), tell me which area to expand. 
