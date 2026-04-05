<p align="right">
  <a href="readme_jp.md">日本語版はこちら</a>
</p>

## Introduction
Tadataka Akimaru  

Real-time Video Pipeline Architect & Systems Engineer

This page is a bit long,  
so feel free to read **only the sections that interest you**.

---

## 📚 Topics

- [Real-time Video Pipeline Architect](#real-time-video-pipeline-architect)  
  → The hottest topic right now: safely handling RTSP without GPL contamination.

- [Satellite Ground Systems](#satellite-ground-systems)  
  → Ground stations, ANT/TLM/CMD, Quick Look, automation, and more.

- [Robotics & 3D/2D Vision](#robotics--3d2d-vision)  
  → Point clouds, 2D extraction, pose estimation.

- [Research Automation](#research-automation)  
  → Modernizing legacy research systems.

- [Tech Stack](#tech-stack)  
  → Frequently used technologies.

- [Legal Notice](#legal-notice)  
  → Notes on FFmpeg / OpenH264 / LGPL.

---

## Real-time Video Pipeline Architect
<p align="right"> <a href="#">to Top</a> </p>

Let me start with the conclusion.

- **I can decode camera input and provide raw frames without any GPL contamination (Non‑GPL).**  
- **I am also preparing a configuration where user applications can send processed Raw (YUV/RGB) frames back for Non‑GPL encoding.**

From the outside, it looks like a simple **“box that outputs raw frames.”**  
Inside, FFmpeg and OpenH264 are isolated inside Docker containers,  
so users never need to worry about licenses or internal structure.

This design allows you to safely extract camera input  
**without any GPL-related legal risk.**

Here is the architecture.

---

### Decoder pipeline (Docker inside)

- RTSP(S) input  
- Metadata & error checks via ffprobe  
- Demux / depay / preprocessing via FFmpeg (LGPL custom build)  
- H.264 decoding via OpenH264 (Cisco, Non‑GPL) inside Docker  
- Output only Raw (YUV/RGB) frames to the user application

```text
          RTSP(S)
        (IP cameras,
        NVR, etc.)
             |
             v
+----------------------------------------------------------+
|                      Docker Container                    |
|                                                          |
|  RTSP(S) Ingestion  →  FFprobe  →  FFmpeg(LGPL)          |
|                                      |                   |
|                                      v                   |
|                             pipe (IPC / bridge)          |
|                                      |                   |
|                                      v                   |
|                      OpenH264 (Cisco, Non‑GPL) decoder   |
|                      (H.264 → raw frames)                |
+----------------------------------------------------------+
             |
             v
   Raw frames (YUV/RGB) → User application
```

Users can process the frames in any language or style they prefer,  
without needing to understand the internal pipeline.

Because FFmpeg(LGPL) is custom-built,  
it can be rebuilt in many combinations depending on requirements.  
This flexibility allows you to avoid GPL contamination  
while still adjusting the pipeline as needed.

If the requirement is  
“GPL is fine — I just want maximum GPU performance,”  
I can also build a minimal FFmpeg configuration focused on GPU decode/encode.  
In that case, Docker isolation is avoided,  
because direct hardware access from Docker often reduces performance.

The pipeline is designed to adapt to a wide range of requirements.

---

### Encoder pipeline (planned)

The encoder side already exists on **Windows**,  
and I plan to **port it to C++**.

The idea is simple:

- Receive Raw (YUV/RGB) frames from the user application  
- Encode them into H.264 using OpenH264 (Cisco, Non‑GPL)  
- Optionally use FFmpeg(LGPL) for muxing / RTSP(S) streaming / file output

```text
User Application → Raw(YUV/RGB)
             |
             v
+----------------------------------------------------------+
|                      Encoder Module                      |
|                                                          |
|  OpenH264 (Cisco, Non‑GPL) encoder → H.264 bitstream     |
|                      |                                   |
|                      v                                   |
|           (optional) FFmpeg(LGPL) mux / RTSP(S) / file   |
+----------------------------------------------------------+
```

You can use only the decoder, only the encoder,  
or combine both.

Together, they form a simple structure:

```text
Camera -> Decoder    Encoder -> Rtsp(s)
            |          ^
            v          |
          user application
```

---

### Shared Memory / Zero-copy (planned option)

By default, I prioritize legal and operational stability  
through **Docker isolation**.

But future options may include:

- Shared memory frame transfer  
- Zero-copy AI preprocessing  
- Ultra-low-latency single-process configurations

The stance is:

**“Provide a safe Docker-based system first,  
and explore zero-copy options together when needed.”**

---

## Satellite Ground Systems
<p align="right"> <a href="#">to Top</a> </p>

In satellite ground systems, I have worked on:

- ANT/TLM/CMD design, implementation, and operations  
- Quick Look (real-time + playback) visualization  
- RF tracking, AGC/CN/BER monitoring  
- Automation (scheduling, state transitions, exception handling)

A simplified diagram looks like this:

```text
ANT -> D/C -> FPGA -> TLM -> Server -> Telemetry data -> Q/L
 ^                                         |
 |                                         v
U/C <--------FPGA <---------------------- CMD
```

- ANT(Antenna), D/C(Down Converter), TLM(Telemetry), Q/L(Quick Look)  
- CMD(Command), U/C(Up Converter)

For automated operations,  
the goal is to retrieve data from the onboard recorder efficiently.  
Higher bitrates are desirable,  
but they narrow the bandwidth and reduce the apparent signal level,  
increasing the risk of lock-off.

Human operators adjust bitrate or stop playback based on these conditions.  
I formalized these rules and automated them.

In short:

- Compute moving averages of AGC data from FPGA  
- Detect rapid level changes  
- Define thresholds for acceptable variation  
- Combine these with lower-level thresholds  
- Decide bitrate adjustments

Combined with UTC-based scheduling from GPS,  
this enabled automated operations.

Legally, uplink operations require supervision  
by a licensed operator,  
so full unmanned operation is not allowed.

However, **TLM-only reception** can be unmanned,  
allowing continuous health monitoring  
and long-term trend analysis of onboard instruments.

---

## Robotics & 3D/2D Vision
<p align="right"> <a href="#">to Top</a> </p>

In robotics, I handled the processing around robot motion:

- Point cloud generation using PCL  
- 3D-to-2D projection and extraction  
- Pose estimation and preprocessing for picking  
- Implementations in Linux / C++ / Python

I specialize in converting sensor data  
into forms that robots can use effectively.

Due to NDAs, I cannot provide further details.

---

## Research Automation
<p align="right"> <a href="#">to Top</a> </p>

For research institutions, I have worked on:

- Reconstructing behavior from old VB6 / legacy logs  
- Migrating to Java / TCP/IP  
- Redesigning UI  
- Ensuring maintainability for long-term operation

The original control software was written in VB6.0,  
but documentation was scarce.  
I analyzed actual logs to reconstruct commands and statuses,  
then migrated the system to Java.

GPIB and RS-232C devices were replaced  
with Linux-based onboard units performing protocol conversion,  
unifying everything under TCP/IP.

The result was a simpler, more maintainable system.

---

## Tech Stack
<p align="right"> <a href="#">to Top</a> </p>

- **Video / AI:** RTSP/RTSPS, FFmpeg(LGPL), OpenH264(Non‑GPL), Raw(YUV/RGB)  
- **Languages:** C/C++, C#, Python, Go  
- **OS:** Linux, Windows  
- **Robotics:** PCL, point cloud, pose estimation  
- **Satellite:** ANT/TLM/CMD, Quick Look, AGC/CN/BER analysis  

---

## Legal Notice
<p align="right"> <a href="#">to Top</a> </p>

This work relies on the following open-source components:

- **FFmpeg** (LGPL 2.1)  
  See: https://www.ffmpeg.org/legal.html  
- **OpenH264 by Cisco** (Non‑GPL binary license)  
  See: https://www.openh264.org/BINARY_LICENSE.txt

All components are used within their respective license terms.  
Decoder and encoder designs avoid GPL contamination by:

- using **LGPL-only FFmpeg builds**,  
- using **Cisco OpenH264 (Non‑GPL)**, and  
- isolating these components inside **Docker containers** where appropriate.

If you need a custom build or have legal constraints,  
I can help design a configuration that stays within these boundaries.

---
