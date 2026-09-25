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
- **Spec-sheet inputs**: bore, stroke, cylinders, firing intervals, compression ratio, redline, idle, peak-torque rpm, peak-power rpm, volumetric efficiency; displacement is derived (`Vd = n·π/4·B²·S`)
- **IMEP from first principles**: `IMEP = η_i(CR) · VE · ρ_air·LHV/AFR` (≈35 bar of charge energy at 100% VE); `η_i = 0.78 · (1 − CR^−0.3)` (Otto cycle, γ≈1.3, derated for heat loss/finite burn)
- **Breathing curve**: VE rises parabolically to its peak, then drops at a cubic knee; the knee rate and the breathing-peak rpm are fitted by bisection so the *brake* torque and power peaks land on the spec-sheet rpm
- **Torque from mean effective pressure**: `T = (IMEP − FMEP) · Vd / (4π)` — standard 4-stroke MEP-to-torque conversion
- **FMEP** (friction) scales with mean piston speed: `FMEP = 0.60 + 0.055·mps + 0.12·(cyl − 2)`
- **Pumping losses** scale with throttle position (manifold vacuum at closed throttle)
- **Redline** entered directly; limiter at +5%. Mean piston speed is shown (and flagged past ~25 m/s)
- **Crank/vehicle coupling**: two inertias joined by a friction clutch (Karnopp stick-slip model) — `J·dω/dt = T_engine − T_clutch`, `m·dv/dt = T_clutch·G·η/r − F_aero − F_roll`
- **Aero/rolling resistance**: standard drag `F = ½·ρ·CdA·v²` and rolling resistance `F = Crr·m·g`
- **Gearing**: either your gearbox ratios × primary × sprockets, or an auto 4/5/6-speed geometric ladder whose top gear puts peak power at terminal velocity. Rolling radius from the rear tyre size (`150/60-17` style); CdA from body style; mass = wet weight + rider
- **Top speed**: highest speed, in any gear below the limiter, where wheel power still beats aero + rolling drag
- **Single-cylinder engines**: one power stroke per 720°, a 1.5× heavier flywheel, and an intra-cycle crank-speed lope (fast through the power stroke, bleeding off into compression) that gives the thumper its uneven idle
- **Idle**: PI controller holding target rpm via a virtual idle-air valve
- **Needle motion**: second-order damped system (`ẍ = ωₙ²(target − x) − 2ζωₙẋ`) for realistic tach/speedo swing

## Sound engineering (the audio)
No recorded samples — the exhaust note is synthesized from a physical model:
- **Digital waveguide** exhaust: header → link pipe → silencer chambers → tailpipe, modeled as delay-line segments with area-change reflection coefficients at each junction, lossy-wall lowpass filtering, and an open-end radiation filter
- **Per-cylinder gas pulses** shaped in crank-angle space (not time), so pulse width naturally shortens with rpm
- **Pipe network**: primaries (user header length, deliberately unequal per cylinder) → N-into-1 collector, 4-2-1 (cylinders paired ~360° apart by firing order), or a separate system per cylinder → link pipe (user length) → silencer. Every junction is an N-port pressure-scattering node
- **Firing intervals** drive the per-cylinder pulse timing directly, so thumper, 270° / V-twin "potato", T-plane, crossplane and flat-plane screams come from the layout, not from separate sound banks
- **Pulse strength** scales with cylinder volume (√Vcyl) and gently with compression (more expansion before the exhaust valve opens = softer blowdown), and follows the breathing curve (torque dips are audible)
- **Singles**: intra-cycle crank-speed lope; **cranking**: compression strokes drag the starter down
- **Intake**: airbox Helmholtz resonance scaled with displacement (f ∝ 1/√V), or pod filter / velocity stacks (runner quarter-wave, louder)
- **Mechanical**: valvetrain voicing (DOHC tick / SOHC / pushrod tappet clatter) and air-cooled fin ring
- **RBJ biquad bandpass filters** for intake resonance, jet/turbulence noise, and mechanical (valve/gear) noise
- **Afterfire/overrun pop** model: probabilistic ignition of unburnt charge in the hot pipe on trailing throttle
- **Stereo room**: Schroeder reverb (4 comb filters + 2 allpass filters per channel)
- Peak limiter, DC blocking, and soft `tanh` clipping on the output stage

## Tunable parameters
All live in the tweak panel (`T`), grouped as Engine, Exhaust & intake, Sound, Chassis & gearing and Rider. Spec values have a number box for exact entry plus a slider. Sound-only settings (silencer, header/link length, header layout, intake, valvetrain, cooling, listener) retune the audio without resetting the ride.

To simulate a real bike: type in its spec sheet (bore, stroke, compression, cylinders + firing layout, redline, the rpm where torque and power peak), then nudge **VE** until the derived peak torque matches the brochure. Tested against published figures: Bullet 350, Duke 390, Street Triple 765 RS and R1 land within ~2–7% on power and torque at the right rpm.

Settings autosave in the browser. **Copy config / Paste config** move a bike around as JSON, so you can keep your own library. The Single/Twin/Triple/Four buttons are generic 349 cc starting points.
