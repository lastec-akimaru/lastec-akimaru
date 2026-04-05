<p align="right">
  <a href="README.md">English version is here</a>
</p>

## Introduction
秋丸 忠隆 / Tadatka Akimaru  

Real-time Video Pipeline Architect & Systems Engineer

このページは少し長いので、  
**気になるところだけ読んでいただければ十分です。**

---

## 📚 Topics

- [Real-time Video Pipeline Architect](#real-time-video-pipeline-architect)  
  → 今いちばん “熱い” 話題。RTSP を Non‑GPL で安全に扱う話です。

- [Satellite Ground Systems](#satellite-ground-systems)  
  → 地上局、ANT/TLM/CMD、Quick Look、運用自動化など。

- [Robotics & 3D/2D Vision](#robotics--3d2d-vision)  
  → 点群処理、2D 抽出、姿勢推定など。

- [Research Automation](#research-automation)  
  → 研究現場のレガシーシステムを現代化する話。

- [Tech Stack](#tech-stack)  
  → よく使う技術の一覧です。

- [Legal Notice](#legal-notice)  
  → FFmpeg / OpenH264 / LGPL まわりの情報。

---

## Real-time Video Pipeline Architect
<p align="right"> <a href="#">to Top</a> </p>

まず、何が言いたいか、結論から書きます。

- **カメラからの入力を、GPL 汚染なし（Non‑GPL）でデコードして、生データとして提供できます。**  
- **ユーザアプリケーションで処理された Raw(YUV/RGB) を受け取り、同じく Non‑GPL でエンコードして出力する構成も準備中です。**

外から見えるのは、ただの **「生フレームが出てくる箱」** だけです。  
内部で動く FFmpeg や OpenH264 は Docker 内に閉じ込めてあり、  
利用者がライセンスや構成を意識する必要はありません。

この構成により、**カメラからの入力を GPL 汚染の心配なく、安全に取り出すことができます。**

それでは、構成の説明に入ります。

---

### Decoder pipeline (Docker inside)

- RTSP(S) からの入力  
- ffprobe によるメタデータ・エラーチェック  
- FFmpeg(LGPL custom build) による demux / depay / 前処理  
- Docker 内での OpenH264(Cisco, Non-GPL) デコード  
- 外部には Raw(YUV/RGB) のみを渡す構成

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

ご覧の通り、ユーザは内部の構成を意識することなく、
好みの言語やスタイルで画像処理を行うことができます。

また、FFmpeg(LGPL) はカスタムビルドしているため、
用途に応じて多様な組み合わせでリビルド可能です。  
これは、必要に応じて GPL 汚染を避けながら、
柔軟に構成を変更できるということでもあります。

一方で、「GPL でも構わない、とにかく GPU を最大限に生かしたい」
という要件にも対応できます。  
その場合は FFmpeg による decode/encode を中心に構成し、
必要な機能だけを選んだ最小構成の FFmpeg を用意することも可能です。

ただし、この構成では Docker 化は避け、
自立したアプリケーションとして設計することを考えています。 
理由は、Docker から直接ハードウェアへアクセスする場合、
パフォーマンス面で不利になるケースが多いためです。

このように、要件に合わせて幅広く調整可能な構成になっています。

---

### Encoder pipeline (planned)

Encoder 側は、すでに **Windows で実装が存在**していて、  
それを **C++ にポーティングする予定**です。

やりたいことはシンプルで：

- ユーザアプリケーションから Raw(YUV/RGB) を受け取る  
- OpenH264(Cisco, Non-GPL) encoder で H.264 にする  
- 必要に応じて FFmpeg(LGPL) で mux / RTSP(S) 配信 / ファイル保存

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

Decoder だけ欲しい構成、Encoder だけ欲しい構成、  
両方をつなげたい構成、どれも切り出せるように設計しています。

DecoderとEncoderの両方を組み合わせると、下記のようになります。

```text
Camera -> Decoder    Encoder -> Rtsp(s)
            |          ^
            v          |
          user application
```
とてもシンプルな構成に仕上げることができると思います。

---

### Shared Memory / Zero-copy (planned option)

デフォルトは **Docker 分離** による法務・運用の安定性を優先していますが、  
将来的には、以下のようなオプションも検討しています。

- Shared memory によるフレーム受け渡し  
- Zero-copy での AI 前処理連携  
- 同一プロセス空間での超低レイテンシ構成

「まずは安全に動くものを Docker で提供し、  
必要になったら一緒に Zero-copy 構成を検討する」というスタンスです。

---

## Satellite Ground Systems
<p align="right"> <a href="#">to Top</a> </p>

衛星の地上局まわりでは、以下のようなことを担当してきました。

- ANT/TLM/CMD 系の設計・実装・運用  
- Quick Look（リアルタイム＋再生）の可視化  
- RF トラッキング、AGC/CN/BER の監視  
- 運用自動化（スケジューリング、状態遷移、例外処理）

地上系の構成を簡単に表すと以下のようになります。

```text
ANT -> D/C -> FPGA -> TLM -> Server -> Telemetry data -> Q/L
 ^                                         |
 |                                         v
U/C <--------FPGA <---------------------- CMD

```

- ANT(Antenna), D/C(Down Converter), TLM(Telemetry), Q/L(Quick Look)
- CMD(Command), U/C(Up Converter)

自動運用では、衛星のデータレコーダに蓄積したデータを効率よく取得するために、
ビットレートを高くしたいと考えます。
しかし、ビットレートを上げると帯域が狭くなり、見た目の受信レベルが下がります。
受信レベルが下がると、ロックオフを起こす恐れが高くなります。

人間がオペレーションする場合は、これらを判断しながらビットレートを下げたり、
データレコーダの再生を止めたりします。

これらの判断をルールベースで整理し、自動化しました。
簡単に言うと、FPGA から得られる AGC データの移動平均を計算し、
急激なレベル変化を監視します。
急激な変化の許容範囲から閾値を決め、
この閾値とレベル下限の閾値を論理和で組み合わせて、
ビットレートの変更を決定します。

このようなルールベースの処理と、GPS から得られる UTC に従ったスケジュールを組み合わせることで、
自動運用を実現しました。

ただし、法律上、アンテナから送信を行う場合は、
国家資格を持つ管理者の監督下で運用することが義務付けられています。
そのため、完全な無人運用はできません。

しかし、送信を行わず（CMD を送らず）、
TLM の受信のみを行う場合は無人化が可能です。
TLM を取得することで、衛星のヘルスチェックや、
搭載機器の経年変化のデータを蓄積することができます。
これらを継続的に集めることは、とても意味のあることだと思っています。

---

## Robotics & 3D/2D Vision
<p align="right"> <a href="#">to Top</a> </p>

ロボティクスでは、ロボットが動くための、前後の処理を担当しました。

- PCL を使った点群生成  
- 3D から 2D への投影・抽出  
- 姿勢推定・ピッキング向けの前処理  
- Linux / C++ / Python ベースの実装

カメラやセンサからのデータを、  
「ロボットが扱いやすい形」に変換する部分が得意です。

大変申し訳ございませんが、こちらについては、守秘義務により、詳細を記載することはできません。


---

## Research Automation
<p align="right"> <a href="#">to Top</a> </p>

研究機関向けには、例えばこんなことをしてきました。

- 古い VB6 / レガシーシステムのログから挙動を再構成  
- Java / TCP/IP ベースへの置き換え  
- UI の再設計  
- 長期運用を前提とした保守性の確保

当時 VB6.0 で実装されていた制御ソフトを、Java へ移行しました。  
VB 側の設計資料がほとんど残っていなかったため、  
実際の動作ログからコマンドやステータスを調査し、Java へ置き換えました。

また、GPIB や RS-232C で制御していた部分は、  
Linux 搭載のオンボード機器でプロトコル変換を行い、  
TCP/IP に統一する構成にしました。

Java への移行と TCP/IP への統一により、  
シンプルで保守しやすいシステムに刷新できたのではないかと思っています。

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
  See: [https://www.ffmpeg.org/legal.html](https://www.ffmpeg.org/legal.html)  
- **OpenH264 by Cisco** (Non‑GPL binary license)  
  See: `https://www.openh264.org/BINARY_LICENSE.txt` [(openh264.org in Bing)](https://www.bing.com/search?q="https%3A%2F%2Fwww.openh264.org%2FBINARY_LICENSE.txt")  

All components are used within their respective license terms.  
Decoder and encoder designs are intended to avoid GPL contamination by:

- using **LGPL-only FFmpeg builds**,  
- using **Cisco OpenH264 (Non‑GPL)**, and  
- isolating these components inside **Docker containers** where appropriate.

If you need a custom build or have legal constraints,  
I can help design a configuration that stays within these boundaries.

---
