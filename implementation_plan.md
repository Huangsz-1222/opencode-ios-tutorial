# Implementation Plan: Using OpenCode on Apple Devices (Web UI + Tailscale Remote Access)

**Language:** [English](#implementation-plan-english) | [中文](#實作計畫中文)

---

## Implementation Plan (English)

> Last updated: 2026-07-31 | Status: approved and executed
> Post-execution note: per the user's request, the full tutorial was later merged into `README.md` (bilingual EN/ZH), and the separate `opencode-ios-tutorial.md` was removed. The server OS was confirmed as Windows, so the delivered tutorial focuses on Windows + WSL2. Section 6 below reflects the original plan.

### 1. Background & Goals
- The user wants to use OpenCode (the open-source AI coding agent) on iPhone/iPad.
- OpenCode cannot run natively on iOS; the officially supported path is `opencode web` (browser interface / PWA).
- This task produces a follow-along tutorial covering server-side setup and iOS usage, including Tailscale remote access and password protection.

### 2. Assumptions (to be confirmed by the user)
1. OpenCode server OS: macOS, Linux, or Windows + WSL2 (the tutorial was originally planned to focus on macOS/Linux, with Windows + WSL noted as differences).
2. Remote access method: Tailscale (approved by the user).
3. Apple devices: iPhone / iPad, iOS 17+.
4. The user already has an API key for an LLM provider (or plans to use OpenCode Zen).

### 3. Architecture Design
- Server side: `opencode web` binds to 127.0.0.1 by default → switch to `--hostname 0.0.0.0` (or `--mdns`) → accessed through a Tailscale node.
- Authentication: `OPENCODE_SERVER_PASSWORD` (+ `OPENCODE_SERVER_USERNAME`).
- iOS side: open `http://<tailscale-ip>:<port>` in Safari → "Add to Home Screen" to make it a PWA.

```mermaid
flowchart LR
  A[iPhone/iPad Safari PWA] -->|Tailscale VPN| B[Mac / Linux server]
  B --> C[opencode web :4096]
  C --> D[LLM Provider API]
```

### 4. Tutorial Outline (originally opencode-ios-tutorial.md)
1. Prerequisites (server, API key, Tailscale account)
2. Installing OpenCode (macOS / Linux / Windows + WSL commands)
3. Starting `opencode web` (password, hostname, mDNS, auto-start at boot)
4. Tailscale installation and setup (server side)
5. iOS usage (Safari connection, Add to Home Screen, notes)
6. Security notes
7. Troubleshooting (common issues table)

### 5. Security Design
- Password protection is always required; Tailscale ACL restricts which devices can access.
- No direct public exposure; if public access is needed, use Cloudflare Tunnel (noted in the tutorial, not implemented).

### 6. Scope of Changes
- Create 2 files in the workspace root:
  - `implementation_plan.md` (this plan)
  - `opencode-ios-tutorial.md` (main tutorial)
- No existing code is modified (the workspace was empty).

### 7. Verification Approach
- Cross-check every command and flag against the official docs (opencode.ai/docs/web).
- If verifiable locally, actually run the `opencode web` startup and password-protection flow.
- Check that the Tailscale install commands match the current official docs (technical verification).

### 8. Known Limitations & Trade-offs
- iOS PWA has known minor issues (e.g., official issue #35480 safe area); noted in the tutorial, not fixed.
- `opencode web` doesn't support all iOS keyboard shortcuts; Blink Shell is recommended for advanced TUI use (noted in the tutorial).
- No public exposure is implemented; Tailscale already satisfies remote access — simpler and more secure.

---

## 實作計畫（中文）

> 執行後註記（2026-07-31）：依使用者要求，完整教學後來已併入 README.md（中英雙語），獨立的 opencode-ios-tutorial.md 已移除；伺服器 OS 確認為 Windows，故正式教學以 Windows + WSL2 為主。第 6 節為原始計畫內容。

### 1. 背景與目標
- 使用者在 iPhone/iPad 上使用 opencode（開源 AI coding agent）。
- opencode 無法在 iOS 原生執行，官方支援路徑為 `opencode web`（瀏覽器介面 / PWA）。
- 本任務產出一份「可照做」的教學文件，涵蓋伺服器端設定與 iOS 端使用，含 Tailscale 遠端存取與密碼保護。

### 2. 假設（需使用者確認）
1. opencode 伺服器 OS：macOS 或 Linux 或 Windows + WSL2（教學以 macOS/Linux 為主，Windows+WSL 附註差異）。
2. 遠端存取方式：Tailscale（你已同意）。
3. Apple 裝置：iPhone / iPad，iOS 17+。
4. 你已有 LLM provider 的 API key（或計畫用 opencode Zen）。

### 3. 架構設計
- 伺服器端：`opencode web` 預設只綁 127.0.0.1 → 改用 `--hostname 0.0.0.0`（或 `--mdns`）→ 透過 Tailscale 節點存取。
- 認證：`OPENCODE_SERVER_PASSWORD`（+ `OPENCODE_SERVER_USERNAME`）。
- iOS 端：Safari 開啟 `http://<tailscale-ip>:<port>` → 「加入主畫面」變成 PWA。

```mermaid
flowchart LR
  A[iPhone/iPad Safari PWA] -->|Tailscale VPN| B[Mac / Linux 伺服器]
  B --> C[opencode web :4096]
  C --> D[LLM Provider API]
```

### 4. 教學文件大綱（opencode-ios-tutorial.md）
1. 事前準備（伺服器、API key、Tailscale 帳號）
2. 安裝 opencode（macOS / Linux / Windows+WSL 三種指令）
3. 啟動 `opencode web`（密碼、hostname、mDNS、開機自動啟動）
4. Tailscale 安裝與設定（伺服器端）
5. iOS 端使用（Safari 連線、加入主畫面、注意事項）
6. 安全注意事項
7. 疑難排解（常見問題表）

### 5. 安全性設計
- 一律要求密碼保護；Tailscale ACL 限定可存取裝置。
- 不直接暴露公網；如需公網存取，改用 Cloudflare Tunnel（文件附註，不實作）。

### 6. 變更範圍
- 在 workspace 根目錄建立 2 個檔案：
  - `implementation_plan.md`（本計畫書）
  - `opencode-ios-tutorial.md`（教學主文件）
- 不修改任何既有程式碼（目前 workspace 為空）。

### 7. 驗證方式
- 對照官方文件（opencode.ai/docs/web）逐項比對指令與旗標。
- 若本機可驗證，實際跑一次 `opencode web` 啟動與密碼保護流程。
- 檢查 Tailscale 安裝指令與現行官方文件一致（技術細節查證）。

### 8. 已知限制與取捨
- iOS PWA 有已知小問題（如官方 issue #35480 safe area），教學中附註但不處理。
- `opencode web` 不支援 iOS 全鍵盤快捷鍵；進階 TUI 操作建議改用 Blink Shell（教學附註）。
- 不實作公網暴露；Tailscale 已滿足遠端需求，更簡單也更安全。

