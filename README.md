# OpenCode on iOS (iPhone/iPad) from Windows

Use [OpenCode](https://opencode.ai) — the open-source AI coding agent — from your **iPhone or iPad**, with the OpenCode server running on a **Windows** machine.

OpenCode cannot run natively on iOS (the sandbox forbids the runtime/filesystem/shell it needs), so this repository provides a step-by-step, verified tutorial that turns your Windows PC into a remote OpenCode server and connects to it from Safari as a PWA:

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

**The complete tutorial is available in two languages:** [English](#english-tutorial-full) | [中文（繁體）](#中文完整教學)

All commands are cross-checked against the official OpenCode docs, Microsoft Learn, and Tailscale docs (links are included at the end of each language version).

**Repository files:** the full tutorial lives in this README (both languages); `implementation_plan.md` records the approved plan (also bilingual EN/ZH); `LICENSE` is the MIT license.

---

## English Tutorial (Full)

> Last updated: 2026-07-31 | Target: Windows 11 (22H2 or later) + iPhone / iPad
> Sources: official OpenCode docs, Microsoft Learn, Tailscale docs (links at the bottom)

### 0. How this works

OpenCode is an agent that runs on a real computer — iPhones and iPads cannot install it directly (the iOS sandbox does not allow the Node/Bun runtime, filesystem access, or shell execution it needs). The approach is therefore:

```
iPhone / iPad (Safari / PWA)
   │  ① Same Wi-Fi: connect directly on the LAN
   │  ② Away from home: Tailscale encrypted tunnel
   ▼
Windows machine
   ├─ Tailscale (Windows side, provides a stable virtual IP)
   └─ WSL2 Ubuntu (mirrored networking mode, shares IP with Windows)
        └─ opencode web --hostname 0.0.0.0 --port 4096
              └─ Accesses project files → calls LLM APIs
```

Key technical decisions (all backed by official docs):
- **Why WSL**: OpenCode's official docs state that WSL gives the best experience on Windows and recommend running `opencode web` from the WSL terminal ([OpenCode Windows docs](https://opencode.ai/docs/windows-wsl)).
- **Why mirrored networking mode**: WSL's default NAT mode gets a new IP on every reboot; mirrored mode makes WSL and Windows share network interfaces, so the Tailscale IP can reach services inside WSL directly ([Microsoft docs](https://learn.microsoft.com/en-us/windows/wsl/networking)).

### 1. Prerequisites (checklist)

- [ ] Windows 11 (22H2 or later; for older versions use Appendix B)
- [ ] WSL2 + Ubuntu (`wsl --install` installs it — see 2.1)
- [ ] A free Tailscale account (sign up at https://tailscale.com)
- [ ] An API key for your LLM provider (or plan to use OpenCode Zen built-in models)
- [ ] iPhone / iPad (iOS 17+ for the best PWA experience)

### 2. Server-side setup (on Windows)

#### 2.1 Install WSL2 + Ubuntu (if not installed yet)

Open PowerShell as **administrator** and run:

```powershell
wsl --install
```

- This installs WSL2 and Ubuntu automatically; **reboot** when prompted.
- After the reboot, set up the Ubuntu username and password, and remember them.
- Confirm the version:

```powershell
wsl --set-default-version 2
wsl -l -v
```

Ubuntu should show VERSION 2.

#### 2.2 Enable mirrored networking mode (important)

Create the file `C:\Users\<your-username>\.wslconfig` with this content:

```ini
[wsl2]
networkingMode=mirrored
```

Then shut down WSL from PowerShell:

```powershell
wsl --shutdown
```

Reopen the WSL terminal (Start menu → Ubuntu). With mirrored mode enabled, WSL and Windows share the same network interfaces, so **the Tailscale IP covers services running inside WSL**.

> If `.wslconfig` doesn't take effect, make sure the filename is `.wslconfig` (leading dot, no extension) in the root of your user profile folder.

#### 2.3 Allow inbound connections on port 4096 (Hyper-V firewall)

Open PowerShell as **administrator** and run (this is the approach from Microsoft's official docs):

```powershell
New-NetFirewallHyperVRule -Name "opencode-web" -DisplayName "OpenCode Web" -Direction Inbound -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -Protocol TCP -LocalPorts 4096
```

#### 2.4 Install OpenCode inside WSL

Open the Ubuntu terminal and run:

```bash
curl -fsSL https://opencode.ai/install | bash
source ~/.bashrc
opencode --version
```

- The installer downloads a prebuilt binary — **no separate Node.js install needed** (installed to `~/.opencode/bin`).
- Success looks like a version number (e.g., 1.18.x).

#### 2.5 Set a login password (skipping this = anyone who can reach it can use it)

Append the password to WSL's shell config so it loads automatically every time:

```bash
echo 'export OPENCODE_SERVER_PASSWORD="replace-with-a-strong-password"' >> ~/.bashrc
echo 'export OPENCODE_SERVER_USERNAME=opencode' >> ~/.bashrc
source ~/.bashrc
```

- The username defaults to `opencode`; you can change it.
- Use 12+ characters and don't reuse a password from another account.

#### 2.6 Start the web server

Switch to the project directory you want OpenCode to work on (recommended: inside the WSL filesystem, e.g., `~/code/your-project`; you can also read Windows drives via `/mnt/c/...`, but it's slower):

```bash
cd ~/code/your-project
opencode web --hostname 0.0.0.0 --port 4096
```

**Verify**: open `http://localhost:4096` in your Windows browser — you should see a login screen; enter `opencode` / your password.

> To make it discoverable as `opencode.local` on the same Wi-Fi, add the `--mdns` flag (it also binds to 0.0.0.0 automatically).

#### 2.7 Install Tailscale (connect from anywhere)

1. Download and install the Windows Tailscale client: https://tailscale.com/download/windows (or run `winget install Tailscale.Tailscale` in PowerShell).
2. Sign in to your Tailscale account (a browser authorization page opens).
3. Get your Tailscale IP in PowerShell:

```powershell
tailscale ip -4
```

You'll get something like `100.101.102.103` — **this is the address your phone can always reach, on any network**.

4. (Optional) Tailscale on Windows starts with your login by default, so no extra configuration is needed.

### 3. Using it on iOS

#### 3.1 First connection

Open in Safari on your iPhone/iPad:

- Same Wi-Fi: `http://<Windows LAN IP>:4096` (e.g., `http://192.168.1.5:4096`; if unsure, run `ipconfig` in PowerShell)
- Away from home / cellular: `http://<Tailscale IP>:4096` (e.g., `http://100.101.102.103:4096`)

Enter username `opencode` and your password, and you can start chatting and letting the agent edit code.

#### 3.2 Add to Home Screen (make it an "app")

1. In Safari, tap the **Share** button → **Add to Home Screen**.
2. Name it (e.g., OpenCode) → **Add**.
3. From now on, open it from your Home Screen to run as a fullscreen PWA that feels like an app.

#### 3.3 Usage notes

- Good for: typing instructions, reviewing the agent's edits, viewing diffs, tracking progress.
- Not ideal for: advanced TUI keyboard shortcuts — for that, use Blink Shell (an iOS SSH/mosh client) to SSH into WSL and run the `opencode` terminal UI.
- iOS suspends Safari when backgrounded, which disconnects the session. The official web UI has auto-reconnect logic, but **don't leave it backgrounded too long**; if the screen seems stuck when you return, just refresh.
- A few minor PWA UI quirks on iOS are known (e.g., safe-area overlap) — tracked in official issues, not blocking core functionality.

### 4. Security notes

- `--hostname 0.0.0.0` without a password = **anyone on the LAN** can operate your agent; always set the password.
- Tailscale's default ACL only allows connections between **your own devices**; don't share nodes casually.
- Do not port-forward 4096 to the public internet via your router; if you truly need public access, use Cloudflare Tunnel or Tailscale Serve instead.
- The password is stored in plaintext in WSL's `~/.bashrc` — acceptable on a personal machine; on a shared machine, use a systemd service with restricted permissions (Appendix A).

### 5. Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| Works in the Windows browser but not on the phone | ① Hyper-V firewall rule missing (2.3) ② Phone and PC on different networks (use the Tailscale IP) ③ Tailscale not signed in |
| Phone connects but shows "not secure" | Using http:// instead of https:// — normal; Safari allows LAN IPs; if blocked, use the Tailscale IP |
| Forgot the password | Edit `~/.bashrc`, re-export a new password, restart `opencode web` |
| `opencode.local` not found | mDNS may be blocked on some networks (e.g., public Wi-Fi); use an IP instead |
| Can't connect after reboot | OpenCode in WSL doesn't auto-start (see Appendix A); the Tailscale IP doesn't change, so just restart `opencode web` |
| Phone PWA layout is off | Known iOS PWA issue (official issue [#35480](https://github.com/anomalyco/opencode/issues/35480) etc.); opening in Safari directly works around it |
| Want the full terminal TUI | Install Blink Shell on iOS, SSH into WSL (`ssh user@Windows-IP`), then run `opencode` |

### Appendix A: Auto-start opencode web at boot (optional)

Ubuntu in WSL supports systemd (enable `[boot] systemd=true` in `wsl.conf` first), then create a service:

```bash
sudo tee /etc/systemd/system/opencode-web.service > /dev/null <<'EOF'
[Unit]
Description=OpenCode Web Server
After=network-online.target

[Service]
Environment=OPENCODE_SERVER_USERNAME=opencode
Environment=OPENCODE_SERVER_PASSWORD=your-strong-password
WorkingDirectory=/home/your-user/code/your-project
ExecStart=/home/your-user/.opencode/bin/opencode web --hostname 0.0.0.0 --port 4096
Restart=always
User=your-user

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now opencode-web
```

> The password is written into the service file — restrict permissions with `sudo chmod 600`.

### Appendix B: Windows 10 / when mirrored mode isn't available (NAT + portproxy)

1. Skip the `.wslconfig` step in 2.2 (NAT mode).
2. Get WSL's own IP with `hostname -I` (e.g., `172.20.10.5`).
3. As administrator in PowerShell, set up port forwarding:

```powershell
netsh interface portproxy add v4tov4 listenport=4096 listenaddress=0.0.0.0 connectport=4096 connectaddress=172.20.10.5
```

4. As administrator in PowerShell, allow inbound 4096 in Windows Firewall:

```powershell
New-NetFirewallRule -DisplayName "OpenCode Web" -Direction Inbound -Protocol TCP -LocalPort 4096 -Action Allow
```

5. Install Tailscale on Windows — the portproxy forwards traffic into WSL.
6. **Note**: WSL's NAT IP can change on every reboot; after a reboot, rerun step 3.

### Appendix C: Fastest alternative without WSL (native Windows)

OpenCode actually runs directly on Windows (officially supported, though WSL is recommended):

```powershell
npm install -g opencode-ai   # requires Node.js first (LTS 20+ recommended)
opencode web --hostname 0.0.0.0 --port 4096
```

- Also add a Windows Defender Firewall inbound rule for 4096, and install Tailscale as in 2.7.
- Pros: zero WSL setup; Cons: not recommended by the official docs (filesystem performance, terminal integration), advanced features may hit issues.

### References (verified on 2026-07-31)

- OpenCode Web docs (flags, password, mDNS): https://opencode.ai/docs/web/
- OpenCode Windows/WSL docs: https://opencode.ai/docs/windows-wsl
- OpenCode install docs: https://opencode.ai/docs/
- Microsoft Learn: WSL networking (mirrored mode, Hyper-V firewall rule, portproxy): https://learn.microsoft.com/en-us/windows/wsl/networking
- Tailscale Windows install docs: https://tailscale.com/kb/1022/install-windows
- Tailscale + WSL2 mirrored test issue (confirms the Tailscale IP can reach WSL services): https://github.com/tailscale/tailscale/issues/14790
- OpenCode iOS-related issues (PWA improvements in progress): https://github.com/anomalyco/opencode/issues/35480, https://github.com/anomalyco/opencode/issues/10288


---

## 中文完整教學
> 最後更新：2026-07-31 ｜ 適用對象：Windows 11（22H2 以上）＋ iPhone / iPad
> 資料來源：opencode 官方文件、Microsoft Learn、Tailscale 官方文件（文末附連結）

### 0. 先搞懂架構

opencode 是跑在「真正的電腦」上的 agent，iPhone/iPad 不能直接安裝（iOS 沙盒不允許 Node/Bun runtime 與 shell 執行）。所以做法是：

```
iPhone / iPad (Safari/PWA)
   │  ① 同一 Wi-Fi：直接連區域網路
   │  ② 外出：走 Tailscale 加密隧道
   ▼
Windows 機器
   ├─ Tailscale（Windows 端，提供穩定的虛擬 IP）
   └─ WSL2 Ubuntu（mirrored 網路模式，與 Windows 共用 IP）
        └─ opencode web --hostname 0.0.0.0 --port 4096
              └─ 存取專案檔案 → 呼叫 LLM API
```

關鍵技術決策（都有官方文件背書）：
- **為什麼用 WSL**：opencode 官方明言 Windows 上「WSL 體驗最佳」，且 `opencode web` 建議在 WSL 終端執行（[opencode Windows 文件](https://opencode.ai/docs/windows-wsl)）。
- **為什麼用 mirrored 網路模式**：WSL 預設 NAT 模式的 IP 每次重開機都會變；mirrored 模式讓 WSL 與 Windows「共用網路介面」，Tailscale IP 直接就能連到 WSL 裡的服務（[Microsoft 文件](https://learn.microsoft.com/en-us/windows/wsl/networking)）。

---

### 1. 事前準備（檢查清單）

- [ ] Windows 11（22H2 以上；低於此版本請改用「附錄 B」方案）
- [ ] WSL2 + Ubuntu（`wsl --install` 即可安裝，見 2.1）
- [ ] Tailscale 免費帳號（https://tailscale.com 註冊）
- [ ] LLM provider 的 API key（或打算用 opencode Zen 內建模型）
- [ ] iPhone / iPad（iOS 17 以上，PWA 體驗較完整）

---

### 2. 伺服器端設定（在 Windows 上）

#### 2.1 安裝 WSL2 + Ubuntu（若還沒裝）

以**系統管理員**身分開啟 PowerShell，執行：

```powershell
wsl --install
```

- 會自動安裝 WSL2 與 Ubuntu，完成後**重新開機**。
- 重開機後會要求設定 Ubuntu 的使用者名稱與密碼，記好它。
- 確認版本：

```powershell
wsl --set-default-version 2
wsl -l -v
```

應該看到 Ubuntu 的 VERSION 是 2。

#### 2.2 啟用 mirrored 網路模式（重要）

在 Windows 建立檔案 `C:\Users\<你的使用者名稱>\.wslconfig`，內容：

```ini
[wsl2]
networkingMode=mirrored
```

然後在 PowerShell 執行（會關閉所有 WSL）：

```powershell
wsl --shutdown
```

重新開啟 WSL 終端（開始選單 → Ubuntu）。mirrored 模式生效後，WSL 與 Windows 共用同一組網路介面，**Tailscale 的 IP 會直接涵蓋 WSL 內的服務**。

> 如果 `.wslconfig` 沒生效，確認檔案名稱是 `.wslconfig`（有開頭點、沒有副檔名），並放在使用者目錄根目錄。

#### 2.3 允許手機連進 4096 port（Hyper-V 防火牆）

以**系統管理員**身分開啟 PowerShell，執行（這是 Microsoft 官方文件提供的做法）：

```powershell
New-NetFirewallHyperVRule -Name "opencode-web" -DisplayName "OpenCode Web" -Direction Inbound -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -Protocol TCP -LocalPorts 4096
```

#### 2.4 在 WSL 安裝 opencode

開啟 Ubuntu 終端，執行：

```bash
curl -fsSL https://opencode.ai/install | bash
source ~/.bashrc
opencode --version
```

- 安裝腳本會下載預編譯 binary，**不需要另外裝 Node.js**（裝到 `~/.opencode/bin`）。
- 看到版本號（如 1.18.x）即成功。

#### 2.5 設定登入密碼（不做這步＝任何人連上都能用）

編輯 WSL 的設定檔，讓密碼在每次開終端時自動載入：

```bash
echo 'export OPENCODE_SERVER_PASSWORD="請換成你的強密碼"' >> ~/.bashrc
echo 'export OPENCODE_SERVER_USERNAME=opencode' >> ~/.bashrc
source ~/.bashrc
```

- 使用者名稱預設就是 `opencode`，可自行更改。
- 密碼建議 12 字元以上、不要與其他帳號共用。

#### 2.6 啟動 web server

先切到你要讓 opencode 工作的專案目錄（建議放在 WSL 檔案系統內，例如 `~/code/你的專案`；也可以直接讀 Windows 磁碟 `/mnt/c/...`，但速度較慢）：

```bash
cd ~/code/你的專案
opencode web --hostname 0.0.0.0 --port 4096
```

**驗證**：在 Windows 瀏覽器開 `http://localhost:4096`，應看到登入畫面，輸入 `opencode` / 你設的密碼。

> 想讓它在同一 Wi-Fi 下用 `opencode.local` 被找到，可加 `--mdns` 旗標（會自動綁定 0.0.0.0）。

#### 2.7 安裝 Tailscale（外出也能連）

1. 下載安裝 Windows 版 Tailscale：https://tailscale.com/download/windows （或 PowerShell 執行 `winget install Tailscale.Tailscale`）。
2. 登入你的 Tailscale 帳號（會開瀏覽器授權）。
3. 在 PowerShell 查你的 Tailscale IP：

```powershell
tailscale ip -4
```

會得到類似 `100.101.102.103` 的 IP——**這就是手機端永遠連得到的位置**，不管你在哪個網路。

4. （可選）讓 Tailscale 開機自動啟動：Windows 版 Tailscale 預設會隨登入啟動，服務開機即連，無需額外設定。

---

### 3. iOS 端使用

#### 3.1 第一次連線

iPhone/iPad 的 Safari 開啟：

- 同一個 Wi-Fi：`http://<Windows 的區域網路 IP>:4096`（如 `http://192.168.1.5:4096`；不確定 IP 就在 PowerShell 跑 `ipconfig`）
- 外出／手機網路：`http://<Tailscale IP>:4096`（如 `http://100.101.102.103:4096`）

輸入使用者名稱 `opencode` 與密碼，即可開始對話、讓 agent 改程式。

#### 3.2 加入主畫面（變成「App」）

1. Safari 打開後，點「分享」按鈕 →「加入主畫面」。
2. 命名（例如 OpenCode）→「新增」。
3. 之後從主畫面點開，就是以全螢幕 PWA 方式執行，介面更像 App。

#### 3.3 使用上的注意事項

- 這介面適合：打字下指令、檢視 agent 的修改、看 diff、追蹤進度。
- 不適合：需要完整 TUI 鍵盤快捷鍵的進階操作——那種情境請用 Blink Shell（iOS 的 SSH/mosh 客戶端）連進 WSL 操作 `opencode` 終端介面。
- iOS 把 Safari 切到背景後連線會中斷，官方 Web UI 有自動重連邏輯，但**不要背景化太久**；回來後如果畫面卡住，重新整理即可。
- 已知 iOS PWA 有少數 UI 小問題（如安全區域重疊），屬官方進行中的 issue，不影響主要功能。

---

### 4. 安全注意事項

- `--hostname 0.0.0.0` 且未設密碼 = **區域網路內任何人**都能操作你的 agent，密碼務必設定。
- Tailscale 預設 ACL 只允許「你自己的裝置」互連，不要隨便分享節點。
- 不要為 4096 port 做路由器 port forwarding 到公網；真的需要公網存取時，改用 Cloudflare Tunnel 或 Tailscale Serve。
- 密碼會以明文存在 WSL 的 `~/.bashrc`，這是個人機可接受的做法；若共用電腦請改用 systemd service + 權限限制（附錄 A）。

---

### 5. 疑難排解

| 症狀 | 原因 / 解法 |
|---|---|
| Windows 瀏覽器打得開，手機打不開 | ① Hyper-V 防火牆規則沒設（2.3）② 手機與電腦不在同一網路（改用 Tailscale IP）③ Tailscale 沒登入 |
| 手機連上了但顯示不安全連線 | 用的是 http:// 而非 https://，正常；Safari 對區域網路 IP 允許，若被擋請改用 Tailscale IP |
| 忘了密碼 | 編輯 `~/.bashrc` 重新 export 新密碼，重啟 `opencode web` |
| `opencode.local` 找不到 | mDNS 在部分網路（如公共 Wi-Fi）會被擋，改用 IP 連線 |
| 重開機後連不上 | WSL 的 opencode 不會自動啟動（見附錄 A）；Tailscale IP 不變，直接重開 `opencode web` 即可 |
| 手機 PWA 畫面錯位 | iOS PWA 已知小問題（官方 issue [#35480](https://github.com/anomalyco/opencode/issues/35480) 等），用 Safari 直接開可暫時避開 |
| 想用完整終端 TUI | 在 iOS 裝 Blink Shell，SSH 進 WSL（`ssh 使用者@Windows-IP`）後跑 `opencode` |

---

### 附錄 A：讓 opencode web 開機自動啟動（可選）

WSL 的 Ubuntu 支援 systemd（`wsl.conf` 內 `[boot] systemd=true` 需先啟用），之後可建立 service：

```bash
sudo tee /etc/systemd/system/opencode-web.service > /dev/null <<'EOF'
[Unit]
Description=OpenCode Web Server
After=network-online.target

[Service]
Environment=OPENCODE_SERVER_USERNAME=opencode
Environment=OPENCODE_SERVER_PASSWORD=你的強密碼
WorkingDirectory=/home/你的使用者/code/你的專案
ExecStart=/home/你的使用者/.opencode/bin/opencode web --hostname 0.0.0.0 --port 4096
Restart=always
User=你的使用者

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now opencode-web
```

> 密碼寫在 service 檔內，權限請設為僅自己可讀（`sudo chmod 600`）。

### 附錄 B：Windows 10／無法用 mirrored 模式時的替代方案（NAT + portproxy）

1. 略過 2.2 的 `.wslconfig`（NAT 模式）。
2. 在 WSL 查 WSL 自己的 IP：`hostname -I`（例如 `172.20.10.5`）。
3. 管理員 PowerShell 建立 port 轉發：

```powershell
netsh interface portproxy add v4tov4 listenport=4096 listenaddress=0.0.0.0 connectport=4096 connectaddress=172.20.10.5
```

4. 管理員 PowerShell 允許 Windows 防火牆 inbound 4096：

```powershell
New-NetFirewallRule -DisplayName "OpenCode Web" -Direction Inbound -Protocol TCP -LocalPort 4096 -Action Allow
```

5. Tailscale 裝在 Windows 上即可，portproxy 會把流量轉進 WSL。
6. **注意**：WSL 的 NAT IP 每次重開機可能改變，重開後要重新執行第 3 步。

### 附錄 C：不裝 WSL 的「最快」替代（原生 Windows）

opencode 其實可以直接在 Windows 跑（官方文件承認可行，但建議 WSL）：

```powershell
npm install -g opencode-ai   # 需要先裝 Node.js（建議 20+ 的 LTS）
opencode web --hostname 0.0.0.0 --port 4096
```

- 再補一條 Windows Defender 防火牆 inbound 4096 規則，並照 2.7 裝 Tailscale 即可。
- 優點：零 WSL 設定；缺點：官方不推薦（檔案系統效能、終端整合較差），進階功能可能踩雷。

---

### 參考資料（查證時間：2026-07-31）

- opencode Web 文件（`opencode web` 旗標、密碼、mDNS）：https://opencode.ai/docs/web/
- opencode Windows/WSL 文件：https://opencode.ai/docs/windows-wsl
- opencode 安裝文件：https://opencode.ai/docs/
- Microsoft Learn：WSL 網路（mirrored 模式、Hyper-V 防火牆規則、portproxy）：https://learn.microsoft.com/en-us/windows/wsl/networking
- Tailscale Windows 安裝文件：https://tailscale.com/kb/1022/install-windows
- Tailscale + WSL2 mirrored 實測 issue（證實 Tailscale IP 可連 WSL 服務）：https://github.com/tailscale/tailscale/issues/14790
- opencode iOS 相關 issue（PWA 持續改善中）：https://github.com/anomalyco/opencode/issues/35480 、 https://github.com/anomalyco/opencode/issues/10288




