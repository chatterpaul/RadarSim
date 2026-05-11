## 🛰️ RadarSim v2.4.0: The Cognitive Imaging Update

We are incredibly excited to announce the release of **RadarSim v2.4.0**. This milestone completes **Phase 30** of the RadarSim roadmap and brings massive performance improvements alongside state-of-the-art radar simulation capabilities.

### 🚀 What's New

- **Lightning-Fast Sensor Fusion:** The newly overhauled `NetworkManager` is now capable of fusing 100 simultaneous tracks across a 5-node radar network with a blistering average latency of **7.79ms** using Covariance Intersection.
- **High-Fidelity Imaging Radar (SAR/ISAR):**
  - Implemented the vectorized **Range-Doppler Algorithm (RDA)** for Synthetic Aperture Radar (SAR) image reconstruction. Achieves 1.5m range and 0.5m cross-range resolution.
  - Added **ISAR Processing** with translational motion compensation for dynamic target profiling.
- **AI Tactical Director (Red Force Agent):**
  - An intelligent adversary agent that performs live radar network coverage analysis and blind zone detection.
  - Features Low-Pd corridor routing and autonomous DRFM jammer deployment to stress-test your EW operators.
- **C++ Standalone Execution:**
  - Migrated build system to **Nuitka**. RadarSim now compiles to optimized native C++ binaries across Windows, Linux, and macOS for maximum performance and security.
- **Scientific Foundation Hardened:**
  - Validated Pulse-Doppler and CPI implementation against *Richards (2005)*.
  - Aligned the AI cognitive routing logic with concepts from *Haykin (2006)*.

### 📸 Visuals

**SAR Image Reconstruction**
<img src="https://raw.githubusercontent.com/SpaceEngineerSS/RadarSim/main/docs/images/sar_viewer.png" width="600">

**3D Tactical View & Sensor Fusion**
<img src="https://raw.githubusercontent.com/SpaceEngineerSS/RadarSim/main/docs/images/3d_tactical.png" width="600">

**A-Scope (CFAR) & Jamming Strobe Analysis**
<img src="https://raw.githubusercontent.com/SpaceEngineerSS/RadarSim/main/docs/images/a_scope_cfar.png" width="600">

---

*Developed by Mehmet Gümüş (@SpaceEngineerSS) - RadarSim v2.x*
