# OpenCode on iOS (iPhone/iPad) from Windows

Use [OpenCode](https://opencode.ai) — the open-source AI coding agent — from your **iPhone or iPad**, with the OpenCode server running on a **Windows** machine.

OpenCode cannot run natively on iOS (the sandbox forbids the runtime/filesystem/shell it needs), so this repo provides a step-by-step, verified tutorial that turns your Windows PC into a remote OpenCode server and connects to it from Safari as a PWA:

```
iPhone / iPad (Safari PWA)
   ├─ Same Wi-Fi  → direct LAN access
   └─ Anywhere    → Tailscale encrypted tunnel
        ▼
Windows machine
   ├─ Tailscale (stable virtual IP)
   └─ WSL2 Ubuntu (mirrored networking, shares Windows IP)
        └─ opencode web --hostname 0.0.0.0 --port 4096
```

## Contents

| File | Description |
|---|---|
| [opencode-ios-tutorial.md](opencode-ios-tutorial.md) | Full tutorial (Traditional Chinese): setup, security, iOS usage, troubleshooting, and appendices (auto-start, NAT/portproxy fallback, native Windows alternative) |
| [implementation_plan.md](implementation_plan.md) | The implementation plan reviewed and approved before writing the tutorial |

## Quick Start (TL;DR)

1. Install WSL2 + Ubuntu: `wsl --install` (admin PowerShell), then reboot.
2. Enable mirrored networking in `C:\Users\<you>\.wslconfig`:

   ```ini
   [wsl2]
   networkingMode=mirrored
   ```

   then `wsl --shutdown`.
3. Allow inbound port 4096 (admin PowerShell):

   ```powershell
   New-NetFirewallHyperVRule -Name "opencode-web" -DisplayName "OpenCode Web" -Direction Inbound -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -Protocol TCP -LocalPorts 4096
   ```

4. Install OpenCode inside WSL:

   ```bash
   curl -fsSL https://opencode.ai/install | bash
   ```

5. Set a password, then start the web server:

   ```bash
   export OPENCODE_SERVER_PASSWORD="your-strong-password"
   opencode web --hostname 0.0.0.0 --port 4096
   ```

6. Install [Tailscale](https://tailscale.com/download/windows) on Windows, sign in, then open `http://<tailscale-ip>:4096` in Safari on your iPhone/iPad and "Add to Home Screen" to use it as an app.

All commands are cross-checked against the official OpenCode docs, Microsoft Learn, and Tailscale docs (links included in the tutorial).

## Security Notes

- **Always** set `OPENCODE_SERVER_PASSWORD` when binding to `0.0.0.0`.
- Do not port-forward port 4096 to the public internet; use Tailscale (or Cloudflare Tunnel if public access is truly required).

## License

[MIT](LICENSE)
