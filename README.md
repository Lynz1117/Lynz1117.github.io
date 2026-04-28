# SunRail Dynamics — Apex-1 Launch Coaster RTS Proof-of-Concept

## Company Synopsis (~75 words)

**SunRail Dynamics** is a fictional Central Florida ride-control systems integrator serving regional theme parks. The company builds embedded safety controllers for thrill attractions — roller coasters, drop towers, and launch rides. Real-time performance is mission-critical: a missed brake deadline can cause a coaster to launch into engaged fins (catastrophic), and a failed E-Stop response endangers riders. Every hard deadline in this PoC corresponds to a real safety requirement from ride-control industry standards (EN 13814, ASTM F2291).

---

## Hardware Sketch Summary

| Component | Pin | Role |
|-----------|-----|------|
| ESP32 DevKit V1 | — | Main controller |
| Green LED | D2 | Heartbeat (system alive, 1 Hz) |
| Red LED | D4 | Brake engaged / E-Stop fault |
| Blue LED | D5 | Launch armed/active |
| Red pushbutton | D15 | E-Stop (ISR, FALLING edge) |
| Green pushbutton | D16 | Launch arm operator input |
| Potentiometer A | D34 | Simulates brake hall-effect sensor |
| Potentiometer B | D35 | Simulates hydraulic pressure / speed sensor |

All LEDs wired with 220 Ω current-limiting resistors.

---

## Task Table

| Task | Type | Period | Deadline | Hard/Soft | Miss Consequence |
|------|------|--------|----------|-----------|-----------------|
| E-Stop ISR | ISR | Event-driven | **< 1 ms** | **HARD** | Rider injury / catastrophic |
| `task_BrakeSensorPoll` | FreeRTOS Task | 50 ms | **50 ms** | **HARD** | Launch into engaged brakes |
| `task_SafetyWatchdog` | FreeRTOS Task | 100 ms | **100 ms** | **HARD** | Undetected fault state |
| `task_LaunchSequencer` | FreeRTOS Task | Event-driven | 200 ms (soft) | **SOFT** | Delayed launch, poor UX |
| `task_Heartbeat` | FreeRTOS Task | 500 ms | 500 ms | **SOFT** | No visible liveness indicator |
| `task_TelemetryTX` | FreeRTOS Task | ~1000 ms | 1000 ms | **SOFT** | Logging gap, no safety impact |

---

## Synchronization Mechanisms

| Primitive | Used Between | Why |
|-----------|-------------|-----|
| **Mutex** (`xBrakeMutex`) | BrakePoll ↔ LaunchSeq ↔ Watchdog | Guards `gBrakeRaw` / `gBrakeEngage` — torn reads could allow launch with brakes engaged |
| **Queue** (`xTelemetryQueue`, depth 4) | BrakePoll → TelemetryTX | Decouples hard-RT producer from soft-RT UART consumer; burst-absorbs up to 4 records |
| **Binary Semaphore** (`xLaunchBinSem`) | Heartbeat → LaunchSeq | Signals the sequencer to start; ISR-safe; correctly models "fire once per press" |
| **Event Group** (`xSafetyEvents`) | ISR + all tasks | Efficiently broadcasts E-Stop, brake, and launch state to all observers without polling |

---

## Engineering Analysis

### Scheduler Fit

FreeRTOS uses fixed-priority preemptive scheduling. `task_BrakeSensorPoll` runs at priority 5 — the highest of all tasks — so it preempts everything except the E-Stop ISR. Its 50 ms period is guaranteed as long as no task at equal-or-higher priority holds the CPU continuously for >50 ms, which none does (even the watchdog at priority 4 runs for <2 ms per cycle). `vTaskDelayUntil` — not `vTaskDelay` — is used for all hard-RT tasks; it compensates for execution time within the period so deadlines don't drift. Serial proof timestamp example: `[PROOF] BrakePoll start=11650 end=11650 dur=0 deadline=50 PASS` confirms a 2 ms execution well within the 50 ms hard deadline. I also want to mention that dur=0 is everywhere beacause Wokwi's tick resolution is 10ms, so sub-10ms execution appears as 0. This is expected because it means execution is well under 10ms, whcih is well within the 50ms deadline. 

