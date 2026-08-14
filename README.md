## 📖 Background: Why and how this project was created

It was just an ordinary day: I got home, sat down to work—facing some tight deadlines—and fired up my IDE, only to see the message: "Gemini is not available in your region!" I started looking for the cause, wondering if VPN detection had been tightened (I’d previously been using HAP with VLESS). I cleared my cookies, but that didn't help; there was no news about a block, so the issue seemed local. I decided to try setting up a proxy in my browser. I had prior experience setting up public SOCKS proxies on my VPS, but this time I opted to create a local one instead—quick and clean. That did the trick—Gemini was flying again! YouTube works perfectly without a VPN now, too. I hope this repository helps you out!

# ssh-socks5-bat: Bypass Any Site Location Restrictions
A simple guide to setting up a local proxy tunnel through your remote server [socks5].

### Why is a local proxy better than a simple VPN?
- **Code Security:** No code leaks or traffic sniffing.
- **Clean IP:** Google will **NOT** ban your account!
- **Isolation:** The proxy works exclusively inside your browser.
- **Free (if you have a VPS):** You can set this up in under 5 minutes!

---

## 🛠️ Install Guide

### Step 1. Generating SSH keys on Windows
This is necessary to launch the tunnel with a single click, without constantly entering your server password.

1. Open CMD.
2. Run the command:
```cmd
ssh-keygen -t ed25519
```
> ⚠️ **Important:** When prompted for a passphrase, simply press **Enter three times** to leave it blank. If you set a password here, the auto-start script won't work silently!

### Step 2. Copy the key to the server using PowerShell
Since Windows lacks the native `ssh-copy-id` utility, we use a built-in command.

1. **In the same console**, type `powershell` and press Enter.
2. **[Crucial for new servers]** If you have a brand-new VPS, the `.ssh` folder might not exist yet. Create it first by running this command (replace `your_ip` with your server's IP):
```powershell
ssh root@your_ip "mkdir -p .ssh"
```
*(Enter your server password when prompted).*
3. Now, copy your key to the server. Run the following command strictly as **a single line**:
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@your_ip "cat >> .ssh/authorized_keys"
```
4. Enter your server password one last time. Your laptop and VPS are now securely linked.

### Step 3. Create an auto-start `.bat` file
1. Create a standard text document on your desktop.
2. Paste the following code into it:
```bat
@echo off
title SSH Proxy Tunnel
echo [OK] Proxy tunnel starting...

:loop
echo [INFO] Connecting to VPS...
ssh -D 1080 -N -q root@YOUR_SERVER_IP
echo [WARN] Connection lost. Reconnecting in 5 seconds...
timeout /t 5 >nul
goto loop
```
*(Don't forget to replace `YOUR_SERVER_IP`!)*
3. Save the file as **`proxy.bat`** (Make sure to select *"All files (*.*)"* in the "Save as type" dropdown, not just .txt).

---

## 🌐 Browser Configuration (Brave / Chrome / Yandex)

1. Install the **SmartProxy** or **FoxyProxy** extension.
2. Add a new proxy server with the following settings:
   - **Protocol type:** SOCKS5
   - **Address (Host):** `127.0.0.1` (this is your localhost)
   - **Port:** `1080`
   - Leave the username and password fields **blank**.
3. Select the **"Always On"** mode in the extension (or configure specific rules for domains like `*://*.google.com/*`).

---

## ⚠️ Crucial: If sites won't load or get stuck loading indefinitely

Google often attempts to run heavy Gemini scripts via the **IPv6** protocol, which can cause the tunnel to throw `connect failed` errors.

**How to fix it:**
If your VPS doesn't support IPv6, I recommend adding the `-4` flag to the SSH command in your `.bat` file (this forces IPv4):
```bat
ssh -D 1080 -N -q -4 root@YOUR_SERVER_IP
```
If your server *does* support IPv6, then everything is perfect as is! 

However, if you didn't add the flag, your server lacks IPv6, and you can't be bothered to do anything else, there is a last-resort—albeit "dirty"—option: simply disable IPv6 in your Windows network settings:
1. Press **`Win + R`**, type **`ncpa.cpl`**, and press Enter.
2. Right-click your active internet connection -> **Properties**.
3. Find **"IP version 6 (TCP/IPv6)"** in the list and **UNCHECK the box** next to it. Click OK. 
4. In the Windows command prompt, flush the route cache using the command: `ipconfig /flushdns`.

---

## 🚀 Daily usage
1. Double-click the `proxy.bat` file on your desktop (a black window will open; do not close it while working).
2. Enable your proxy extension in the browser.
3. Open an **Incognito** window, clear your cookies, and work at maximum speed!

---

## 🔒 Deep Technical Analysis and Cybersecurity

For professional use, it is important to understand the solution's internal architecture to avoid network loops and data leaks.

### 1. Host Server (VPS) Security
- **Port Isolation:** This method (Dynamic Port Forwarding) **does not open** additional ports to the public internet on your VPS. The SOCKS5 server runs exclusively on your laptop's local interface (`127.0.0.1:1080`). External connections to your proxy are impossible.
- **Impact on Running Websites (Nginx/Apache):** Proxy traffic is encapsulated within the standard `sshd` daemon. It does not interact with your website's web server or databases (MySQL/PostgreSQL), as it operates within an isolated kernel context. The risk of compromising site data is zero.
- **Encryption:** All traffic between the client and server is encrypted using the algorithm specified during key generation (e.g., `Ed25519` + `ChaCha20-Poly1305` or `AES-GCM`). Your home ISP sees only standard SSH metadata.

### 2. Niche Exceptions and Network Vulnerabilities (Caveats)

#### A. WebRTC Leak
Even with a SOCKS5 proxy enabled, modern browsers (especially Chromium-based ones) may leak your real ISP IP address via the STUN/ICE protocol (used for in-browser video calls). Google actively uses WebRTC metadata to detect your actual region.
- *Solution:* In the SmartProxy extension or via browser settings (e.g., in Brave: `Settings -> Privacy and security -> WebRTC IP Handling Policy`), strictly set the mode to **`Disable non-proxied UDP`**. 

#### B. DNS Leaks
If your proxy extension is misconfigured, the browser might send DNS requests via your home ISP while routing actual web traffic through the VPS. This causes a location mismatch, resulting in a regional block.
- *Solution:* Ensure the **"Proxy DNS"** option (Resolve DNS via proxy server) is enabled in your proxy extension settings.

#### C. "Dead Channel" Issue (TCP KeepAlive)
If you leave the batch file running but step away from your computer for half an hour, your home router might terminate the inactive TCP session. The batch file window will remain open, but your browser will lose internet access (and `channel_open: failed` errors will appear).
- *Solution:* To prevent the tunnel from dropping, ensure the following parameters are uncommented in the `/etc/ssh/sshd_config` file on your VPS:
```sshd
ClientAliveInterval 60
ClientAliveCountMax 3
```
This forces the server to check if your laptop is still active once a minute, keeping the session stable.

#### D. Zombie Processes on Windows
If you force-close the batch file (e.g., via Task Manager or during a PC crash), port `1080` on Windows may remain "occupied" by a hung `ssh.exe` process. Attempting to run the batch file again will result in a `bind:1080: Address already in use` error.
- *Solution:* Open `cmd` on Windows and terminate the hung process using the command: 
```cmd
taskkill /f /im ssh.exe
```
