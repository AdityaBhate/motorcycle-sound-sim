# 349 Twin — Instrument Pods

A single-file, no-dependency simulation of a motorcycle tach + speedo, with a from-scratch engine physics model and a from-scratch synthesized exhaust note (no samples). Built as a hobby project to see how far "just math" can get you toward something that feels like a real bike.

Open `index.html` in a browser. Everything — physics, audio DSP, rendering — is vanilla JS + Canvas 2D + Web Audio.

## Controls
`W` throttle · `Q` part throttle · `Shift` clutch · `N`/`M` shift up/down · `E` start · `T` tweak panel · `A` mute

## Tech stack
- Vanilla HTML/CSS/JS, single file, zero build step, zero dependencies
- Canvas 2D for the tach/speedo rendering
- Web Audio API, engine sound runs in an **AudioWorklet** (own thread, sample-accurate) with a ScriptProcessor fallback
- Fixed-timestep physics loop (1/240 s) decoupled from rendering (`requestAnimationFrame`)

## Engine model (the math)
- **Displacement → bore/stroke**: solved from displacement, cylinder count, and bore/stroke ratio (`bore = ∛(Vcyl·4·ratio/π)`)
- **Torque from mean effective pressure**: `T = (IMEP − FMEP) · Vd / (4π)` — standard 4-stroke MEP-to-torque conversion
- **IMEP** follows a breathing curve (volumetric efficiency vs. rpm/redline) looked up from a table
- **FMEP** (friction) scales with mean piston speed: `FMEP = 0.60 + 0.055·mps + 0.12·(cyl − 2)`
- **Pumping losses** scale with throttle position (manifold vacuum at closed throttle)
- **Redline** derived from a target max piston speed: `rpm = mps·30/stroke`
- **Crank/vehicle coupling**: two inertias joined by a friction clutch (Karnopp stick-slip model) — `J·dω/dt = T_engine − T_clutch`, `m·dv/dt = T_clutch·G·η/r − F_aero − F_roll`
- **Aero/rolling resistance**: standard drag `F = ½·ρ·CdA·v²` and rolling resistance `F = Crr·m·g`
- **Terminal velocity / gearing**: solved by bisection from peak power vs. drag, then a geometric 6-speed gear ladder built off that
- **Idle**: PI controller holding target rpm via a virtual idle-air valve
- **Needle motion**: second-order damped system (`ẍ = ωₙ²(target − x) − 2ζωₙẋ`) for realistic tach/speedo swing

## Sound engineering (the audio)
No recorded samples — the exhaust note is synthesized from a physical model:
- **Digital waveguide** exhaust: header → link pipe → silencer chambers → tailpipe, modeled as delay-line segments with area-change reflection coefficients at each junction, lossy-wall lowpass filtering, and an open-end radiation filter
- **Per-cylinder gas pulses** shaped in crank-angle space (not time), so pulse width naturally shortens with rpm
- **N-port collector** junction (pressure-scattering) where multiple cylinder primaries merge
- **RBJ biquad bandpass filters** for intake resonance, jet/turbulence noise, and mechanical (valve/gear) noise
- **Afterfire/overrun pop** model: probabilistic ignition of unburnt charge in the hot pipe on trailing throttle
- **Stereo room**: Schroeder reverb (4 comb filters + 2 allpass filters per channel)
- Peak limiter, DC blocking, and soft `tanh` clipping on the output stage

## Tunable parameters
Cylinder count, displacement, bore/stroke ratio, peak IMEP, max piston speed, crank angle (firing order), exhaust type, listening position, and clutch release time are all live-adjustable from the tweak panel (`T`) — the whole engine/drivetrain/audio model rebuilds from these inputs.
