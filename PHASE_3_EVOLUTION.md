# Phase 3 Evolution: Signal Processing, Tracking & Electronic Warfare

RadarSim has evolved from a parametric radar simulator into a signal-level
pulse-Doppler platform with nonlinear tracking and electronic warfare capabilities.

> **Developed by Mehmet Gümüş** — [github.com/SpaceEngineerSS](https://github.com/SpaceEngineerSS)

---

## Phase 26: Pulse-Doppler Processing Engine

### Signal Chain Architecture

```
CPI Generation → Matched Filter → MTI Canceller → Doppler FFT → R-D Map
```

### Key Equations

| Component | Formula | Reference |
|-----------|---------|-----------|
| **Signal Model** | s(t,n) = Σ A·p(t−2R/c)·exp(j·4π·v·n·T_PRI/λ) | Richards (2005), Eq. 3.6 |
| **MF Gain** | G_mf = 10·log₁₀(B·T) dB | Richards (2005), Eq. 4.6 |
| **MTI 2-Pulse** | \|H(f)\| = 2\|sin(πf/PRF)\| | Richards (2005), Ch. 3.4 |
| **MTI 3-Pulse** | \|H(f)\| = 4·sin²(πf/PRF) | Richards (2005), Ch. 3.4 |
| **Blind Speed** | v_blind = λ·PRF/2 | Richards (2005), Eq. 3.16 |
| **Doppler Res** | Δv = λ·PRF/(2·N) | Richards (2005), Ch. 4 |

### Implementation Details

- **CPI Generator**: Vectorized NumPy — 64×1024 complex data in ~2ms
- **Range Compression**: Pre-compressed sinc responses (parametric hybrid)
- **MTI**: `np.diff` along slow-time axis (2-pulse, 3-pulse)
- **Doppler FFT**: Hamming window → FFT → fftshift
- **Validation**: 21/21 tests passed

---

## Phase 27: Extended Kalman Filter (Polar Tracking)

### Mathematical Derivation

**State Vector**: x = [x, y, vx, vy]^T (Cartesian, 4×1)

**Measurement Model**: h(x) = [r, θ]^T (polar, 2×1)

```
r = √(x² + y²)
θ = atan2(y, x)
```

**Jacobian H = ∂h/∂x** (2×4):

```
H = | x/r    y/r    0    0 |
    | -y/r²  x/r²   0    0 |
```

**Partial Derivatives**:
- ∂r/∂x = x/√(x²+y²) = x/r
- ∂r/∂y = y/√(x²+y²) = y/r
- ∂θ/∂x = -y/(x²+y²) = -y/r²
- ∂θ/∂y = x/(x²+y²) = x/r²
- ∂r/∂vx = ∂r/∂vy = ∂θ/∂vx = ∂θ/∂vy = 0

### EKF Update Equations

1. **Predict**: x̂⁻ = F·x̂, P⁻ = F·P·F^T + Q
2. **Innovation**: ỹ = z − h(x̂⁻) (with angle wrapping to [-π, π])
3. **Jacobian**: H evaluated at x̂⁻
4. **Innovation Covariance**: S = H·P⁻·H^T + R
5. **Kalman Gain**: K = P⁻·H^T·S⁻¹
6. **State Update**: x̂ = x̂⁻ + K·ỹ
7. **Covariance**: P = (I−K·H)·P⁻·(I−K·H)^T + K·R·K^T (Joseph form)

### Adaptive Measurement Noise

R matrix scales based on SNR from Pulse-Doppler engine:

```
scale = 10^((20 - SNR_dB) / 20)

SNR > 20 dB → scale ≈ 0.3  → Trust measurement
SNR ≈ 20 dB → scale = 1.0  → Balanced
SNR < 10 dB → scale ≈ 3.2  → Trust prediction
SNR < 0 dB  → scale ≈ 10   → Heavily trust prediction
```

### Singularity Protection

- **Range = 0**: Guard with ε = 1.0m to prevent division by zero
- **Angle wrap**: Innovation θ mapped to [-π, π]
- **Joseph form**: Prevents P from becoming non-positive-definite
- **Divergence detection**: NIS > 25 (5σ for 2 DOF χ²)

### Validation: 28/28 Tests Passed

| Test | Description | Status |
|------|-------------|--------|
| Jacobian (5 states) | Analytical vs numerical (< 1e-4) | ✅ |
| Velocity columns | H[:, 2:4] == 0 | ✅ |
| Singularity guard | No inf/nan at origin | ✅ |
| Straight-line tracking | RMS < 100m | ✅ |
| Covariance shrink | tr(P) decreases | ✅ |
| 2G turn (EKF) | RMS < 500m | ✅ |
| 2G turn (EKF vs LKF) | Both < 500m | ✅ |
| Angle wrapping (7 cases) | All in [-π, π] | ✅ |
| Adaptive R (4 tests) | Monotonic, disabled mode | ✅ |
| Uncertainty ellipse (3) | Shape, size, geometry | ✅ |
| 2×2 inverse | vs np.linalg.inv, roundtrip | ✅ |

---

## Phase 28: Electronic Warfare (EA/ECCM)

### DRFM Jammer — Range Gate Pull-Off (RGPO)

**State Machine**:

```
IDLE  ──activate()──→  CAPTURE  ──dwell_s──→  PULL  ──max_pull──→  RELEASE  ──→  IDLE
```

**RGPO Physics**:
1. Jammer receives radar pulse, stores in DRFM memory
2. Retransmits with controlled delay Δτ(t)
3. Apparent range: `R_app(t) = R_true + pull_rate · t`
4. Delay per update: `Δτ_step = 2 · pull_rate · dt / c`

**CPI Injection**:
- False return placed at `bin = round(R_false / Δr)` with sinc point-spread
- Phase-coherent Doppler: `φ_n = 4π · v_false · n · T_PRI / λ`
- Amplitude scaled by J/S gain: `A_jam = A_skin · 10^(G_js/20)`

### Frequency Agility (ECCM)

Hop carrier frequency across N discrete frequencies per CPI:

```
J/S_agile = J/S_static − 10·log₁₀(N_hops)
```

| N_hops | J/S Reduction |
|--------|---------------|
| 2      | 3.0 dB        |
| 10     | 10.0 dB       |
| 20     | 13.0 dB       |
| 100    | 20.0 dB       |

### PRF Stagger (ECCM)

Vary PRI to defeat RGPO:

```
PRI_n = T_PRI · (1 + δ · u_n)    where u_n ∈ [-1, 1], δ = jitter%/100
```

**RGPO Discrimination**: Real target maintains consistent range across varied PRIs (correlation ≈ 1.0). RGPO jammer's fixed delay decorrelates with PRI variation (correlation → 0).

### EKF Coast Mode (Jamming Resilience)

Signal-to-Jamming-plus-Noise Ratio:

```
SJNR_dB = SNR_dB − 10·log₁₀(1 + 10^(JSR/10))
```

| SJNR_dB | EKF Action |
|---------|------------|
| ≥ 6 dB  | Normal measurement update |
| < 6 dB  | COAST — prediction only, P inflated 2%/scan |
| N > max_coast | TRACK DROP |

### Burn-Through Range

Range at which radar overcomes jamming:

```
R_bt = √(Pt · Gt · σ / (Pj · Gj · 4π · SNR_req · Bj/Br))
```

### Validation: 37/37 Tests Passed

| Test Category | Count | Status |
|---------------|-------|--------|
| DRFM state machine | 7 | ✅ |
| CPI injection | 3 | ✅ |
| Frequency agility (4 N values) | 5 | ✅ |
| Frequency variation & hop set | 3 | ✅ |
| PRF stagger | 3 | ✅ |
| RGPO discrimination | 2 | ✅ |
| EKF coast mode (6 scenarios) | 6 | ✅ |
| SJNR calculation | 3 | ✅ |
| ECCM controller | 4 | ✅ |
| Burn-through range | 1 | ✅ |

---

## Phase 29: Multi-Radar Network Fusion

### Covariance Intersection (CI)

Standard Kalman fusion assumes **zero cross-correlation** between sources.
This is WRONG for multi-radar systems and leads to **divergent** estimates.

CI provides consistent fusion regardless of unknown correlations:

```
P_fused = (ω·P₁⁻¹ + (1-ω)·P₂⁻¹)⁻¹
x_fused = P_fused · (ω·P₁⁻¹·x₁ + (1-ω)·P₂⁻¹·x₂)

where ω ∈ [0, 1] minimizes tr(P_fused)
```

**Key Property**: `tr(P_fused) ≤ min(tr(P₁), tr(P₂))` — guaranteed non-divergent.

### Strobe Triangulation (Jammer Localization)

When jamming is active, radars measure AOA but not range. With N ≥ 2 bearings:

```
A·p = b

A = | sin(θ₁)  -cos(θ₁) |    b = | x₁·sin(θ₁) - y₁·cos(θ₁) |
    | sin(θ₂)  -cos(θ₂) |        | x₂·sin(θ₂) - y₂·cos(θ₂) |

p = (A^T·A)⁻¹·A^T·b
```

### GDOP (Geometric Dilution of Precision)

| Geometry | GDOP | Quality |
|----------|------|---------|
| 90° separation | ~1.0 | Ideal |
| 45° separation | ~1.5 | Good |
| Near-collinear | > 5 | Poor |

### Latency Model

| Data Link | Delay | Use Case |
|-----------|-------|----------|
| Direct | 50-200 ms | Co-located systems |
| JTIDS | 1-3 s | Tactical |
| Link-16 | ~12 s | Theater-wide |

### Validation: 29/29 Tests Passed

| Test Category | Count | Status |
|---------------|-------|--------|
| CI trace reduction | 7 | ✅ |
| Multi-estimate CI | 3 | ✅ |
| Strobe triangulation | 5 | ✅ |
| GDOP | 3 | ✅ |
| Latency model | 4 | ✅ |
| Track association | 3 | ✅ |
| NetworkManager integration | 3 | ✅ |
| Performance (100×5) | 1 | ✅ |

---

## Phase 30: SAR/ISAR Imaging & AI Tactical Director

### Vectorized Range-Doppler Algorithm (RDA)

5-stage batch-FFT pipeline:

```
1. Range FFT → multiply conj(chirp) → IFFT     [all pulses at once]
2. Azimuth FFT (Corner Turn implicit)
3. RCMC: ΔR(fd) = λ²·R₀·fd² / (8·v²)          [circular shift per Doppler bin]
4. Azimuth matched filter: exp(j·π·fd²/Ka)
5. Azimuth IFFT → focused SAR image
```

**Resolution equations** (verified by test suite):

| Parameter | Formula | Value (B=100MHz, D=1m) |
|-----------|---------|------------------------|
| Range | Δr = c / (2·B) | 1.50 m |
| Azimuth (stripmap) | Δa = D / 2 | 0.50 m |

### ISAR Processing

For moving targets, the target's own rotation creates the synthetic aperture:

```
Cross-range: Δcr = λ / (2·Δθ)
where Δθ = ω_rot · T_CPI

Pipeline: Range compress → Motion compensate → Cross-range FFT
```

### AI Tactical Director (Red Force Agent)

Coverage analysis → Blind zone detection → Attack planning:

```
Pd(r) = exp(-0.5 · (r / R_det)⁴)    [Swerling-1 approximation]

Blind zone: contiguous cells where Pd < 0.3
Route cost: minimize Σ Pd(cell) along path
```

| Difficulty | Strategy |
|-----------|----------|
| EASY | Random straight-line approach |
| MEDIUM | Low-Pd corridor navigation |
| HARD | Multi-axis attack + DRFM jammer deployment |

### Validation: 25/25 Tests Passed

| Test Category | Count | Status |
|---------------|-------|--------|
| SAR resolution (Δr, Δa) | 5 | ✅ |
| Vectorized RDA | 4 | ✅ |
| ISAR processor | 4 | ✅ |
| AI coverage map | 2 | ✅ |
| AI blind zones | 3 | ✅ |
| AI attack plans | 4 | ✅ |
| Performance benchmarks | 3 | ✅ |

---

## References

1. Richards, M.A. "Fundamentals of Radar Signal Processing", 2nd Ed., McGraw-Hill, 2005
2. Bar-Shalom, Y. "Estimation with Applications to Tracking and Navigation", Wiley, 2001
3. Skolnik, M.I. "Radar Handbook", 3rd Ed., McGraw-Hill, Ch. 3-5, 9, 24
4. Blackman, S. "Multiple-Target Tracking with Radar Applications", Artech House, 1986
5. Schleher, D.C. "Electronic Warfare in the Information Age", Artech House, 1999
6. Van Brunt, L.B. "Applied ECM", Vol. 1-3, EW Engineering, 1978
7. Julier, S. & Uhlmann, J. "A Non-divergent Estimation Algorithm", ACC, 1997
8. Poisel, R. "Electronic Warfare Target Location Methods", Artech House, 2012
9. Cumming, I. & Wong, F. "Digital Processing of SAR Data", Artech House, 2005
10. Chen, V. & Ling, H. "Time-Frequency Transforms for Radar Imaging", Artech House, 2002



