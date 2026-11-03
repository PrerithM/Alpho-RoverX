# RoverMania

![RoverMania Hero](./public/Rover%20HeroPage.png)

[![Next.js 16](https://img.shields.io/badge/Next.js-16.2.4-black?logo=next.js)](https://nextjs.org/)
[![React 19](https://img.shields.io/badge/React-19.2.4-61DAFB?logo=react)](https://react.dev/)
[![Tailwind CSS v4](https://img.shields.io/badge/Tailwind_CSS-v4-38B2AC?logo=tailwind-css)](https://tailwindcss.com/)
[![WebRTC WHEP](https://img.shields.io/badge/Streaming-WebRTC_WHEP-FF6F00)](https://github.com/bluenviron/mediamtx)
[![Gemini 2.5 Flash](https://img.shields.io/badge/AI_Vision-Gemini_2.5_Flash-8E75C2?logo=google-gemini)](https://aistudio.google.com/)
[![Web Audio API](https://img.shields.io/badge/Audio-Web_Audio_API_Synthesizer-4CAF50)](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-3178C6?logo=typescript)](https://www.typescriptlang.org/)

RoverMania is a real-time, aerospace-inspired rover telemetry and command console. Engineered with **Next.js 16**, **React 19**, and **Tailwind CSS v4**, it couples sub-second **WebRTC (WHEP)** video streaming and dual-channel **WebSocket** motor control with a procedural **Web Audio API sound engine** and multimodal **Google Gemini 2.5 Flash** vision analysis.

---

## Key Features

- **Aerospace Tactical Cockpit**: Heavyweight brushed-aluminum styling with a 240-degree rotary throttle dial, tactile mechanical buttons, spatial 3D D-pad, and live telemetry indicators (latency, battery, and signal status).
- **Procedural Cockpit Synthesizer**: Zero external audio downloads. A custom Web Audio API engine models CNC aluminum resonance, mechanical contact transients, analog high-shelf warmth, and emergency klaxons directly in the browser.
- **Ultra-Low Latency WHEP Video**: Direct peer-to-peer WebRTC video stream through MediaMTX over standard WHEP HTTP POST signaling for glass-to-glass latency under 200ms.
- **Multimodal AI Reconnaissance**: Edge-rendered Gemini 2.5 Flash API endpoint captures live video frames on command to detect obstacles, terrain types, and visual landmarks.
- **Dynamic In-App Network Switcher**: Real-time IP switching modal with quick presets (Lab Default, Local AP, Home Network, and Local Simulator) stored in `localStorage`—no codebase edits needed to change target rovers.
- **Session Security & Operator Gate**: Password-protected operator sessions backed by Edge middleware and HTTP-only cookie guards.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Operator Ground Station                     │
│               Next.js 16 Cockpit (Port 3000)                │
│                                                             │
│   ┌───────────────────┐  ┌──────────────────────────────┐   │
│   │   Tactile HUD     │  │      Procedural Web Audio    │   │
│   │ D-Pad, Throttle,  │  │ Resonant Clicks, Relays,     │   │
│   │ Telemetry Gauge   │  │ Sirens (Zero External Audio) │   │
│   └────────┬──────────┘  └──────────────────────────────┘   │
└────────────┼──────────────────────────────┬─────────────────┘
             │                              │
     WebSocket (Port 8765)           WebRTC WHEP (Port 8889)
     Motor Control Packets          H.264 Video Stream (<200ms)
     JSON { L: float, R: float }            │
             │                              │
             ▼                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Raspberry Pi (Rover Core)                   │
│                                                             │
│   ┌─────────────────────────┐  ┌────────────────────────┐   │
│   │  Python WebSocket Host  │  │  MediaMTX Media Server │   │
│   │   (RPi.GPIO / Motors)   │  │  (Pi Camera Source)    │   │
│   └─────────────────────────┘  └────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                               ▲
                               │ Frame Snapshot (Base64)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               Google Gemini 2.5 Flash API                   │
│          Edge Multimodal Frame & Terrain Analysis           │
└─────────────────────────────────────────────────────────────┘
```

---

## Hardware Requirements

| Component | Recommendation | Function |
|---|---|---|
| **SBC** | Raspberry Pi 4 Model B (or Pi 5 / Pi 3B+) | Primary on-board computer |
| **Camera** | Raspberry Pi Camera Module (v2, v3, or HQ) | Low-latency 720p/30fps video capture |
| **Motor Driver** | L298N, TB6612FNG, or L9110S dual H-Bridge | DC motor power switching & PWM direction |
| **Chassis & Motors** | 2WD or 4WD robotic chassis with geared DC motors | Locomotion |
| **Power Source** | Independent 5V 3A USB-C battery pack (Pi) + 7.4V/11.1V Li-Po (Motors) | Clean, isolated power without brownouts |

---

## Part 1 — Raspberry Pi Setup

### 1. Network & Camera Initialization

Ensure the Raspberry Pi connects to the same local Wi-Fi network as your operator workstation, then note its local IP:

```bash
hostname -I
```

Enable camera support via configuration:

```bash
sudo raspi-config
```
Navigate to **Interface Options** -> **Camera**, enable it, and reboot. Verify detection:

```bash
libcamera-hello --list-cameras
```

### 2. MediaMTX (WebRTC / WHEP Server)

MediaMTX serves WebRTC streams directly over WHEP:

```bash
# Download ARM64 binary (for 64-bit Pi OS)
wget https://github.com/bluenviron/mediamtx/releases/latest/download/mediamtx_linux_arm64v8.tar.gz

tar -xzf mediamtx_linux_arm64v8.tar.gz
sudo mv mediamtx /usr/local/bin/
sudo mv mediamtx.yml /usr/local/etc/
```

Edit `/usr/local/etc/mediamtx.yml` to define the camera path:

```yaml
paths:
  cam:
    source: rpiCamera
    rpiCameraWidth: 1280
    rpiCameraHeight: 720
    rpiCameraFPS: 30
```

Configure MediaMTX to run continuously as a systemd service:

```bash
sudo nano /etc/systemd/system/mediamtx.service
```

```ini
[Unit]
Description=MediaMTX WHEP WebRTC Streamer
After=network.target

[Service]
ExecStart=/usr/local/bin/mediamtx /usr/local/etc/mediamtx.yml
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mediamtx
```

> [!TIP]
> You can test the stream directly by visiting `http://<PI_IP>:8889/cam/` in any desktop browser.

### 3. Motor Controller WebSocket Daemon

Install required Python modules on the Pi:

```bash
pip install websockets RPi.GPIO
```

Create `/home/pi/rover_ws.py`:

```python
import asyncio
import json
import websockets
import RPi.GPIO as GPIO

# BCM pin definitions (adjust to your wiring)
LEFT_FORWARD  = 17
LEFT_BACKWARD = 18
RIGHT_FORWARD = 22
RIGHT_BACKWARD = 23

GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)
for pin in [LEFT_FORWARD, LEFT_BACKWARD, RIGHT_FORWARD, RIGHT_BACKWARD]:
    GPIO.setup(pin, GPIO.OUT)
    GPIO.output(pin, GPIO.LOW)

def drive(left: float, right: float):
    GPIO.output(LEFT_FORWARD,   GPIO.HIGH if left  > 0.1 else GPIO.LOW)
    GPIO.output(LEFT_BACKWARD,  GPIO.HIGH if left  < -0.1 else GPIO.LOW)
    GPIO.output(RIGHT_FORWARD,  GPIO.HIGH if right > 0.1 else GPIO.LOW)
    GPIO.output(RIGHT_BACKWARD, GPIO.HIGH if right < -0.1 else GPIO.LOW)

async def handler(websocket):
    print("Telemetry link established with cockpit.")
    try:
        async for message in websocket:
            data = json.loads(message)
            drive(float(data.get("L", 0)), float(data.get("R", 0)))
    except websockets.ConnectionClosed:
        pass
    finally:
        drive(0, 0)
        print("Telemetry link dropped. Motors halted.")

async def main():
    async with websockets.serve(handler, "0.0.0.0", 8765):
        print("WebSocket motor server active on port 8765.")
        await asyncio.Future()

if __name__ == "__main__":
    asyncio.run(main())
```

Register and start the motor control service:

```bash
sudo nano /etc/systemd/system/rover-ws.service
```

```ini
[Unit]
Description=Rover WebSocket Motor Controller
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/rover_ws.py
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rover-ws
```

---

## Part 2 — Web Application Setup

### 1. Clone & Install Dependencies

```bash
git clone https://github.com/PrerithM/Rover-Mania.git
cd Rover-Mania
npm install
```

### 2. Configure Environment Variables

Create `.env.local` in the project root:

```env
GEMINI_API_KEY=your_google_gemini_api_key_here
```

> [!NOTE]
> Obtain a Gemini API key at [Google AI Studio](https://aistudio.google.com/app/apikey). The application utilizes the **Gemini 2.5 Flash** model for sub-second multimodal frame inspection.

### 3. Run the Development Server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

To produce an optimized production bundle:

```bash
npm run build
npm run start
```

---

## Cockpit Operations Manual

### Authentication & Boot Sequence

1. Accessing `/dashboard` redirects unauthenticated sessions to `/login`.
2. Enter the default operator passphrase: `PrerithRover`.
3. The landing screen initializes a 3-second system self-test verifying operator handle, audio context, and telemetry endpoints.
4. Click **Quick Connect** or configure custom network targets via the **Rover Network Settings** gear icon.

> [!IMPORTANT]
> Change the default password in `src/app/api/auth/route.ts` prior to exposing your cockpit to shared or public networks.

### Drive Controls & Actuation

| Input | Actuation | Command Payload |
|---|---|---|
| `W` / `Arrow Up` | Forward Drive | `{ L: 1.0 * speed, R: 1.0 * speed }` |
| `S` / `Arrow Down` | Reverse Drive | `{ L: -1.0 * speed, R: -1.0 * speed }` |
| `A` / `Arrow Left` | Pivot Left | `{ L: -0.85 * speed, R: 0.85 * speed }` |
| `D` / `Arrow Right` | Pivot Right | `{ L: 0.85 * speed, R: -0.85 * speed }` |
| Key Release | Safe Neutral | `{ L: 0.0, R: 0.0 }` |
| `Space` / **E-STOP** | Emergency Stop | Immediate hard stop + audio klaxon |
| **Rotary Throttle Dial** | Speed Multiplier | Scales output from `0.1x` (10%) to `1.0x` (100%) |

### Dynamic Target IP Switching

You do not need to modify code to switch networks or test with multiple rovers:

1. Click the **Network / IP Settings** icon in the TopBar or Landing screen.
2. Select a preconfigured preset (**Lab Default**, **Local AP**, **Home Network**, or **Simulator**) or input a custom IPv4 address.
3. Save changes—both WebSocket motor telemetry and WHEP video feeds reconnect instantaneously and persist in your browser's local storage.

### Multimodal AI Vision Inspection

1. Navigate to the **Rover Vision / Tactical Analysis** panel on the right side of the cockpit.
2. Click any of the predefined reconnaissance queries (*"Scan for obstacles"*, *"Analyze terrain stability"*, *"Identify key objects"*) or type a custom prompt.
3. The client captures the active video frame from the `<canvas>` buffer and relays it to the `/api/vision` Edge handler for Gemini multimodal evaluation.

---

## Project Structure

```
Rover-Mania/
├── public/
│   ├── Rover HeroPage.png       # Cockpit identity & hero banner
│   └── textures/                # Brushed-metal tactile surface textures
├── src/
│   ├── app/
│   │   ├── api/
│   │   │   ├── auth/route.ts    # Session token management & password verification
│   │   │   └── vision/route.ts  # Gemini 2.5 Flash Edge vision analysis endpoint
│   │   ├── dashboard/           # Tactical HUD cockpit interface
│   │   ├── login/               # Password gate page
│   │   ├── globals.css          # Tailwind CSS v4 design tokens
│   │   ├── layout.tsx           # Application layout shell
│   │   └── page.tsx             # Diagnostic boot sequence & quick-connect launch
│   ├── components/
│   │   ├── AiAnalysisPanel.tsx  # Interactive Gemini AI assistant panel
│   │   ├── ControlPanel.tsx     # Connection metrics & Emergency Stop actuator
│   │   ├── DrivePanel.tsx       # 240° rotary throttle dial & pedal controls
│   │   ├── IpSettingsModal.tsx  # Dynamic rover target IP switcher & presets
│   │   ├── SpatialDPad.tsx      # 3D spatial directional pad with touch/keyboard states
│   │   ├── TopBar.tsx           # Real-time telemetry status bar (ping, signal, battery)
│   │   └── VisorFeed.tsx        # Low-latency WebRTC WHEP viewport & frame capture
│   ├── hooks/
│   │   ├── useRoverWebRTC.ts    # Direct WHEP WebRTC connection hook
│   │   └── useRoverWebSocket.ts # Motor control WebSocket client with reconnect logic
│   ├── utils/
│   │   └── cockpitAudio.ts      # Pure Web Audio API synthesized acoustics engine
│   └── proxy.ts                 # Next.js authentication middleware guard
├── next.config.ts
├── package.json
└── tsconfig.json
```

---

## Remote Access via Cloudflare Tunnel

To operate your rover over the internet without exposing raw local ports:

```bash
# Install cloudflared on the Raspberry Pi
sudo apt-get install cloudflared

# Route your WebRTC & WebSocket endpoints securely
cloudflared tunnel --url http://localhost:8889
cloudflared tunnel --url http://localhost:8765
```

Update your cockpit's **Target IP / Host** in the settings modal to use your secure tunnel hostname.
