# Changelog

All notable changes to RadarSim will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.4.0] - 2026-05-11 (SAR/ISAR Imaging & AI Tactical Director)

### Added - Phase 30: Imaging Radar & Adversarial AI

#### SAR/ISAR Imaging (`src/advanced/sar_isar.py`)
- **rda_vectorized()**: Batch FFT Range-Doppler Algorithm — 5-stage pipeline with vectorized range compression, RCMC, and azimuth matched filter
- **ISARProcessor**: Inverse SAR with translational motion compensation (cross-correlation alignment) and cross-range FFT imaging
- **SARImageResult**: Container with image_db, axes, and verified resolution metadata
- Resolution verified: `Δr = c/(2B) = 1.5m`, `Δa = D/2 = 0.5m`

#### AI Tactical Director (`src/advanced/ai_director.py`) [NEW]
- **AIDirector**: Red Force agent that analyzes Blue Force radar coverage
- **Coverage Map**: 2D Pd grid based on R⁴ range equation
- **Blind Zone Detection**: Flood-fill connected component analysis
- **Attack Planning**: 3 difficulty levels (Easy/Medium/Hard)
- **Low-Pd Routing**: Greedy corridor navigation minimizing exposure
- **Jammer Deployment**: Optimal DRFM positioning at blind zone edges

### Validation
- **25 new tests** (`tests/test_phase30.py`): SAR resolution, RDA, ISAR, coverage, blind zones, attack plans, performance
- **Performance**: RDA 1024×512 < 2s, ISAR 64×256 < 1s, Coverage 100×100 < 500ms
- **217/217 total tests passed** — zero regressions

### References
- Cumming & Wong, "Digital Processing of SAR Data", 2005
- Chen & Ling, "Time-Frequency Transforms for Radar Imaging", 2002

## [2.3.0] - 2026-05-11 (Multi-Radar Network Fusion)

### Added - Phase 29: Distributed Sensor Network & Covariance Intersection

#### Network Manager (`src/simulation/network_manager.py`) [NEW]
- **NetworkManager**: Multi-radar orchestrator with node registration, track exchange, and CI fusion
- **CovarianceIntersection**: Julier & Uhlmann (1997) non-divergent fusion — guarantees `tr(P_fused) ≤ min(tr(Pᵢ))`
- **StrobeTriangulator**: Least Squares AOA bearing intersection for jammer localization
- **TrackAssociator**: Track-to-Track Association (T2TA) with Euclidean gating
- **LatencyModel**: Configurable FIFO delay queue (Link-16 / JTIDS simulation)
- **GDOP**: Geometric Dilution of Precision computation

#### Sensor Fusion Integration (`src/advanced/sensor_fusion.py`)
- Re-exports CI and Triangulator from network_manager for backward compatibility

#### 3D Tactical View (`src/ui/tactical_3d.py`)
- **Gold fused tracks**: CI-fused tracks rendered as gold scatter symbols
- **Fusion gain ellipses**: Covariance ellipses show GDOP-driven uncertainty reduction
- **Jammer positions**: Triangulated jammer locations as magenta diamonds

### Validation
- **29 new tests** (`tests/test_network_fusion.py`): CI, triangulation, GDOP, latency, T2TA, integration, performance
- **Performance**: 100 targets × 5 radars fused in < 5s
- **192/192 total tests passed** — zero regressions

### References
- Julier, S. & Uhlmann, J. "A Non-divergent Estimation Algorithm in the Presence of Unknown Correlations", ACC, 1997
- Poisel, R. "Electronic Warfare Target Location Methods", Artech House, 2012
- Blackman, S. "Multiple-Target Tracking with Radar Applications", 1986

## [2.2.0] - 2026-05-11 (Electronic Warfare — EA/ECCM)

### Added - Phase 28: Electronic Attack & Counter-Countermeasures

#### Electronic Attack (EA)
- **DRFMJammer** class (`src/physics/ecm.py`): Digital RF Memory jammer with state machine (IDLE→CAPTURE→PULL→RELEASE)
  - **RGPO** (Range Gate Pull-Off): Phase-coherent CPI injection with progressive range offset
  - **VGPO** (Velocity Gate Pull-Off): Doppler modulation mode
  - `inject_into_cpi()`: Adds coherent false returns to Pulse-Doppler CPI data
  - Configurable: pull rate, max pull distance, capture dwell, J/S gain

