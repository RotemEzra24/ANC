Hybrid Active Noise Cancellation (ANC) System for Generators

Final year project — B.Sc. Electrical Engineering, Afeka Tel Aviv Academic College of Engineering.

Author: Rotem Ezra 
Supervisor: Mr. Yaron Knobel 
Repository: https://github.com/RotemEzra24/ANC
⸻
Overview

A hybrid ANC system that reduces low-frequency noise (50–500 Hz) emitted by generators 
and internal combustion engines.

The core design principle is a strict separation between the signal path and the control 
plane: the entire cancellation path is analog, so its latency does not depend on clock 
rate, CPU load or OS scheduling. The microcontroller never touches the audio — it only 
monitors signal envelopes and adjusts gain.

A metal enclosure is placed over the generator. Four reference microphones sit inside (one 
per panel), four exciters are bonded to the panels from the outside, and a single error 
microphone outside the enclosure closes the control loop.

Signal chain (per channel)

MAX4466 mic -> C_IN (1uF) -> R_IN (10k) -> TL074 (180 deg inversion + band-pass)
            -> C_OUT (1uF) -> MCP41010 (digital gain) -> C_W (1uF)
            -> TPA3116D2 (Class-D) -> exciter on enclosure panel

⸻
Key code sections

All firmware lives in ESP32_CODE. 
The sections below explain the parts that implement the core engineering ideas described in 
the project book.

1. Calibration constants

All tuning parameters sit together at the top of the file: growth threshold, minimum 
envelope, minimum protected gain, cooldown period and smoothing factor. These values are 
installation-dependent — they were recalibrated after the ADC divider was added, because the 
divider halves every measured value.

2. setWiper() — SPI write to the digital potentiometer

Writes a gain code to one MCP41010: command byte 0x11, then the value, wrapped in a 
chip-select pulse. SCK and MOSI are shared across all four devices; only the CS line 
distinguishes them, which is what lets a single ESP32 drive four channels with six GPIO pins 
instead of twelve. MISO is unused — the MCP41010 is write-only.

3. measureEnvelope() — signal level measurement

Peak-to-peak measurement over a 50 ms window. The window length is deliberate: at the lowest 
working frequency (50 Hz, 20 ms period) it contains at least 2.5 full cycles, which is what 
makes the peak reading trustworthy. A shorter window can miss the wave crest entirely and 
return a false level.

4. Exponential smoothing

E_smooth = 0.3 * E_raw + 0.7 * E_smooth

Raw ADC readings fluctuate between 224 and 389 even at zero gain. Without smoothing, those 
random fluctuations produce growth ratios above 1.7 between consecutive samples and trigger 
false alarms — which is exactly what happened in the first version of the protection 
algorithm.

5. Feedback protection algorithm — the central contribution

Exponential growth in signal level is the unique signature of a runaway acoustic feedback 
loop; steady generator noise never produces it. The detector fires only when all three 
conditions hold at once:
Condition	Value	Why
Current gain	> 15	Below this there is not enough loop gain to sustain feedback
Previous smoothed envelope	> threshold	Prevents computing ratios on background noise
Growth ratio	> 2.5	The level above which growth is a loop, not an acoustic event

On trigger the gain drops by 40 steps and the target follows it, so the soft ramp cannot 
climb straight back into the same condition. Monitoring then pauses for a 2-second cooldown.

Validated result: a real runaway was caught at a growth ratio of 3.54 — the envelope 
jumped from the 224–389 background band to 2821. Gain was cut from 64 to 24 within a single 
sampling window, and the envelope settled back to 271–347 with no further alarms.

6. Serial buffer flush

Fixes a real bug found during integration: parseInt() leaves the CR/LF in the input buffer, 
which is then read as a 0 on the next loop iteration and silently zeroes the gain after 
every command entered by the operator.

7. Soft gain ramp

One step per 100 ms toward the target; a full 0-to-255 sweep takes about 25.5 seconds. A 
sudden jump to high gain would start a feedback loop faster than the protection can sample 
it, so ramping guarantees the detector gets several windows to see the trend first. It also 
prevents an audible pop in the exciters on every gain change.
⸻
Hardware
Component	Part	Qty	Role
Op-amp	TL074CN (DIP-14)	1	Four channels in one package
Controller	ESP32 DevKit	1	Supervisory control only
Microphone	MAX4466	5	4 reference + 1 error
Digital pot	MCP41010 (10k, SPI)	4	Gain control
Power amp	TPA3116D2 / HW-576	4	Class-D, mono, 24 V
Buck converter	LM2596S	1	24 V to 5 V for microphones
Supply	24 V / 6 A DC	1	~3.75 A total draw

Key design values

- Gain: -R_F / R_IN = -1 — 180 degree inversion at unity gain
- Band-pass: 15.9 Hz – 1061 Hz, covering the 50–500 Hz generator band
- VBIAS = 12 V (Vcc/2), shared by all four non-inverting inputs
- VBIAS_LO = 1.65 V, biasing P0B so the signal stays inside the MCP41010's legal range

Pin mapping (ESP32)

All ADC channels use ADC1 only — ADC2 is unavailable while Wi-Fi is active, and using it 
would tie the protection mechanism to the radio state.
Channel	ADC pin	CS pin
1	GPIO34	GPIO5
2	GPIO35	GPIO17
3	GPIO32	GPIO16
4	GPIO33	GPIO4
Error mic	GPIO36	—

Shared across all four potentiometers: GPIO18 (SCK), GPIO23 (MOSI).
⸻
Build and flash

Requires Arduino IDE 2.x with the Espressif ESP32 board package.

1. Board: ESP32 Dev Module
2. Upload speed: 115200 — higher rates failed to detect the flash chip on this board
3. Open the Serial Monitor at 115200 and enter a target gain (0–255)

If upload stalls at "Connecting...", hold the BOOT button until writing begins, and 
disconnect the SPI wires during upload — they share pins used at boot.
⸻
Known limitations

- Phase inversion is a fixed 180 degrees, while the physical path adds frequency-dependent 
- phase shift. Simulated deviation reaches 15 degrees at 50 Hz and 23 degrees at 500 Hz, 
- which bounds the achievable cancellation depth at the band edges.
- Reference microphones measure inside the enclosure while the goal is the noise outside.
- A single error microphone defines one quiet point, not a quiet field.
- Protection thresholds are installation-dependent and need recalibration after any hardware 
- change.

Future work

- All-pass stage with a second digital potentiometer for frequency-dependent phase tuning
- CD4051 analog multiplexer to allow one error microphone per panel
- Migration from perfboard to PCB
- Acoustic characterisation of each panel with its exciter
