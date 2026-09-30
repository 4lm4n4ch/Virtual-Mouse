# Virtual Mouse / Hand Gesture Camera Control: Research Report

**Date:** 2026-09-21
**Scope:** Software-only solutions for laptop + built-in webcam (no embedded hardware)
**Context:** University project evaluation

---

## Table of Contents

1. [Overview of Approaches](#1-overview-of-approaches)
2. [Detailed Analysis of Each Approach](#2-detailed-analysis-of-each-approach)
3. [Comparison Matrix](#3-comparison-matrix)
4. [CONDENSATION vs MediaPipe: Deep Dive](#4-condensation-vs-mediapipe-deep-dive)
5. [Community and Ecosystem Size](#5-community-and-ecosystem-size)
6. [Current Trends and Future Directions](#6-current-trends-and-future-directions)
7. [Linux/Wayland Considerations](#7-linuxwayland-considerations)
8. [Recommendations for a University Project](#8-recommendations-for-a-university-project)

---

## 1. Overview of Approaches

The dork search across 376 results identified **7 distinct software-only approaches** for building a virtual mouse from hand gestures using only a laptop webcam. Excluding hardware-dependent solutions (Leap Motion, Tap Strap, ESP32/Arduino, depth cameras, Google Soli), the viable software-only approaches are:

| # | Approach | Era | Core Technique |
|---|----------|-----|----------------|
| 1 | **MediaPipe Hands + PyAutoGUI** | 2020--present | Pre-trained 21-landmark hand model + OS mouse injection |
| 2 | **cvzone wrapper** | 2021--present | Higher-level wrapper around MediaPipe |
| 3 | **CONDENSATION (Particle Filter)** | 1998--2010s | Probabilistic contour tracking via factored sampling |
| 4 | **CNN/YOLO custom training** | 2017--present | Train a neural network to classify hand gestures |
| 5 | **Browser-based (Handtrack.js)** | 2019--present | TensorFlow.js hand detection in the browser |
| 6 | **Color/marker detection (Pre-ML)** | 2004--2015 | HSV color segmentation or colored tape tracking |
| 7 | **UCL MotionInput** | 2021--present | Research-grade modular touchless computing framework |

---

## 2. Detailed Analysis of Each Approach

### 2.1 MediaPipe Hands + PyAutoGUI (The Dominant Stack)

**What it is:** Google's MediaPipe provides a pre-trained ML pipeline that detects 21 3D hand-knuckle coordinates from a standard RGB webcam in real time. A Python script reads these landmarks and uses PyAutoGUI or pynput to inject OS-level mouse events.

**Technical architecture:**
- Two-stage ML pipeline: (1) a palm detection single-shot detector model, (2) a hand landmark regression model producing 21 3D points
- Palm detection achieves 95.7% average precision (source: [MediaPipe Hands docs](https://github.com/google-ai-edge/mediapipe/blob/master/docs/solutions/hands.md))
- Trained on ~30,000 real-world images plus synthetic renders
- Latency: 17.12ms CPU / 12.27ms GPU on Pixel 6 (source: [Google AI Edge Hand Landmarker](https://developers.google.com/edge/mediapipe/solutions/vision/hand_landmarker))
- Platforms: Python, Android, iOS, Web (JavaScript)

**How gesture-to-mouse mapping works:**
- Landmark #8 (index fingertip) position maps to cursor coordinates via `numpy.interp()` scaling from normalized [0,1] to screen resolution
- Landmark #4 (thumb tip) proximity to #8 triggers left click
- Various finger combinations map to right click, scroll, drag
- Exponential smoothing (averaging current + previous positions) reduces jitter
- Source: ["I Ditched My Mouse" (Towards Data Science)](https://towardsdatascience.com/i-ditched-my-mouse-how-i-control-my-computer-with-hand-gestures-in-60-lines-of-python/)

**Implementation size:** 35--60 lines of Python for a minimal working version; 200--500 lines for full feature set (move, left/right click, drag, scroll).

**Academic validation:**
- A 2026 IJCOPE study with 25 participants reported **96% gesture recognition accuracy** after brief familiarization (source: [IJCOPE 2026 PDF](https://ijcope.org/wp-content/uploads/2026/05/Real-Time-Hand-Gesture-Recognition-for-Touchless-Computer-Interaction-A-MediaPipe-Based-Virtual-Mouse-Implementation.pdf))
- A 2025 IJRASET paper measured **92% tracking accuracy at ~25 fps** on standard webcams (source: [IJRASET](https://www.ijraset.com/research-paper/real-time-hand-gesture-recognition-for-markerless-virtual-mouse-control))
- An IEEE 2024 paper documented advanced controls including finger air writing alongside virtual mouse (source: [IEEE 10463446](https://ieeexplore.ieee.org/document/10463446/))

**GitHub ecosystem (verified stats, 2026-09-21):**
- MediaPipe repo: **37,027 stars**, 6,166 forks, last pushed 2026-09-18 ([google-ai-edge/mediapipe](https://github.com/google-ai-edge/mediapipe))
- PyAutoGUI: **12,703 stars**, 1,435 forks ([asweigart/pyautogui](https://github.com/asweigart/pyautogui))
- Top virtual mouse repo: Viral-Doshi/Gesture-Controlled-Virtual-Mouse -- **862 stars**, 255 forks ([link](https://github.com/Viral-Doshi/Gesture-Controlled-Virtual-Mouse))
- PyPI downloads: **~2 million/month** for the `mediapipe` package (source: [PyPI Stats](https://pypistats.org/packages/mediapipe))

**Pros:**
- Fastest path from zero to working demo (hours, not days)
- Massive tutorial ecosystem: 30+ ready repos, 10+ blog tutorials, video guides
- No training required -- model is pre-trained
- Runs on CPU at real-time speeds on any modern laptop
- Strong academic paper base for citations
- Active maintenance (last MediaPipe push: 2026-09-18)

**Cons:**
- Hand must be visible and reasonably well-lit
- Limited precision vs. physical mouse (jitter, coordinate quantization)
- Landmark accuracy drops under occlusion or unusual hand poses
- PyAutoGUI does not natively support Wayland (Linux-specific concern)
- Academic depth is limited: you are mostly *using* a pre-trained model, not building one

---

### 2.2 cvzone Wrapper (Simplified MediaPipe)

**What it is:** A Python package wrapping MediaPipe and OpenCV into higher-level abstractions. Instead of writing raw MediaPipe detection/drawing code, you call `HandDetector()` and get finger positions in 2--3 lines.

**Verified stats (2026-09-21):**
- cvzone repo: **1,329 stars**, 282 forks, last pushed 2024-05-10 ([cvzone/cvzone](https://github.com/cvzone/cvzone))
- Companion course: [Computer Vision Zone Hand Tracking](https://www.computervision.zone/courses/hand-tracking/)
- Video tutorial: [YouTube cvzone virtual mouse](https://www.youtube.com/watch?v=OA8mwoDTXtw)

**Pros:**
- Even less boilerplate than raw MediaPipe
- Good beginner on-ramp

**Cons:**
- Extra dependency layer; masks what is happening underneath
- Less actively maintained (last push May 2024)
- Lower academic value -- you are wrapping a wrapper
- Not recommended for a university project that needs to demonstrate understanding

---

### 2.3 CONDENSATION Algorithm (Classical Particle Filter)

**What it is:** CONditional DENSity fropagATION, published by Michael Isard and Andrew Blake in 1998 in the International Journal of Computer Vision (Vol. 29, No. 1, pp. 5--28). It is a particle-filter-based probabilistic tracker that maintains a weighted set of samples ("particles") representing hypotheses about an object's state, and propagates them forward using a motion model plus observation likelihood.

**Core technical details:**
- Uses factored sampling (a Monte Carlo method) to estimate the conditional probability density p(x_t | z_1,...,z_t)
- Overcomes Kalman filtering's fundamental limitation: inability to handle multi-modal probability distributions caused by cluttered backgrounds
- Supports non-linear motion models, including mixed discrete/continuous trackers with auto-regressive process switching
- Demonstrated real-time tracking at 30Hz on 1998 hardware
- Source: [Isard & Blake 1998 (PDF via Stanford)](http://vision.stanford.edu/teaching/cs231b_spring1415/papers/isard-blake-98.pdf), [Oxford abstract](https://www.robots.ox.ac.uk/~ab/abstracts/ijcv98.html)

**For hand tracking specifically:**
- Tracks hand *contours* (outlines), not landmarks -- fundamentally different from MediaPipe's approach
- Can handle fast/agile motion, cluttered backgrounds, and re-initialization when a hand leaves and re-enters the scene
- Demonstrated tracking hand movements including finger flexing and hand drawing motions
- Source: [Edinburgh CONDENSATION home page](https://homepages.inf.ed.ac.uk/rbf/CVonline/LOCAL_COPIES/ISARD1/condensation.html)

**Relevant academic papers:**
- "Hand Contour Tracking Using Condensation and Partitioned Sampling" (source: [ResearchGate](https://www.researchgate.net/publication/221247285_Hand_Contour_Tracking_Using_Condensation_and_Partitioned_Sampling), [Springer](https://link.springer.com/chapter/10.1007/978-3-540-69736-7_37))
- "Virtual Mouse -- Inputting Device by Hand Gesture Tracking and Recognition" using Condensation + active shape model (source: [Springer](https://link.springer.com/chapter/10.1007/3-540-40063-X_12))
- "Real-time Two Hand Gesture Recognition with Condensation + HMMs" (source: [IEEE](https://ieeexplore.ieee.org/document/8369811/))

**Python implementation resources:**
- `cesarorosco/condensationPy`: CONDENSATION in Python based on original OpenCV, **2 stars** ([link](https://github.com/cesarorosco/condensationPy))
- `johnhw/pfilter`: Generic Python particle filter, **181 stars** ([link](https://github.com/johnhw/pfilter))
- `rlabbe/filterpy`: Kalman/particle filter library, **3,875 stars** ([link](https://github.com/rlabbe/filterpy))
- SciPy Cookbook: Basic particle filter tutorial (source: [SciPy Cookbook](https://scipy-cookbook.readthedocs.io/items/ParticleFilter.html))
- "Particle Filter Made Simple" (2025 arXiv tutorial): [arXiv 2511.01281](https://arxiv.org/pdf/2511.01281)

**Pros:**
- Strongest academic/theoretical depth -- you implement the algorithm from scratch or near-scratch
- Excellent for demonstrating understanding of probabilistic inference, Bayesian filtering, Monte Carlo methods
- The foundational paper (Isard & Blake 1998) is a landmark in computer vision with thousands of citations
- Handles fast motion and clutter through multi-hypothesis tracking
- No dependency on proprietary pre-trained models

**Cons:**
- Significantly more complex to implement (weeks vs. hours)
- No turnkey repos -- you must build the hand-tracking pipeline yourself
- Contour-based tracking is less precise than 21-landmark detection
- Tiny community: the one Python CONDENSATION repo has 2 stars
- The approach is pre-deep-learning (1998) and largely superseded in practice
- Requires strong mathematical background (probability theory, sampling methods)
- Getting from "tracking a contour" to "controlling the mouse" requires additional engineering

---

### 2.4 CNN / YOLO Custom Training

**What it is:** Training a convolutional neural network or YOLO object detector to classify hand gestures directly from images, then mapping classified gestures to mouse actions.

**Key resources:**
- "Hand Gesture Recognition based Virtual Mouse using CNN" (source: [SciSpace PDF](https://scispace.com/pdf/hand-gesture-recognition-based-virtual-mouse-using-cnn-3mh0egcg.pdf))
- YOLOv8 hand gesture recognition on OAK-D (source: [PyImageSearch](https://pyimagesearch.com/2023/05/15/hand-gesture-recognition-with-yolov8-on-oak-d-in-near-real-time/))
- Hand gesture recognition with TensorFlow + OpenCV (source: [TechVidvan](https://techvidvan.com/tutorials/hand-gesture-recognition-tensorflow-opencv/))

**Accuracy note:** One study found YOLOv7 hand detection at 99.8% accuracy vs. MediaPipe at 57.6% in a specific clinical application (source: [saiwa.ai comparison](https://saiwa.ai/blog/openpose-vs-mediapipe/)), though this comparison is domain-specific and not representative of general gesture recognition.

**Pros:**
- Can achieve very high accuracy for specific, well-defined gesture sets
- Demonstrates ML training skills (data collection, augmentation, model architecture)
- Customizable to arbitrary gesture vocabularies

**Cons:**
- Requires collecting and labeling training data (hundreds to thousands of images)
- Training takes time and compute (though fine-tuning on small datasets is feasible)
- Harder to get started: no single turnkey pipeline
- For the specific use case of "fingertip position to cursor," MediaPipe's landmark approach is more natural than classification
- Higher code complexity than MediaPipe approach

---

### 2.5 Browser-Based (Handtrack.js / TensorFlow.js)

**What it is:** Running hand detection entirely in the browser using TensorFlow.js. Handtrack.js is the most prominent library, detecting 7 hand poses (open, closed, pinch, point, face, pointtip, pinchtip) via a pre-trained object detection neural network.

**Verified stats (2026-09-21):**
- handtrack.js: **2,930 stars**, 257 forks, last pushed 2022-07-20 ([victordibia/handtrack.js](https://github.com/victordibia/handtrack.js))
- Source: [TensorFlow Blog 2019](https://blog.tensorflow.org/2019/11/handtrackjs-tracking-hand-interactions.html)

**Pros:**
- Zero installation for end users -- runs in any modern browser
- Cross-platform by nature
- Good for web demos and prototypes

**Cons:**
- **Cannot control the OS mouse** -- browser sandbox prevents system-level mouse injection
- Lower performance than native Python + MediaPipe (JavaScript overhead, no GPU compute path on most laptops)
- Library not actively maintained (last push 2022)
- Only detects bounding boxes, not 21 landmarks -- less precise positioning
- Not suitable for a "virtual mouse" project that needs to actually move the system cursor

**Verdict:** Not viable for the stated goal of controlling the system mouse. Useful only for in-browser gesture demos.

---

### 2.6 Color/Marker Detection (Pre-ML Approaches)

**What it is:** Using HSV color space segmentation, contour analysis, or physical colored markers (tape, caps) to track hand/finger positions without any machine learning.

**Key implementations:**
- Color detection + segmentation: [urvi367/Virtual-Mouse](https://github.com/urvi367/Virtual-Mouse)
- Colored tape tracking: "Mouse Simulation Using Two Coloured Tapes" ([arXiv 1204.1277](https://arxiv.org/pdf/1204.1277))
- Color-based cursor control: "Mouse Control using Web Camera based on Colour Detection" ([arXiv 1403.4722](https://arxiv.org/pdf/1403.4722))
- Contour analysis: "Thumbs Up: Hand Gesture Recognition" ([PyImageSearch](https://pyimagesearch.com/2015/02/05/thumbs-hand-gesture-recognition/))
- HOG+SVM hand detector with dlib: ([LearnOpenCV](https://learnopencv.com/training-a-custom-object-detector-with-dlib-making-gesture-controlled-applications/))

**Pros:**
- Extremely lightweight -- no ML dependencies
- Educational value for understanding basic CV (color spaces, morphological operations, contour detection)
- Runs on anything

**Cons:**
- Extremely fragile: sensitive to lighting, background color, skin tone
- Colored markers are cumbersome
- Very low accuracy and reliability compared to ML approaches
- Obsolete as a practical approach

---

### 2.7 UCL MotionInput (Research-Grade Framework)

**What it is:** A modular, research-grade touchless computing framework developed by UCL in collaboration with Intel, Microsoft, IBM, and the NHS. Supports gesture, voice, facial expression, and eye-gaze input. Published on the Microsoft Store.

**Key details:**
- Version 3.4 released May 2024 with 24 new AI tools
- Supports DirectX game control, gaze-based cursor, finger air writing
- Spin-out company MotionInput Games established 2023
- Sources: [UCL MotionInput v3](https://www.ucl.ac.uk/engineering/computer-science/collaborate/ucl-industry-exchange-network-ucl-ixn/touchless-computing-ucl-motioninput-3), [arXiv 2108.04357](https://arxiv.org/pdf/2108.04357)

**Pros:**
- Most feature-complete solution
- Strong academic backing (UCL + industry partners)

**Cons:**
- Windows-only (Microsoft Store distribution)
- Very large/complex system -- not suitable for building as a student project
- Better as a reference than as a starting point
- Using it as-is would not demonstrate the student's own implementation work

---

## 3. Comparison Matrix

| Dimension | MediaPipe + PyAutoGUI | CONDENSATION | CNN/YOLO Custom | Handtrack.js | Color Detection | cvzone |
|---|---|---|---|---|---|---|
| **Setup time** | Hours | Weeks | Days--weeks | Hours | Hours | Hours |
| **Lines of code (minimal)** | 35--60 | 300--1000+ | 200--500+ | 20--50 (browser only) | 50--150 | 20--40 |
| **Accuracy** | ~92--96% | Medium (contour-level) | Up to 99%+ (narrow tasks) | Medium (bbox only) | Low--Medium | ~92--96% |
| **Latency** | <20ms per frame | ~33ms (30Hz historical) | 20--100ms depending on model | 50--200ms (browser) | <10ms | <20ms per frame |
| **Academic depth** | Low--Medium (use pre-trained model) | **Very High** (implement algorithm) | High (train/evaluate model) | Low | Low--Medium | Very Low |
| **Community size** | **Very Large** (37K stars, 2M downloads/mo) | Tiny (2 stars for Python impl) | Medium (scattered) | Small (2.9K stars, unmaintained) | Tiny | Medium (1.3K stars) |
| **Modernity** | Current (2020--2026) | Historical (1998) | Current | Dated (2019, stale) | Obsolete | Current but wrapper |
| **OS mouse control** | Yes | Yes (with work) | Yes (with work) | **No** (browser sandbox) | Yes | Yes |
| **Requires training data** | No | No | **Yes** | No | No | No |
| **Runs on CPU** | Yes | Yes | Depends on model | Yes (browser) | Yes | Yes |
| **Linux support** | Yes (X11; Wayland needs workaround) | Yes | Yes | Yes (browser) | Yes | Yes |

---

## 4. CONDENSATION vs MediaPipe: Deep Dive

This is the central comparison for a university project that wants to balance academic merit with practical feasibility.

### 4.1 Philosophical Difference

| Aspect | CONDENSATION | MediaPipe |
|--------|-------------|-----------|
| **Era** | Classical computer vision (1998) | Modern deep learning (2020) |
| **Core technique** | Probabilistic particle filter with factored sampling | Pre-trained CNN-based landmark regression |
| **What it tracks** | Object contours (outlines, shapes) | 21 specific anatomical landmarks |
| **How it handles uncertainty** | Explicit probability distributions over state space | Implicit (model confidence scores) |
| **Motion model** | Programmer-defined (auto-regressive processes) | Learned from training data |
| **Mathematical foundation** | Bayesian inference, Monte Carlo sampling, importance sampling | Gradient descent, backpropagation (hidden inside pre-trained model) |

### 4.2 Academic Merit

**CONDENSATION wins on:**
- Depth of algorithmic understanding required
- Mathematical rigor (probability theory, sampling, state estimation)
- Forces the student to implement core logic, not just call an API
- The original paper (Isard & Blake 1998, IJCV) is a landmark in computer vision
- Demonstrates classical CS fundamentals valued in academic settings
- You can write a literature review spanning 25+ years of particle filter research

**MediaPipe wins on:**
- Massive recent citation base (dozens of 2024--2026 papers)
- Can cite accuracy studies with concrete numbers (96% in user study)
- Easier to produce quantitative results for a report
- Can extend the base system in interesting ways (adding gestures, voice, smoothing algorithms)
- More publishable as a practical contribution

### 4.3 Implementation Effort

**CONDENSATION path (estimated 4--8 weeks):**
1. Study particle filter theory (1 week)
2. Implement basic particle filter in Python (1 week) -- using filterpy or from scratch
3. Implement hand contour extraction via OpenCV (edge detection, skin segmentation) (1 week)
4. Integrate CONDENSATION tracking with hand contour (1--2 weeks)
5. Map tracked contour center/extremities to mouse position (1 week)
6. Add gesture recognition on top (optional, 1--2 weeks)

**MediaPipe path (estimated 1--2 weeks):**
1. Install dependencies: `pip install mediapipe opencv-python pyautogui numpy` (10 minutes)
2. Follow the "60 lines of Python" tutorial (1--2 hours)
3. Add click/scroll/drag gesture mappings (1--2 days)
4. Implement smoothing, dead zones, gesture state machine (2--3 days)
5. Test, tune, document (2--3 days)
6. Add extensions: voice commands, custom gestures, GUI overlay (optional, 1 week)

### 4.4 Recommended Hybrid Approach

For maximum academic value with practical results, consider:

1. **Implement both** -- MediaPipe as the "modern baseline" and CONDENSATION as the "classical comparison"
2. **Benchmark them side-by-side** -- same webcam, same gestures, measure accuracy, latency, robustness
3. **Write the paper comparing them** -- this is a genuine research contribution

This gives you:
- A working demo (MediaPipe, done in week 1)
- Academic depth (CONDENSATION, done by week 4--6)
- A comparative evaluation (the actual contribution of the paper)

---

## 5. Community and Ecosystem Size

### 5.1 By GitHub Stars (verified 2026-09-21)

| Project | Stars | Forks | Last Push |
|---------|-------|-------|-----------|
| google-ai-edge/mediapipe | 37,027 | 6,166 | 2026-09-18 |
| asweigart/pyautogui | 12,703 | 1,435 | 2024-08-20 |
| rlabbe/filterpy | 3,875 | 678 | 2024-02-07 |
| victordibia/handtrack.js | 2,930 | 257 | 2022-07-20 |
| cvzone/cvzone | 1,329 | 282 | 2024-05-10 |
| Viral-Doshi/Gesture-Controlled-Virtual-Mouse | 862 | 255 | 2023-11-13 |
| takeyamayuki/NonMouse | 200 | 24 | 2026-08-27 |
| johnhw/pfilter (particle filter) | 181 | 25 | 2023-04-03 |
| OTAKUWeBer/Wayland-automation | 44 | 7 | 2026-08-18 |
| cesarorosco/condensationPy | 2 | 7 | 2023-02-21 |

### 5.2 By PyPI Downloads

- **mediapipe**: ~2,004,523 downloads/month (source: [PyPI Stats](https://pypistats.org/packages/mediapipe))
- Latest release: v1.0.1, April 27, 2026

### 5.3 By Tutorial Availability

- **MediaPipe virtual mouse**: 10+ distinct blog tutorials, 5+ YouTube videos, 30+ GitHub repos with READMEs
- **CONDENSATION hand tracking**: 0 dedicated tutorials found; must learn from the original paper + generic particle filter tutorials
- **CNN/YOLO hand gestures**: 3--5 tutorials (PyImageSearch, TechVidvan)
- **Color detection mouse**: 2--3 tutorials (mostly outdated)

### 5.4 By Academic Paper Count

- **MediaPipe-based virtual mouse**: 20+ papers found (2022--2026), spanning IEEE, Springer, IJCOPE, IRJMETS, JETIR, SciTePress, and others
- **CONDENSATION + hand tracking**: 5--8 papers found (1998--2018), in IJCV, IEEE, Springer, ACM
- **CNN/YOLO hand gesture mouse**: 5--10 papers found (2021--2025)

---

## 6. Current Trends and Future Directions

### 6.1 Transformer-Based Gesture Recognition (2024--2026)

The hottest research trend is combining MediaPipe's hand landmarks with transformer architectures for gesture *classification* (not just tracking):

- **MediaPipe + Transformer** achieved >99% accuracy on gesture classification (source: [MDPI 2025](https://www.mdpi.com/2673-4591/108/1/22))
- **GestFormer** (CVPR 2024): multiscale wavelet pooling transformer for dynamic gestures
- **ConvMixFormer** (WACV 2025): resource-efficient transformer for hand gesture recognition
- Comprehensive survey: "Visual Hand Gesture Recognition with Deep Learning" (2025, [arXiv 2507.04465](https://arxiv.org/pdf/2507.04465))

### 6.2 Commercial Solutions

- **Neural Lab AirTouch** (CES 2025): webcam-only gesture control for Windows/Android/Linux. Supports 9 gestures (up to 15 customizable), plug-and-play under 5 minutes setup. Compared to Apple Vision Pro-like control without the $3,500 headset. (source: [Engadget](https://www.engadget.com/computing/accessories/neural-labs-airtouch-brings-gesture-control-to-windows-and-android-devices-with-just-a-webcam-180031750.html), [Cybernews CES review](https://cybernews.com/tech/neural-lab-hand-gesturing-feature-ces-2025/))

### 6.3 Multimodal Fusion

- **MotionInput v3.4** (UCL, 2024): combines gesture + voice + gaze + facial expression for touchless computing (source: [UCL News](https://www.ucl.ac.uk/news/2024/apr/ai-enhanced-touchless-computing-helps-users-take-control-their-gaming-experience))
- **WatchHand** (arXiv 2025): using standard smartwatch IMU sensors for continuous hand pose tracking (source: [arXiv 2602.21610](https://arxiv.org/pdf/2602.21610))
- Hybrid mouse + hand tracking: **HandOver** (arXiv 2025) for precise 3D object manipulation (source: [arXiv 2508.00211](https://arxiv.org/pdf/2508.00211))

### 6.4 Direction of the Field

The field is moving toward:
1. **Higher-level gesture semantics** -- not just "where is the finger" but "what does the gesture mean"
2. **Transformer architectures** replacing pure CNN pipelines for temporal gesture sequence understanding
3. **On-device deployment** with quantized models (MediaPipe already runs on-device)
4. **Multimodal input fusion** (gesture + gaze + voice)
5. **Accessibility applications** as a primary motivator (post-COVID touchless interfaces)

---

## 7. Linux/Wayland Considerations

Since the project runs on a laptop (likely Linux given the Arch Linux environment detected), Wayland compatibility matters:

- **PyAutoGUI** does NOT work on Wayland -- it relies on X11 protocols
- **pynput** has limited Wayland support
- **Workarounds:**
  - `ydotool`: Wayland-compatible input injection (replaces xdotool)
  - `Wayland-automation` library: Python PyAutoGUI alternative for wlroots compositors (Hyprland, Sway, labwc). 44 stars, actively maintained (last push 2026-08-18). (source: [OTAKUWeBer/Wayland-automation](https://github.com/OTAKUWeBer/Wayland-automation))
  - `nabeelmusthafawork/handTrack`: a MediaPipe virtual mouse with explicit Linux/Wayland support ([link](https://github.com/nabeelmusthafawork/handTrack))
- **xdotool** does not work on Wayland at all (source: [GitLab wiki](https://gitlab.com/nokun/gestures/-/wikis/xdotool-on-wayland))
- Running under X11 or XWayland avoids all these issues

---

## 8. Recommendations for a University Project

### 8.1 Easiest to Implement

**MediaPipe Hands + PyAutoGUI** -- unambiguously. A working prototype can be built in 35--60 lines of Python in under 2 hours. The "I Ditched My Mouse" tutorial provides a copy-paste starting point.

### 8.2 Newest / Most Modern

**MediaPipe + Transformer hybrid** is the cutting edge (2025--2026). However, implementing a transformer from scratch is complex. For practical modernity, **MediaPipe Hands** (continuously updated, last push Sept 2026) is the most modern production-ready approach.

### 8.3 Largest Community

**MediaPipe** by a landslide: 37K GitHub stars, 2M PyPI downloads/month, 20+ recent academic papers, 30+ tutorial repos. No other approach comes close.

### 8.4 Best Academic Depth

**CONDENSATION** (particle filter) for classical CS depth. **CNN/YOLO custom training** for ML depth. MediaPipe is weak here -- calling `mp.solutions.hands.Hands()` does not demonstrate deep understanding.

### 8.5 Best Overall for University Project

**Recommended approach: MediaPipe as primary + CONDENSATION as comparison study**

The optimal strategy depends on what the university values:

#### Option A: "Build something that works well" (Demo-focused)
- **Stack:** Python + MediaPipe Hands + PyAutoGUI (or Wayland-automation on Wayland) + OpenCV + NumPy
- **Time:** 1--2 weeks
- **Deliverable:** Working virtual mouse with move, click, scroll, drag + GUI overlay showing landmarks
- **Extensions:** Add voice commands, custom gesture vocabulary, smoothing comparison study
- **Academic angle:** Cite the IJCOPE 2026 accuracy study, replicate their methodology, report your own accuracy numbers

#### Option B: "Demonstrate deep algorithmic understanding" (Theory-focused)
- **Stack:** Python + OpenCV + filterpy (or hand-written particle filter) + PyAutoGUI
- **Time:** 4--8 weeks
- **Deliverable:** CONDENSATION-based hand contour tracker controlling the mouse
- **Academic angle:** Implement and explain the Isard & Blake 1998 algorithm, compare with MediaPipe, discuss particle filter theory
- **Risk:** Higher chance of partial/non-working result due to complexity

#### Option C: "Best of both worlds" (Recommended for maximum grade)
- **Phase 1 (Week 1--2):** Build the MediaPipe virtual mouse as a working baseline
- **Phase 2 (Week 3--6):** Implement a CONDENSATION-based tracker
- **Phase 3 (Week 7--8):** Run comparative benchmarks (accuracy, latency, robustness to occlusion/lighting)
- **Deliverable:** A paper/report comparing classical (1998) vs. modern (2026) hand tracking for virtual mouse control
- **Academic angle:** This is a genuine comparative study with original experimental results

### 8.6 Recommended Technology Stack

```
Core:
  - Python 3.10+
  - OpenCV (cv2) -- webcam capture and image processing
  - MediaPipe 1.0.x -- hand landmark detection
  - NumPy -- coordinate math and interpolation

Mouse injection (pick based on desktop environment):
  - PyAutoGUI -- if running X11
  - Wayland-automation -- if running Wayland (Hyprland/Sway)
  - pynput -- cross-platform alternative (partial Wayland support)

For CONDENSATION comparison:
  - filterpy -- particle filter implementation
  - scipy -- for statistical operations
  - matplotlib -- for particle visualization

Optional extensions:
  - pyttsx3 or gTTS -- voice feedback
  - tkinter or PyQt -- GUI overlay
  - pytest -- testing
```

### 8.7 Key References to Cite

1. **Isard, M. & Blake, A.** (1998). "CONDENSATION -- Conditional Density Propagation for Visual Tracking." *International Journal of Computer Vision*, 29(1), 5--28. [PDF](http://vision.stanford.edu/teaching/cs231b_spring1415/papers/isard-blake-98.pdf)
2. **Zhang, F. et al.** (2020). "MediaPipe Hands: On-device Real-time Hand Tracking." *arXiv preprint*. [ResearchGate](https://www.researchgate.net/publication/342302340_MediaPipe_Hands_On-device_Real-time_Hand_Tracking)
3. **IJCOPE 2026 study** -- "Real-Time Hand Gesture Recognition for Touchless Computer Interaction: A MediaPipe-Based Virtual Mouse Implementation." [PDF](https://ijcope.org/wp-content/uploads/2026/05/Real-Time-Hand-Gesture-Recognition-for-Touchless-Computer-Interaction-A-MediaPipe-Based-Virtual-Mouse-Implementation.pdf)
4. **Elfring, J. et al.** -- "Particle Filters: A Hands-On Tutorial." [PDF](https://pdfs.semanticscholar.org/fff9/cad8a37e55669bf695772fbdcf728b75fa37.pdf)
5. **arXiv 2511.01281** (2025). "Particle Filter Made Simple: A Step-by-Step Beginner-friendly Guide."
6. **Chuang, C. et al.** -- "Hand Contour Tracking Using Condensation and Partitioned Sampling." [Springer](https://link.springer.com/chapter/10.1007/978-3-540-69736-7_37)

---

## Source Index

All URLs cited in this report were accessed and verified on 2026-09-21. GitHub star/fork counts and PyPI download statistics were retrieved via the GitHub API and PyPI Stats on the same date.