#### Electronic Counter-Countermeasures (ECCM)
- **FrequencyAgility** (`src/advanced/eccm.py`): N-hop carrier frequency randomization
  - J/S reduction: `10·log₁₀(N)` dB (10 hops → 10 dB reduction)
  - Hop set generation across configurable bandwidth
- **PRFStagger** (`src/advanced/eccm.py`): PRI jitter ±δ% to defeat RGPO
  - RGPO discrimination via cross-PRI correlation analysis
  - Staggered PRI sequence for ambiguity resolution
- **ECCMController**: Unified orchestrator for all ECCM techniques

#### EKF Hardening
- **Coast Mode** (`src/tracking/ekf.py`): SJNR-aware measurement freeze
  - `update_with_jsr()`: Freezes measurement updates when SJNR < 6 dB
  - Covariance inflation during coast (2%/scan)
  - `should_drop_track`: Auto-drop after max_coast_scans exceeded
  - SJNR = SNR_dB − 10·log₁₀(1 + 10^(JSR/10))

#### UI/UX
- **Advanced → Electronic Warfare** submenu: Jamming, Freq Agility, PRF Stagger toggles
- **Symbology** (`src/ui/symbology.py`): Pulsing magenta "JAM" indicator on jammed targets
- **A-Scope** (`src/ui/a_scope.py`): Orange noise strobe overlay when jamming active
- Status bar feedback for all EW/ECCM state changes

### Validation
- **37 new tests** (`tests/test_eccm.py`): DRFM, frequency agility, PRF stagger, EKF coast, SJNR, burn-through
- **163/163 total tests passed** — zero regressions

### References
- Schleher, "Electronic Warfare in the Information Age", Artech House, 1999
- Skolnik, "Radar Handbook", 3rd Ed., Ch. 24

## [2.1.0] - 2026-05-11 (Extended Kalman Filter)

### Added - Phase 27: EKF Polar Tracking
- **`ExtendedKalmanFilter`** — Nonlinear tracking with polar [r, θ] measurements
  - Analytically derived Jacobian: H = [x/r, y/r, 0, 0; -y/r², x/r², 0, 0]
  - Joseph form covariance update for numerical stability
  - Angle wrapping for circular statistics (innovation ∈ [-π, π])
  - Singularity guard: r_safe = max(r, 1.0m)
- **Adaptive R from SNR** — Dynamic measurement noise scaling:
  - High SNR (>20 dB): R × 0.5 (trust measurement)
  - Low SNR (<10 dB): R × 5.0 (trust prediction)
- **Polar Tracker** — `TrackManager.update_polar()` for direct [r, θ] processing
- **Uncertainty Ellipse** — P-matrix eigendecomposition for 95% confidence
  visualization on 3D tactical map
- **GUI Toggle** — 🎯 "Use Extended Kalman Filter (Polar)" in Advanced menu
- **28 new validation tests** — Jacobian verification, 2G turn tracking,
  angle wrapping, adaptive R, ellipse geometry, matrix inversion

### References
- Bar-Shalom, Y. "Estimation with Applications to Tracking", 2001, Ch. 5.3
- Developed by Mehmet Gümüş (github.com/SpaceEngineerSS)

## [2.0.0] - 2026-05-11 (Pulse-Doppler Engine)

### Added - Phase 26: Signal-Level Pulse-Doppler Processing
- **`PulseDopplerProcessor`** — Complete coherent signal chain:
  CPI Generation → Matched Filter → MTI Canceller → Doppler FFT → R-D Map
- **CPI Generation** — Vectorized complex baseband signal model:
  s(t,n) = Σ A·p(t-2R/c)·exp(j·4π·v·n·T_PRI/λ) with AWGN
- **MTI Canceller** — 2-pulse and 3-pulse implementations with >60 dB DC null
- **Doppler FFT** — Hamming-windowed FFT for -42.5 dB sidelobe suppression
- **`AntennaPattern`** — Sinc², Gaussian, and Taylor-weighted beam models
  with two-way gain computation for monostatic radar
- **`RangeDopplerMap`** — Container with range/velocity axes, blind speeds,
  and processing metadata for UI consumption
- **Dual-mode R-D Scope** — [PULSE-DOPPLER] displays actual FFT output;
  [SYNTHETIC] falls back to legacy Gaussian blob mode
- **Blind speed markers** — v_blind = λ·PRF/2 visualized on R-D map
- **MTI notch indicators** — Shaded rejection band near zero Doppler
- **GUI controls** — Pulse-Doppler toggle + MTI enable in Advanced menu
- **21 new validation tests** covering matched filter gain, MTI null depth,
  target localization, blind speed, antenna patterns, and reproducibility

