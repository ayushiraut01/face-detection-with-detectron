<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Face Detection with Detectron2</title>
<link href="https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Space+Mono:wght@400;700&family=Rajdhani:wght@300;500;700&display=swap" rel="stylesheet">
<style>
  :root {
    --bg: #060810;
    --surface: #0d1117;
    --card: #111827;
    --accent: #00f5ff;
    --accent2: #ff2d78;
    --accent3: #7c3aed;
    --text: #e2e8f0;
    --muted: #64748b;
    --border: rgba(0,245,255,0.15);
  }

  * { margin: 0; padding: 0; box-sizing: border-box; }

  body {
    background: var(--bg);
    color: var(--text);
    font-family: 'Rajdhani', sans-serif;
    font-size: 17px;
    line-height: 1.7;
    overflow-x: hidden;
  }

  /* SCANLINE OVERLAY */
  body::before {
    content: '';
    position: fixed;
    inset: 0;
    background: repeating-linear-gradient(0deg, transparent, transparent 2px, rgba(0,245,255,0.015) 2px, rgba(0,245,255,0.015) 4px);
    pointer-events: none;
    z-index: 100;
  }

  /* HERO */
  .hero {
    position: relative;
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 80px 60px;
    overflow: hidden;
  }

  .hero-grid {
    position: absolute;
    inset: 0;
    background-image:
      linear-gradient(rgba(0,245,255,0.06) 1px, transparent 1px),
      linear-gradient(90deg, rgba(0,245,255,0.06) 1px, transparent 1px);
    background-size: 60px 60px;
    animation: gridMove 20s linear infinite;
  }

  @keyframes gridMove {
    0% { transform: translateY(0); }
    100% { transform: translateY(60px); }
  }

  .hero-glow {
    position: absolute;
    width: 700px;
    height: 700px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(0,245,255,0.08) 0%, transparent 70%);
    top: -200px;
    right: -200px;
    pointer-events: none;
  }

  .hero-glow2 {
    position: absolute;
    width: 500px;
    height: 500px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(255,45,120,0.07) 0%, transparent 70%);
    bottom: -150px;
    left: -100px;
    pointer-events: none;
  }

  .badge {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    background: rgba(0,245,255,0.08);
    border: 1px solid var(--border);
    color: var(--accent);
    font-family: 'Space Mono', monospace;
    font-size: 11px;
    padding: 6px 16px;
    letter-spacing: 3px;
    text-transform: uppercase;
    margin-bottom: 28px;
    position: relative;
    z-index: 2;
    width: fit-content;
    animation: fadeSlideIn 0.8s ease forwards;
  }

  .badge::before {
    content: '';
    width: 6px;
    height: 6px;
    background: var(--accent);
    border-radius: 50%;
    animation: pulse 2s infinite;
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; transform: scale(1); }
    50% { opacity: 0.4; transform: scale(0.7); }
  }

  .hero-title {
    font-family: 'Bebas Neue', sans-serif;
    font-size: clamp(64px, 9vw, 130px);
    line-height: 0.9;
    letter-spacing: -1px;
    position: relative;
    z-index: 2;
    animation: fadeSlideIn 0.9s ease 0.1s both;
  }

  .hero-title .line1 { color: var(--text); }
  .hero-title .line2 {
    background: linear-gradient(135deg, var(--accent), var(--accent3));
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }
  .hero-title .line3 { color: var(--accent2); }

  .hero-subtitle {
    font-size: 16px;
    color: var(--muted);
    max-width: 600px;
    margin-top: 24px;
    font-family: 'Space Mono', monospace;
    line-height: 1.8;
    position: relative;
    z-index: 2;
    animation: fadeSlideIn 1s ease 0.2s both;
  }

  .hero-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin-top: 40px;
    position: relative;
    z-index: 2;
    animation: fadeSlideIn 1.1s ease 0.3s both;
  }

  .tag {
    background: rgba(124,58,237,0.15);
    border: 1px solid rgba(124,58,237,0.4);
    color: #a78bfa;
    font-family: 'Space Mono', monospace;
    font-size: 11px;
    padding: 6px 14px;
    letter-spacing: 1px;
    transition: all 0.3s;
  }
  .tag:hover {
    background: rgba(124,58,237,0.3);
    transform: translateY(-2px);
  }

  /* FACE DETECTION BOX ANIMATION */
  .detection-visual {
    position: absolute;
    right: 80px;
    top: 50%;
    transform: translateY(-50%);
    width: 280px;
    height: 320px;
    z-index: 2;
    display: none;
  }

  @media (min-width: 900px) { .detection-visual { display: block; } }

  .face-box {
    position: absolute;
    inset: 30px;
    border: 2px solid var(--accent);
    animation: scanBox 3s ease-in-out infinite;
  }

  .face-box::before, .face-box::after {
    content: '';
    position: absolute;
    width: 20px;
    height: 20px;
    border-color: var(--accent2);
    border-style: solid;
  }
  .face-box::before { top: -2px; left: -2px; border-width: 3px 0 0 3px; }
  .face-box::after { bottom: -2px; right: -2px; border-width: 0 3px 3px 0; }

  .face-icon {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
  }

  .face-circle {
    width: 80px;
    height: 80px;
    border-radius: 50%;
    border: 2px dashed rgba(0,245,255,0.4);
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 40px;
    animation: rotate 8s linear infinite;
  }

  @keyframes rotate {
    to { transform: rotate(360deg); }
  }

  .scan-line {
    position: absolute;
    left: 0; right: 0;
    height: 2px;
    background: linear-gradient(90deg, transparent, var(--accent), transparent);
    animation: scan 2.5s ease-in-out infinite;
    box-shadow: 0 0 10px var(--accent);
  }

  @keyframes scan {
    0% { top: 30px; opacity: 0; }
    10% { opacity: 1; }
    90% { opacity: 1; }
    100% { top: calc(100% - 30px); opacity: 0; }
  }

  .detection-label {
    position: absolute;
    top: 10px;
    left: 30px;
    font-family: 'Space Mono', monospace;
    font-size: 10px;
    color: var(--accent);
    letter-spacing: 2px;
    animation: blink 1.5s step-end infinite;
  }

  @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0.3; } }

  .conf-badge {
    position: absolute;
    bottom: 14px;
    right: 30px;
    font-family: 'Space Mono', monospace;
    font-size: 10px;
    color: #4ade80;
    letter-spacing: 1px;
  }

  @keyframes scanBox {
    0%, 100% { box-shadow: 0 0 20px rgba(0,245,255,0.3), inset 0 0 20px rgba(0,245,255,0.05); }
    50% { box-shadow: 0 0 40px rgba(0,245,255,0.6), inset 0 0 30px rgba(0,245,255,0.1); }
  }

  @keyframes fadeSlideIn {
    from { opacity: 0; transform: translateY(30px); }
    to { opacity: 1; transform: translateY(0); }
  }

  /* SECTIONS */
  .container {
    max-width: 1100px;
    margin: 0 auto;
    padding: 0 60px;
  }

  section { padding: 80px 0; border-top: 1px solid rgba(255,255,255,0.05); }

  .section-label {
    font-family: 'Space Mono', monospace;
    font-size: 11px;
    color: var(--accent);
    letter-spacing: 4px;
    text-transform: uppercase;
    margin-bottom: 16px;
    display: flex;
    align-items: center;
    gap: 12px;
  }
  .section-label::after {
    content: '';
    flex: 1;
    height: 1px;
    background: var(--border);
    max-width: 100px;
  }

  .section-title {
    font-family: 'Bebas Neue', sans-serif;
    font-size: clamp(42px, 6vw, 72px);
    line-height: 1;
    margin-bottom: 40px;
    color: var(--text);
  }

  /* OVERVIEW CARDS */
  .overview-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
  }

  @media (max-width: 700px) { .overview-grid { grid-template-columns: 1fr; } }

  .overview-card {
    background: var(--card);
    border: 1px solid rgba(255,255,255,0.07);
    padding: 32px;
    position: relative;
    overflow: hidden;
    transition: transform 0.3s, border-color 0.3s;
  }

  .overview-card:hover {
    transform: translateY(-4px);
    border-color: rgba(0,245,255,0.25);
  }

  .overview-card::before {
    content: '';
    position: absolute;
    top: 0; left: 0; right: 0;
    height: 2px;
    background: linear-gradient(90deg, var(--accent), var(--accent3));
    opacity: 0;
    transition: opacity 0.3s;
  }
  .overview-card:hover::before { opacity: 1; }

  .overview-card .num {
    font-family: 'Bebas Neue', sans-serif;
    font-size: 64px;
    color: rgba(0,245,255,0.12);
    line-height: 1;
    margin-bottom: 8px;
    transition: color 0.3s;
  }
  .overview-card:hover .num { color: rgba(0,245,255,0.2); }

  .overview-card h3 {
    font-family: 'Bebas Neue', sans-serif;
    font-size: 28px;
    letter-spacing: 1px;
    color: var(--text);
    margin-bottom: 10px;
  }

  .overview-card p {
    color: var(--muted);
    font-size: 15px;
    line-height: 1.7;
  }

  /* FEATURES */
  .features-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 16px;
  }

  .feature-item {
    background: var(--card);
    border: 1px solid rgba(255,255,255,0.06);
    padding: 28px;
    display: flex;
    flex-direction: column;
    gap: 12px;
    transition: all 0.3s;
    position: relative;
    overflow: hidden;
  }

  .feature-item:hover {
    border-color: rgba(255,45,120,0.3);
    transform: translateY(-3px);
  }

  .feature-icon {
    font-size: 28px;
    line-height: 1;
  }

  .feature-item h3 {
    font-family: 'Rajdhani', sans-serif;
    font-weight: 700;
    font-size: 18px;
    color: var(--text);
    letter-spacing: 0.5px;
  }

  .feature-item p {
    color: var(--muted);
    font-size: 14px;
    line-height: 1.7;
  }

  .feature-item .tag-list {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 4px;
  }

  .mini-tag {
    font-family: 'Space Mono', monospace;
    font-size: 10px;
    padding: 3px 8px;
    background: rgba(255,45,120,0.1);
    border: 1px solid rgba(255,45,120,0.25);
    color: #fb7185;
    letter-spacing: 1px;
  }

  /* TECH TABLE */
  .tech-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 12px;
  }

  .tech-card {
    background: var(--card);
    border: 1px solid rgba(255,255,255,0.06);
    padding: 24px;
    text-align: center;
    transition: all 0.3s;
  }

  .tech-card:hover {
    background: rgba(124,58,237,0.1);
    border-color: rgba(124,58,237,0.4);
    transform: translateY(-3px);
  }

  .tech-emoji { font-size: 32px; margin-bottom: 10px; }
  .tech-name {
    font-family: 'Bebas Neue', sans-serif;
    font-size: 22px;
    color: var(--text);
    letter-spacing: 1px;
    margin-bottom: 4px;
  }
  .tech-role {
    font-family: 'Space Mono', monospace;
    font-size: 11px;
    color: var(--muted);
    letter-spacing: 1px;
  }

  /* CODE BLOCKS */
  .code-block {
    background: #0a0e18;
    border: 1px solid var(--border);
    border-left: 3px solid var(--accent);
    padding: 24px 28px;
    font-family: 'Space Mono', monospace;
    font-size: 13px;
    color: var(--accent);
    margin: 16px 0;
    position: relative;
    overflow-x: auto;
  }

  .code-block .comment { color: var(--muted); }
  .code-block .cmd { color: #f9a8d4; }
  .code-block .path { color: #86efac; }

  .code-label {
    font-family: 'Space Mono', monospace;
    font-size: 10px;
    color: var(--muted);
    letter-spacing: 2px;
    text-transform: uppercase;
    margin-bottom: 8px;
  }

  /* INSTALL STEPS */
  .install-steps {
    display: flex;
    flex-direction: column;
    gap: 0;
  }

  .install-step {
    display: flex;
    gap: 24px;
    padding: 32px 0;
    border-bottom: 1px solid rgba(255,255,255,0.05);
    position: relative;
  }

  .step-num {
    font-family: 'Bebas Neue', sans-serif;
    font-size: 48px;
    color: rgba(0,245,255,0.15);
    line-height: 1;
    min-width: 60px;
    transition: color 0.3s;
  }

  .install-step:hover .step-num { color: rgba(0,245,255,0.4); }

  .step-content h3 {
    font-family: 'Rajdhani', sans-serif;
    font-weight: 700;
    font-size: 20px;
    color: var(--text);
    margin-bottom: 12px;
    letter-spacing: 0.5px;
  }

  /* APPLICATIONS */
  .app-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
    gap: 12px;
  }

  .app-item {
    background: var(--card);
    border: 1px solid rgba(255,255,255,0.06);
    padding: 20px 24px;
    display: flex;
    align-items: center;
    gap: 16px;
    font-family: 'Rajdhani', sans-serif;
    font-weight: 700;
    font-size: 17px;
    transition: all 0.3s;
    letter-spacing: 0.3px;
  }

  .app-item:hover {
    background: rgba(0,245,255,0.05);
    border-color: var(--border);
    transform: translateX(6px);
  }

  .app-dot {
    width: 8px;
    height: 8px;
    background: var(--accent);
    border-radius: 50%;
    flex-shrink: 0;
    box-shadow: 0 0 10px var(--accent);
  }

  /* FUTURE */
  .future-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: 12px;
  }

  .future-item {
    background: var(--card);
    border: 1px solid rgba(255,255,255,0.06);
    border-left: 3px solid var(--accent3);
    padding: 20px 24px;
    font-family: 'Rajdhani', sans-serif;
    font-weight: 500;
    font-size: 16px;
    color: var(--muted);
    display: flex;
    align-items: center;
    gap: 12px;
    transition: all 0.3s;
  }

  .future-item:hover {
    border-left-color: var(--accent2);
    color: var(--text);
    transform: translateY(-2px);
  }

  .future-arrow { color: var(--accent3); font-size: 18px; }

  /* PROJECT STRUCTURE */
  .struct-tree {
    background: #0a0e18;
    border: 1px solid var(--border);
    padding: 28px;
    font-family: 'Space Mono', monospace;
    font-size: 13px;
    line-height: 2;
  }

  .struct-dir { color: var(--accent); }
  .struct-file { color: #a78bfa; margin-left: 24px; }
  .struct-comment { color: var(--muted); }

  /* AUTHOR */
  .author-card {
    background: var(--card);
    border: 1px solid rgba(255,255,255,0.08);
    padding: 40px;
    display: flex;
    gap: 32px;
    align-items: flex-start;
    position: relative;
    overflow: hidden;
  }

  .author-card::before {
    content: '';
    position: absolute;
    top: 0; left: 0; right: 0;
    height: 3px;
    background: linear-gradient(90deg, var(--accent2), var(--accent3), var(--accent));
  }

  .author-avatar {
    width: 80px;
    height: 80px;
    background: linear-gradient(135deg, var(--accent3), var(--accent2));
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 36px;
    flex-shrink: 0;
  }

  .author-info h2 {
    font-family: 'Bebas Neue', sans-serif;
    font-size: 36px;
    letter-spacing: 2px;
    color: var(--text);
    margin-bottom: 4px;
  }

  .author-role {
    font-family: 'Space Mono', monospace;
    font-size: 12px;
    color: var(--accent);
    letter-spacing: 2px;
    margin-bottom: 16px;
  }

  .author-links {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    margin-top: 16px;
  }

  .author-link {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: 'Space Mono', monospace;
    font-size: 12px;
    color: var(--muted);
    text-decoration: none;
    border: 1px solid rgba(255,255,255,0.08);
    padding: 8px 16px;
    transition: all 0.3s;
  }

  .author-link:hover {
    color: var(--accent);
    border-color: var(--border);
    background: rgba(0,245,255,0.05);
  }

  /* LICENSE */
  .license-badge {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    background: rgba(124,58,237,0.12);
    border: 1px solid rgba(124,58,237,0.35);
    padding: 14px 24px;
    font-family: 'Space Mono', monospace;
    font-size: 13px;
    color: #a78bfa;
    letter-spacing: 2px;
  }

  /* FOOTER */
  footer {
    background: #060810;
    border-top: 1px solid rgba(255,255,255,0.05);
    padding: 40px 60px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    flex-wrap: gap;
    gap: 20px;
  }

  .footer-brand {
    font-family: 'Bebas Neue', sans-serif;
    font-size: 22px;
    color: var(--muted);
    letter-spacing: 3px;
  }

  .footer-note {
    font-family: 'Space Mono', monospace;
    font-size: 11px;
    color: rgba(100,116,139,0.6);
    letter-spacing: 1px;
  }

  @media (max-width: 640px) {
    .hero, .container, footer { padding-left: 24px; padding-right: 24px; }
    .author-card { flex-direction: column; }
  }
</style>
</head>
<body>

<!-- HERO -->
<section class="hero">
  <div class="hero-grid"></div>
  <div class="hero-glow"></div>
  <div class="hero-glow2"></div>

  <div style="position:relative;z-index:2;max-width:700px">
    <div class="badge">🎯 Computer Vision Project</div>

    <h1 class="hero-title">
      <div class="line1">FACE</div>
      <div class="line2">DETECTION</div>
      <div class="line3">DETECTRON2</div>
    </h1>

    <p class="hero-subtitle">
      Powered by Facebook AI's Detectron2 · Faster R-CNN & Mask R-CNN ·<br>
      Real-time detection across images, videos & live webcam streams.
    </p>

    <div class="hero-tags">
      <span class="tag">PyTorch</span>
      <span class="tag">Detectron2</span>
      <span class="tag">OpenCV</span>
      <span class="tag">Faster R-CNN</span>
      <span class="tag">Mask R-CNN</span>
      <span class="tag">COCO Dataset</span>
      <span class="tag">Python</span>
    </div>
  </div>

  <!-- Detection Box Visual -->
  <div class="detection-visual">
    <div class="face-box">
      <div class="scan-line"></div>
      <div class="face-icon">
        <div class="face-circle">👤</div>
        <div style="font-family:'Space Mono',monospace;font-size:11px;color:rgba(0,245,255,0.6);letter-spacing:1px;">ANALYZING...</div>
      </div>
      <div class="detection-label">FACE_001</div>
      <div class="conf-badge">CONF: 98.7%</div>
    </div>
  </div>
</section>

<!-- OVERVIEW -->
<section>
  <div class="container">
    <div class="section-label">01 — Overview</div>
    <h2 class="section-title">PROJECT OVERVIEW</h2>

    <p style="color:var(--muted);max-width:720px;margin-bottom:40px;font-size:16px;line-height:1.9">
      Face detection is a core component in AI-driven applications — from surveillance systems 
      and facial recognition platforms to smart attendance tracking. This project implements 
      a deep learning-based pipeline that detects and highlights human faces in real-time using 
      bounding boxes, powered by Detectron2.
    </p>

    <div class="overview-grid">
      <div class="overview-card">
        <div class="num">01</div>
        <h3>IMAGES</h3>
        <p>Static image face detection with high-precision bounding box visualization and confidence scores.</p>
      </div>
      <div class="overview-card">
        <div class="num">02</div>
        <h3>VIDEOS</h3>
        <p>Frame-by-frame video processing pipeline that detects and tracks faces across every frame.</p>
      </div>
      <div class="overview-card">
        <div class="num">03</div>
        <h3>LIVE WEBCAM</h3>
        <p>Real-time webcam stream detection with near-instantaneous face identification and annotation.</p>
      </div>
      <div class="overview-card">
        <div class="num">04</div>
        <h3>DEEP LEARNING</h3>
        <p>Built on Faster R-CNN and Mask R-CNN architectures pre-trained on the COCO dataset for maximum accuracy.</p>
      </div>
    </div>
  </div>
</section>

<!-- FEATURES -->
<section>
  <div class="container">
    <div class="section-label">02 — Features</div>
    <h2 class="section-title">KEY FEATURES</h2>

    <div class="features-grid">
      <div class="feature-item">
        <div class="feature-icon">⚡</div>
        <h3>Real-Time Face Detection</h3>
        <p>Detects human faces instantly using advanced object detection models optimized for speed and precision.</p>
      </div>
      <div class="feature-item">
        <div class="feature-icon">🎯</div>
        <h3>High Accuracy Detection</h3>
        <p>Uses Detectron2's Faster R-CNN and Mask R-CNN models for industry-leading detection precision.</p>
      </div>
      <div class="feature-item">
        <div class="feature-icon">📷</div>
        <h3>Live Webcam Support</h3>
        <p>Seamlessly connects to any webcam for real-time detection from live camera feeds.</p>
      </div>
      <div class="feature-item">
        <div class="feature-icon">🔲</div>
        <h3>Bounding Box Visualization</h3>
        <p>Draws precise bounding boxes around detected faces with confidence scores for clear identification.</p>
      </div>
      <div class="feature-item">
        <div class="feature-icon">🧠</div>
        <h3>Deep Learning Powered</h3>
        <p>Built using PyTorch and Detectron2, ensuring state-of-the-art performance on complex visual inputs.</p>
      </div>
      <div class="feature-item">
        <div class="feature-icon">🔌</div>
        <h3>Easy to Extend</h3>
        <p>Modular codebase designed to be expanded with additional AI capabilities:</p>
        <div class="tag-list">
          <span class="mini-tag">Face Recognition</span>
          <span class="mini-tag">Emotion Detection</span>
          <span class="mini-tag">Mask Detection</span>
          <span class="mini-tag">Attendance System</span>
        </div>
      </div>
    </div>
  </div>
</section>

<!-- TECHNOLOGIES -->
<section>
  <div class="container">
    <div class="section-label">03 — Stack</div>
    <h2 class="section-title">TECHNOLOGIES USED</h2>

    <div class="tech-grid">
      <div class="tech-card">
        <div class="tech-emoji">🐍</div>
        <div class="tech-name">Python</div>
        <div class="tech-role">Programming Language</div>
      </div>
      <div class="tech-card">
        <div class="tech-emoji">🔥</div>
        <div class="tech-name">PyTorch</div>
        <div class="tech-role">Deep Learning Framework</div>
      </div>
      <div class="tech-card">
        <div class="tech-emoji">🤖</div>
        <div class="tech-name">Detectron2</div>
        <div class="tech-role">Object Detection Framework</div>
      </div>
      <div class="tech-card">
        <div class="tech-emoji">👁️</div>
        <div class="tech-name">OpenCV</div>
        <div class="tech-role">Image & Video Processing</div>
      </div>
      <div class="tech-card">
        <div class="tech-emoji">📦</div>
        <div class="tech-name">COCO Dataset</div>
        <div class="tech-role">Pre-trained Model Training</div>
      </div>
    </div>
  </div>
</section>

<!-- PROJECT STRUCTURE -->
<section>
  <div class="container">
    <div class="section-label">04 — Structure</div>
    <h2 class="section-title">PROJECT STRUCTURE</h2>

    <div class="struct-tree">
      <div class="struct-dir">📁 face-detection-with-detectron2/</div>
      <div class="struct-file">├── <span style="color:#86efac">face_detection_with_detectron.py</span> &nbsp;<span class="struct-comment">// Main detection script</span></div>
      <div class="struct-file">├── <span style="color:#fbbf24">README.md</span> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="struct-comment">// Documentation</span></div>
      <div class="struct-file">└── <span style="color:#a78bfa">LICENSE</span> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="struct-comment">// Apache 2.0</span></div>
    </div>
  </div>
</section>

<!-- INSTALLATION -->
<section>
  <div class="container">
    <div class="section-label">05 — Setup</div>
    <h2 class="section-title">INSTALLATION</h2>

    <div class="install-steps">
      <div class="install-step">
        <div class="step-num">01</div>
        <div class="step-content">
          <h3>Clone the Repository</h3>
          <div class="code-label">TERMINAL</div>
          <div class="code-block">
            <span class="comment"># Clone via Git</span><br>
            <span class="cmd">git clone</span> <span class="path">https://github.com/ayushiraut01/face-detection-with-detectron.git</span>
          </div>
        </div>
      </div>

      <div class="install-step">
        <div class="step-num">02</div>
        <div class="step-content">
          <h3>Navigate to Project Folder</h3>
          <div class="code-label">TERMINAL</div>
          <div class="code-block">
            <span class="cmd">cd</span> <span class="path">face-detection-with-detectron</span>
          </div>
        </div>
      </div>

      <div class="install-step">
        <div class="step-num">03</div>
        <div class="step-content">
          <h3>Install Dependencies</h3>
          <div class="code-label">TERMINAL</div>
          <div class="code-block">
            <span class="comment"># Install PyTorch</span><br>
            <span class="cmd">pip install</span> <span class="path">torch torchvision</span><br><br>
            <span class="comment"># Install OpenCV</span><br>
            <span class="cmd">pip install</span> <span class="path">opencv-python</span><br><br>
            <span class="comment"># Install Detectron2</span><br>
            <span class="cmd">pip install</span> <span class="path">detectron2</span>
          </div>
        </div>
      </div>

      <div class="install-step">
        <div class="step-num">04</div>
        <div class="step-content">
          <h3>Run the Detection Script</h3>
          <div class="code-label">TERMINAL</div>
          <div class="code-block">
            <span class="cmd">python</span> <span class="path">face_detection_with_detectron.py</span>
          </div>
          <p style="color:var(--muted);font-size:14px;margin-top:12px;font-family:'Space Mono',monospace">
            → System auto-detects faces and renders bounding boxes.
          </p>
        </div>
      </div>
    </div>
  </div>
</section>

<!-- APPLICATIONS -->
<section>
  <div class="container">
    <div class="section-label">06 — Applications</div>
    <h2 class="section-title">REAL-WORLD USE CASES</h2>

    <div class="app-list">
      <div class="app-item"><div class="app-dot"></div>Smart Surveillance Systems</div>
      <div class="app-item"><div class="app-dot"></div>Face Recognition Applications</div>
      <div class="app-item"><div class="app-dot"></div>Automated Attendance Systems</div>
      <div class="app-item"><div class="app-dot"></div>Security & Access Control</div>
      <div class="app-item"><div class="app-dot"></div>Social Media Image Tagging</div>
      <div class="app-item"><div class="app-dot"></div>AI-Powered Camera Systems</div>
    </div>
  </div>
</section>

<!-- FUTURE -->
<section>
  <div class="container">
    <div class="section-label">07 — Roadmap</div>
    <h2 class="section-title">FUTURE IMPROVEMENTS</h2>

    <div class="future-grid">
      <div class="future-item"><span class="future-arrow">→</span> Face Recognition System</div>
      <div class="future-item"><span class="future-arrow">→</span> Emotion Detection</div>
      <div class="future-item"><span class="future-arrow">→</span> Mask Detection</div>
      <div class="future-item"><span class="future-arrow">→</span> Multi-Face Tracking</div>
      <div class="future-item"><span class="future-arrow">→</span> Web Application Deployment</div>
      <div class="future-item"><span class="future-arrow">→</span> Mobile & Cloud Integration</div>
    </div>
  </div>
</section>

<!-- AUTHOR -->
<section>
  <div class="container">
    <div class="section-label">08 — Author</div>
    <h2 class="section-title">ABOUT THE AUTHOR</h2>

    <div class="author-card">
      <div class="author-avatar">👩‍💻</div>
      <div class="author-info">
        <h2>AYUSHI RAUT</h2>
        <div class="author-role">SOFTWARE & AI ENGINEER</div>
        <p style="color:var(--muted);font-size:15px;max-width:500px;line-height:1.8">
          Building intelligent systems at the intersection of computer vision, 
          deep learning, and real-world AI applications.
        </p>
        <div class="author-links">
          <a href="mailto:ayushiraut8@gmail.com" class="author-link">📧 ayushiraut8@gmail.com</a>
          <a href="https://github.com/ayushiraut01" class="author-link" target="_blank">🐙 github.com/ayushiraut01</a>
        </div>
      </div>
    </div>
  </div>
</section>

<!-- LICENSE -->
<section style="padding-bottom:60px">
  <div class="container">
    <div class="section-label">09 — License</div>
    <h2 class="section-title">LICENSE</h2>
    <div class="license-badge">⚖️ &nbsp; APACHE LICENSE 2.0</div>
    <p style="color:var(--muted);font-size:14px;margin-top:16px;font-family:'Space Mono',monospace;line-height:1.8">
      This project is freely available under the Apache 2.0 License.<br>
      Use it, fork it, build on it.
    </p>
  </div>
</section>

<!-- FOOTER -->
<footer>
  <div class="footer-brand">FACE DETECTION · DETECTRON2</div>
  <div class="footer-note">Built with PyTorch · Detectron2 · OpenCV · Apache 2.0</div>
</footer>

</body>
</html>
