# fanctrl.yaml -- Implementation Notes

Alternative firmware for the ECB board. Derived from `ecb1.yaml` (Ver 3.1).

## Summary

- **Shared control loop.** All four fans share a single target temperature and sensor source instead of per-fan targets/sources. This simplifies configuration for the common case where all fans serve the same enclosure.
- **Safety override system.** On any alarm condition every fan is forced to 100 % regardless of auto/manual mode. Three trigger types:
  - *Overtemp* -- any sensor above the configurable alarm threshold for 10 s (5 consecutive ticks).
  - *Sensor failure* -- the selected PI source returns NaN, or all sensors return NaN simultaneously.
  - *Fan stall* -- any fan reporting 0 RPM while commanded above 50 % speed for 30 s (15 ticks).
  Overtemp and sensor-fail alarms clear automatically with hysteresis (default 3 C below threshold). Stall alarms persist until manual reset via the Safety Reset button.
- **Slew-rate limiter.** Fan speed changes are capped at 3 % per 2 s tick to reduce acoustic noise from abrupt speed transitions.
- **Cold shutoff.** When the measured temperature is more than 2 C below target, auto-mode fans are turned off entirely instead of running at minimum speed.
- **Alarm LED.** A pulsing LED on GPIO2 activates during any safety override and a binary sensor publishes the alarm state to Home Assistant.
- **NaN-safe sensor reads.** Uses `raw_state` to detect NaN immediately even when `filter_out: NaN` is active, preventing the PI controller from acting on stale data.
- **API instead of MQTT.** Uses the ESPHome native API with encryption; MQTT is removed. Web server v3 is enabled for local access.

## Control Loop

The control loop runs as a 2-second interval with a 2-second startup delay.

### ecb1.yaml (original)

Four independent PI controllers, one per fan. Each fan has its own target temperature number input and sensor source selector (8 options: AHT20 T/H, 4x Dallas, SHT40 T/H). The PI calculation per fan:

```
error = actual - target
integral += error                          (clamped to +/-100)
speed = error * 20 + integral * 0.1        (clamped to 1..100)
```

If speed < 10 the fan is turned off, otherwise turned on at the calculated speed. No NaN checking is performed on sensor values. Overtemp alarm uses a `for: 10s` condition on the raw sensor states and only pulses the alarm LED -- fans are not forced to maximum.

### fanctrl.yaml (this file)

Single shared PI controller for all four auto-mode fans. One target temperature, one sensor source selector (Dallas, ESP32 internal, AHT20).

```
error = actual - target
if 0 < raw_speed < 100: integral += error  (clamped to +/-350)
raw_speed = error * 20 + integral * 0.03   (clamped to 0..100)
```

The integral is only accumulated when the output is not saturated (anti-windup). After the PI calculation, a slew-rate limiter restricts changes to 3 % per tick. Fans whose temperature is more than 2 C below target are shut off. During any safety override, PI output is ignored and all fans run at 100 %.

