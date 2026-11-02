# EA SPORTS UFC 3 PS4 Port: Complete Setup & Multiplayer Guide (Linux & Windows)

A comprehensive guide for setting up and configuring the open-source **EA SPORTS UFC 3 native PS4 PC port** ([GNSB/UFC-3-PS4-PORT-LINUX-WINDOWS](https://github.com/GNSB/UFC-3-PS4-PORT-LINUX-WINDOWS)), enabling both **Native Peer-to-Peer Online Netplay** and **Streamed Couch PvP (Sunshine + Moonlight)** over Tailscale, along with troubleshooting steps for real-world issues encountered along the way.

---

## Table of Contents
1. [Overview & Architecture](#1-overview--architecture)
2. [Prerequisites & File Extraction](#2-prerequisites--file-extraction)
3. [Runtime Installation & Configuration](#3-runtime-installation--configuration)
4. [Multiplayer Mode 1: Native Peer-to-Peer Netplay](#4-multiplayer-mode-1-native-peer-to-peer-netplay)
5. [Multiplayer Mode 2: Streamed Couch PvP (Sunshine + Moonlight)](#5-multiplayer-mode-2-streamed-couch-pvp-sunshine--moonlight)
6. [Real-World Pitfalls & Troubleshooting](#6-real-world-pitfalls--troubleshooting)
   - [Issue 1: "EA Servers are Offline" / Firewall Block](#issue-1-ea-servers-are-offline--firewall-block)
   - [Issue 2: Server Stops When Host Closes Game](#issue-2-server-stops-when-host-closes-game)
   - [Issue 3: Tailscale One-Way Node Sharing](#issue-3-tailscale-one-way-node-sharing)
   - [Issue 4: Both Players Moving the Same Joystick (Shared Controller Bug)](#issue-4-both-players-moving-the-same-joystick-shared-controller-bug)
   - [Issue 5: Sunshine Linux Security Advisory (GHSA-fp6g-27w5-489j)](#issue-5-sunshine-linux-security-advisory-ghsa-fp6g-27w5-489j)
7. [Quick Reference Commands](#7-quick-reference-commands)

---

## 1. Overview & Architecture

Unlike traditional console emulators that emulate hardware components and virtual machines, this project operates via a **native binary loader approach**:
* **Direct x86-64 Execution:** Because the PS4 CPU is built on x86-64 architecture, the game's executable (`eboot.bin`) is loaded and executed directly by modern PC processors without CPU dynamic recompilation overhead.
* **Vulkan Graphics Layer:** Translates PS4 low-level graphics commands (GNM/GNMX) into Vulkan using a custom renderer derived from **shadPS4**.
* **System Translation & OS Hooks:** Replaces Sony system libraries (`libkernel`, `libScePad`, `libSceAudioOut`, `libSceNet`) with native host implementations (SDL3 for windowing/input, native sockets for networking).

---

## 2. Prerequisites & File Extraction

### 2.1 Required Game Files
The port runtime **does not provide copyrighted game data**. You must supply your own decrypted PS4 dumps:
* **Base Game:** PlayStation 4 Title ID `CUSA06534` (Base game, ~18.9 GB)
* **Game Update:** Update **v1.14** (~4.4 GB)

> **Important:** The port cannot read raw `.pkg` archives directly. The packages must be extracted into a decrypted filesystem structure.

### 2.2 Package Unpacking
You can extract PlayStation 4 `.pkg` packages on Linux using utilities such as `pkgextract`:

```bash
# Extract the base game package
pkgextract -o ./game/CUSA06534 /path/to/EA.Sports.UFC.3-CUSA06534-Game.pkg

# Extract the update package
pkgextract -o ./game/CUSA06534-patch /path/to/EA.Sports.UFC.3-CUSA06534-Update-v1.14.pkg
```

### 2.3 Merging Update 1.14 (Zero Disk Waste)
The project provides a utility script (`scripts/merge_update.py`) that merges the update onto the base game using filesystem **hard links**, resulting in zero extra disk usage:

```bash
python3 runtime/UFC3Native/scripts/merge_update.py ./game/CUSA06534
```
This produces `game/CUSA06534-merged` with updated files overlaid automatically.

---

## 3. Runtime Installation & Configuration

### 3.1 Obtaining the Release
Download the latest release tarball or zip from the project releases:
* **Linux:** `UFC3Native-<version>-linux-x86_64.tar.gz`
* **Windows:** `UFC3Native-<version>-windows-x86_64.zip`

Extract the archive:
```bash
tar -xzf UFC3Native-*-linux-x86_64.tar.gz -C runtime/
```

### 3.2 Verifying Integrity
You can verify symbols and relocations using the preparation tool:
```bash
python3 runtime/UFC3Native/scripts/prepare.py ./game/CUSA06534-merged
```
Expected output confirms symbol imports, relocations, module count, and entrypoints.

### 3.3 Convenient Wrapper Scripts
Create convenience scripts in your working directory:

**`start_launcher.sh` (Opens the GTK GUI control panel):**
```bash
#!/usr/bin/env bash
cd "$(dirname "$0")/runtime/UFC3Native"
exec ./ufc3.sh "$@"
```

**`start_game.sh` (Direct launch with saved settings):**
```bash
#!/usr/bin/env bash
cd "$(dirname "$0")/runtime/UFC3Native"
exec ./ufc3.sh --play "$@"
```

---

## 4. Multiplayer Mode 1: Native Peer-to-Peer Netplay

Because official EA/PSN servers cannot authenticate custom binaries, the port bundles a custom Python server (`tools/ufc3_server.py`) that emulates the EA Blaze protocol, authentication, and matchmaking.

### 4.1 Networking Requirements
* Both players must run the **same port build** and the **same update version (1.14)**.
* Cross-play between Linux and Windows is supported.
* For playing over the internet across different networks, both players connect through **Tailscale**.

### 4.2 Hosting a Match
1. Host opens the launcher, selects **Host server**.
2. Selects their **Tailscale IPv4 address (`100.x.y.z`)** in the Host address field. *(Never select `127.0.0.1`)*.
3. Host shares their Tailscale IP with the friend and clicks **Play**.

### 4.3 Joining a Match
1. Client (Friend) opens their launcher, selects **Join server**.
2. Enters the Host's Tailscale IP (`100.x.y.z`) in the Join address field.
3. Clicks **Play**.

### 4.4 In-Game Matchmaking
* Both players navigate to **Online → Quick Fight** in the in-game menu.
* Both click **Search** at approximately the same time.
* The local server intercepts the requests and pairs both players into the match.
* **Rematches:** Pressing Rematch on the post-fight results screen works out of the box.

---

## 5. Multiplayer Mode 2: Streamed Couch PvP (Sunshine + Moonlight)

If your friend does not have the game installed or wants zero setup on their end, you can stream the game using **Sunshine** (Host) and **Moonlight** (Client). The friend's controller is recognized as an independent local player.

### 5.1 Host Configuration (Sunshine on Linux)
1. Ensure hardware encoders are detected (e.g. `VAAPI` on AMD or `NVENC` on NVIDIA).
2. Ensure `/dev/uinput` permissions allow virtual controller creation.
3. Add the game to `~/.config/sunshine/apps.json`:
   ```json
   {
     "name": "EA SPORTS UFC 3",
     "detached": [
       "/absolute/path/to/start_game.sh"
     ],
     "image-path": "/absolute/path/to/icon0.png"
   }
   ```
4. Restart the Sunshine service:
   ```bash
   systemctl --user restart app-dev.lizardbyte.app.Sunshine.service
   ```

### 5.2 Client Configuration (Moonlight on Windows/Linux)
1. Install Moonlight on the client PC.
2. Click **Add Host (+)** and enter the Host's Tailscale IP (`100.x.y.z`).
3. Note the 4-digit pairing PIN shown on the client screen.
4. On the host, open `https://localhost:47990` in a browser, navigate to the **PIN** tab, enter the code, and submit.
5. In Moonlight, launch **EA SPORTS UFC 3**.

### 5.3 Playing Local 1v1
* In UFC 3, go to **Fight Now**.
* Host's physical controller moves to the **Red Corner** (Player 1).
* Friend's controller (passed through Moonlight) moves to the **Blue Corner** (Player 2).

---

## 6. Real-World Pitfalls & Troubleshooting

### Issue 1: "EA Servers are Offline" / Firewall Block
* **Symptom:** The client gets an in-game error saying *"EA servers are offline"* when attempting to connect to Quick Fight.
* **Root Cause:** On Linux hosts, `ufw` (Uncomplicated Firewall) blocks incoming connections on the `tailscale0` network interface by default.
* **Solution:** Allow incoming traffic on the Tailscale interface on the host:
  ```bash
  sudo ufw allow in on tailscale0
  ```

---

### Issue 2: Server Stops When Host Closes Game
* **Symptom:** If the host exits UFC 3 or restarts the game, the client immediately loses connection and cannot rejoin.
* **Root Cause:** When using the launcher's built-in "Host server" mode, the Python server process is tied to the game lifecycle; exiting the game terminates the server.
* **Solution:** Run a dedicated standalone server in an independent terminal:
  ```bash
  python3 runtime/UFC3Native/tools/ufc3_server.py --bind 0.0.0.0
  ```
  Both players then select **Join server** in their respective launchers, allowing either player to restart their game without disrupting the server.

---

### Issue 3: Tailscale One-Way Node Sharing
* **Symptom:** Friend runs `Test-NetConnection -ComputerName <HOST_IP> -Port 40443` in PowerShell and receives:
  `Ping to <HOST_IP> failed with status: TimedOut`
  `TcpTestSucceeded : False`
  even though the host could ping the friend.
* **Root Cause:** In Tailscale, machine sharing between two distinct accounts is **strictly one-way**. If the friend shared their node with the host, the host can reach the friend, but the friend cannot reach the host.
* **Solution:**
  1. Host logs into the [Tailscale Admin Console](https://login.tailscale.com/admin/machines).
  2. Finds their PC node → clicks **`...`** → selects **Share...**.
  3. Sends an invite link to the friend.
  4. Once the friend accepts, bidirectional traffic is established, and `TcpTestSucceeded` returns `True`.

---

### Issue 4: Both Players Moving the Same Joystick (Shared Controller Bug)
* **Symptom:** During Sunshine + Moonlight couch PvP, moving the friend's controller also moves Player 1 instead of Player 2.
* **Root Cause:** The game was launched while `"mode": "join"` or `"mode": "host"` was set in `launcher.json` (which sets `BB_ONLINE_SERVER`). The port runtime explicitly disables multi-controller mapping when `BB_ONLINE_SERVER` is set, forcing all gamepads to control Player 1 for netplay.
* **Solution:**
  1. In the launcher, set **Online play** to **Offline** (or edit `launcher.json` to `"mode": "offline"`).
  2. Ensure `local_multiplayer=1` is present in `bbport.ini`.
  3. Alternatively, open the in-game overlay menu (**Insert** or **L3 + R3**), scroll to **Controllers (local multiplayer)**, check **One player per controller**, and explicitly assign:
     - **Port 1:** Host's physical gamepad
     - **Port 2:** Moonlight virtual controller (`Sunshine (libvirtualhid) X-Box Series Controller`)

---

### Issue 5: Sunshine Linux Security Advisory (GHSA-fp6g-27w5-489j)
* **Symptom:** Running Sunshine on Linux versions `<= v2026.906.222525` is vulnerable to a high-severity privilege escalation vulnerability.
* **Root Cause:** Untrusted GUI modules could execute with Sunshine's elevated Linux capabilities.
* **Solution:** Update Sunshine to version `v2026.914.233613` or newer (e.g. via `yay -S sunshine-bin`). Initialization logs will confirm capability dropping:
  `[misc] drop_elevated_privileges succeeded in dropping capabilities`

---

## 7. Quick Reference Commands

| Action | Command |
|---|---|
| Allow Tailscale through Linux Firewall | `sudo ufw allow in on tailscale0` |
| Check Tailscale IPv4 Address | `tailscale ip -4` |
| Test Connectivity from Client (PowerShell) | `Test-NetConnection -ComputerName <HOST_IP> -Port 40443` |
| Run Standalone Multiplayer Server | `python3 runtime/UFC3Native/tools/ufc3_server.py --bind 0.0.0.0` |
| Restart Sunshine Stream Service | `systemctl --user restart app-dev.lizardbyte.app.Sunshine.service` |
| Open In-Game Controls & Settings Overlay | Press **Insert** (Keyboard) or **L3 + R3** (Gamepad) |

---

## Disclaimer
This project is an independent community port and is not affiliated with, endorsed by, or associated with Electronic Arts, Sony Interactive Entertainment, or the UFC. No copyrighted assets or binaries are distributed.