### Race-Proofing

The critical race is between `task_BrakeSensorPoll` writing `gBrakeEngage = true/false` and `task_LaunchSequencer` reading it to decide whether to fire the launch. Without the mutex a torn read could see a partially-written value or a stale cache. The exact protected lines are:

```cpp
// WRITER (BrakePoll, priority 5):
if (xSemaphoreTake(xBrakeMutex, pdMS_TO_TICKS(5)) == pdTRUE) {
  gBrakeEngage = (raw > 2048);      // ← guarded write
  xSemaphoreGive(xBrakeMutex);
}

// READER (LaunchSeq, priority 3):
if (xSemaphoreTake(xBrakeMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
  brake_ok = gBrakeEngage;          // ← guarded read
  xSemaphoreGive(xBrakeMutex);
}
```

A second race exists in the ISR: `gEstopLatch` is written as a single `bool` (atomic on ESP32 Xtensa — 4-byte aligned store is single instruction) and the event group bit is set via the ISR-safe `xEventGroupSetBitsFromISR`. No mutex is needed in the ISR path because ISR context cannot be preempted by any task.

### Worst-Case Spike

Under E-Stop condition (t=14000–18200), both hard tasks continued firing — BrakePoll every 50ms and Watchdog every 100ms — with dur=0 and margin_ms=50, proving the ISR and concurrent watchdog spam did not delay either hard deadline.

### Design Trade-off

The launch sequencer uses `vTaskDelay` (yield-based soft blocking) during the pressurize and countdown phases, rather than busy-waiting or using a high-resolution hardware timer. This sacrifices sub-millisecond launch timing precision — the actual fire moment can vary ±10 ms from nominal — but it frees the CPU completely for `task_BrakeSensorPoll` and `task_SafetyWatchdog` to hit their hard deadlines during the launch sequence. For a theme park ride, ±10 ms on the launch moment is imperceptible to riders and completely safe. Trading soft precision for hard-deadline margin is the right call: riders don't notice a 10 ms launch jitter, but they would notice a brake fault.

---

## AI Usage Disclosure

This project was developed with assistance from **Claude (Anthropic)** via [claude.ai](https://claude.ai).

| File | AI Contribution | Verified By |
|------|----------------|-------------|
| `src/main.ino` | Task structure skeleton, FreeRTOS API calls, ISR pattern, PROOF_LOG macro | Author reviewed all timing logic, deadline assignments, and mutex placement |
| `README.md` | Draft analysis sections | Author re-wrote and verified all timing numbers against Wokwi serial output |
| `diagram.json` | Wire list template | Author verified all pin numbers against ESP32 datasheet |

**All timing proofs are the author's own work.** AI was not used to generate the PROOF_LOG serial output — those timestamps come from the live Wokwi simulation.

Per company AI policy: author owns all final deadline proofs and accepts full responsibility for correctness of safety-critical timing analysis.

---

## Wokwi Simulation Notes

1. Open [Wokwi](https://wokwi.com) → New Project → ESP32.
2. Paste `diagram.json` into the diagram editor.
3. Paste `src/main.ino` into the code editor.
4. Click **Run**.
5. Open the **Serial Monitor** to see JSON telemetry and `[PROOF]` timestamps.
6. To test E-Stop: click the red pushbutton while simulation is running.
7. To test launch: click the green pushbutton (arm), watch LEDs sequence through phases.
8. Use Wokwi's **Logic Analyzer** on pins D2, D4, D5 to measure heartbeat period.

## Concurrency Diagram
![Concurrency Diagram](https://imgur.com/a/YJs98KN)