### Fixed - Phase 26 Bug Hardening
- `engine.py` L291: Removed duplicate `self.clutter_enabled = False`
- `sensor_fusion.py` L464: Fixed `np.softmax` → `scipy.special.softmax`

### References
- Richards, M.A. "Fundamentals of Radar Signal Processing", 2nd Ed., 2005
- Skolnik, M.I. "Radar Handbook", 3rd Ed., Ch. 3-5, 9
- Developed by Mehmet Gümüş (github.com/SpaceEngineerSS)

## [1.0.0] - 2025-12-23 (Gold Release)

### Added - Phase 19: Clutter, MTI & ECCM
- **Environmental Clutter** (Ground/Sea/Rain) with SNR degradation
- **MTI Filter** with configurable velocity threshold
- **ECCM Frequency Agility** to counter jamming

### Added - Phase 20: Precision Tracking
- **Monopulse Angle Estimation** (Sum/Difference patterns)
- **Ambiguity Analysis Widget** (PRF vs Range/Velocity trade-off)
- Sub-beamwidth angular accuracy calculation

### Added - Phase 21: Statistical Analysis
- **ROC Curves** (Pd vs Pfa for Swerling models)
- **SNR Histogram** (Detection strength distribution)
- **Metrics Module** (Albersheim equation, Swerling Pd)

### Added - Phase 22: System Capabilities
- **Scenario Export** (Save simulation state to YAML)
- **Keyboard Shortcuts** (Space, R, 1-4, F11)
- **Performance Monitor** overlay widget

### Added - Phase 23: Signal Visualization
- **B-Scope ECM Strobes** (Jammer direction visualization)
- **A-Scope CFAR Hover** (CUT/Guard/Reference cell display)

### Added - Phase 24: Documentation
- Complete README.md rewrite with v1.0 feature list
- Technical docs: physics_engine.md, signal_processing.md, user_guide.md
- Automated screenshot capture (10 images)

### Previous Features (Phase 15-18)
- **Terrain Masking** with 4/3 Earth refraction
- **RHI Scope** (Range-Height Indicator)
- **3D Tactical Map** with OpenGL
- **SAR Viewer** with realistic imagery
- **Advanced Menu** (Clutter, MTI, ECCM, Monopulse toggles)
- **MIL-STD-2525D Symbology**

### Changed
- Main window now uses QTabWidget for display areas
- PPI/B-Scope use affiliation-based coloring



## [2.0.0] - 2025-12-22

### Added
- **3D Coordinate System**: Full 3D position, velocity, and acceleration support
- **ITU-R P.676 Atmospheric Model**: Oxygen and water vapor absorption (1-100 GHz)
- **Swerling RCS Models**: Implementation of Swerling I, II, III, IV fluctuation models
- **Extended Kalman Filter**: Nonlinear tracking with polar measurements (range, azimuth, elevation, range_rate)
- **Track Lifecycle Management**: TENTATIVE → CONFIRMED → COASTING → DROPPED status
- **SimulationConfig Class**: Centralized configuration management
- **SimulationStatistics**: Comprehensive statistics tracking
- **Probability of Detection**: Albersheim approximation with Swerling correction
- **Scientific Documentation**: Complete formula documentation with IEEE references
- **Open Source Files**: LICENSE (MIT), CONTRIBUTING.md, CHANGELOG.md

### Changed
- **radar_physics.py**: Complete rewrite with 3D support and scientific enhancements
- **target_tracking.py**: EKF implementation replacing simple Kalman filter
- **main.py**: Modern simulation loop with configuration and logging
- **README.md**: Professional documentation with formulas and references

### Fixed
- Atmospheric attenuation now uses scientifically accurate ITU-R model
- Monopulse angle error includes SNR-dependent thermal noise
- Track association uses proper Mahalanobis distance gating

### References
- Skolnik, "Radar Handbook", 3rd Ed., 2008
- Bar-Shalom, "Estimation with Applications to Tracking", 2001
- ITU-R P.676-12, 2017

## [1.0.0] - 2025-12-23

### Added
- Initial release
- Basic radar equation implementation
- 2D target tracking with Kalman filter
- ECM simulation (chaff, decoy, jamming)
- Pygame visualization
- Proportional Navigation guidance

---

## Version Numbering

- **MAJOR**: Incompatible API changes or fundamental algorithm changes
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, backward compatible
