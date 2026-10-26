# Smart-Heater-ASUTP (ZEBRA PRO EVO300)

### Autonomous Electric Heating System with Dynamic Power Management & Monitoring

An autonomous, distributed industrial-grade control system (ICS/ASUTP) for residential electric heating based on **ZEBRA PRO EVO300** heating elements. The project seamlessly integrates a robust hardware relay automation board, custom microcontroller firmware, and intelligent software logic to deliver maximum energy efficiency, fail-safe operation, and hardware redundancy.

---

## 🛠️ Key System Features

*   **Dynamic Topology Shifting (Series ⇄ Parallel):** A hardware relay bridge dynamically toggles the heating element circuits between two states: an eco/soft mode (series connection) for low-power temperature maintenance and a forced/rapid mode (parallel connection) for quick room warm-up.
*   **Tuya Thermostat v1 Integration:** Room climate control is tied to **Tuya Thermostats version 1 (v1)**. High-speed physical data exchange with ESPHome runs over a low-level, local `tuya_uart` component (completely bypassing v3 modifications), ensuring instant responsiveness without any cloud dependency.
*   **Gradual Power Ramp-up:** A step-by-step sequential activation of executive systems (Heating ➔ Water Valves ➔ Sockets ➔ Lighting) utilizing precise time delays after a Home Assistant server boot. This eliminates dangerous inrush currents, power grid surges, and high CPU spikes.
*   **Edge Autonomy (ESPHome):** Internal firmware logic running on ESP8266 modules handles local timeouts and processes physical buttons or thermostats independently, ensuring the heating system remains functional even during a total loss of connection to the Home Assistant server.

---


### Terminology (Project-Specific)
*   **"Cold State"**: All power relays on the switcher are OFF, the circuit is open, and there is no load on the heaters.
*   **"Relay Pre-Pull"**: The switcher mechanically closes one relay, but **no current flows** to the heaters (the main circuit remains intentionally open). This is a topology preparation step.
*   **"Heartbeat / Pulses"**: One‑second impulses sent by the thermostat. This is a “get ready” signal, not a heating command.
*   **"Trigger Point"**: Thermostat hysteresis activation (temperature drops below the threshold). This is the real heating start command.
*   **"Topology Commutation"**: Switching the heater connection scheme (series ⇄ parallel) via relay closure/opening. Must be completed **before** power is applied.

---

## Main Operation Algorithm (Normal Mode)

1.  **Idle Mode.**
    *   Temperature is within the hysteresis band (e.g., 19–21°C for a 20°C target).
    *   Thermostat is silent.
    *   Switcher holds all relays in the **"cold state"**. Circuit is open. No heating occurs.

2.  **Preparation Phase (“Heartbeat”, ~0.5°C before threshold).**
    *   As temperature approaches the lower hysteresis limit, the thermostat sends short **one‑second pulses**.
    *   The switcher detects the pulses and performs a **"pre‑pull of the first relay"**.
    *   **Critical:** At this stage, **no current is supplied** to the heaters. The circuit remains intentionally open. Purpose: prepare the topology, avoid inrush current and transients.

3.  **Trigger Point (Heating Start).**
    *   Temperature drops below the hysteresis threshold. Thermostat issues the start command.
    *   Switcher executes **topology commutation** (switches the heater connection scheme).
    *   Only after commutation is complete does the **"pre‑pulled" relay** close to allow power flow.
    *   A gradual power ramp‑up algorithm starts: every ~1 minute, the next stage (relay) is added until target power is reached.

---

## Link Loss Scenarios & Fail‑Safe

Switcher behavior depends on the phase when the link to the thermostat is lost.

| Phase When Link Is Lost | Switcher Action | Purpose |
| --- | --- | --- |
| **During Preparation Phase**<br>(pulses present, no start command yet) | Switcher **automatically releases** the "pre‑pulled" relay via internal logic. | Prevent uncontrolled heating. Circuit remains open. |
| **After Trigger Point Activation**<br>(gradual ramp‑up already started) | **No link required.** Switcher completes the ramp‑up procedure autonomously. | Guaranteed transition to operating mode even without communication. |
| **Complete Link Loss**<br>(thermostat not involved) | Thermostat operates autonomously by hysteresis. On trigger activation, heating runs **without gradual start** (direct full power). | Priority: prevent freezing. Smoothness is sacrificed to keep the space heated. |

---



## 📂 Repository Structure

The project follows a clean, modular structure matching your deployment files:

*   `./automationss_ha/` — Home Assistant core automation scripts (Global startup dispatcher, presence zone control, veranda logic).
*   `./components/my_components/tuya_uart/` — Custom low-level ESPHome external components tailored specifically for **Tuya Thermostats v1** over the UART bus.
*   `./device/` / Root — ESPHome `.yaml` configuration files for various hardware node types:
    *   `thermostat-electric-heating.yaml` — Primary climate management module interfacing with Tuya v1 thermostats.
    *   `switch-heater-four-sonoff-4x.yaml` — 4-channel power switching block configured for Sonoff 4CH Pro R3.
    *   `switch-heater-four-tuya-wr3-4x.yaml` — 4-channel power switching block configured for Tuya WR3 chips.
    *   `switch-heater-two.yaml` — 2-channel power switching block configured for Sonoff Dual R2.
    *   `monitor-heater.yaml` — Integration module for continuous power monitoring (Sonoff Pow lines).
*   `./` Root (UI Dashboards) — Pre-configured front-end interface cards (`Lovelace UI`) for managing climate and manual relay overrules (`lovelace_thermostat.yaml`, `lovelace_switch.yaml`).
*   `./` Root (Schematics) — Complete electrical engineering blueprints of the automation control board (`wiring diagrams.drawio` and `wiring diagrams.html` for interactive browser viewing).

---

## 📐 Hardware Specifications & Module Roles

The entire system is powered strictly by the **stationary mains grid**. Battery backup power has been completely removed from the architecture to maximize reliability and eliminate battery degradation risks in unheated environments.

### Operational Roles Matrix
*   **"Thermostat" Module (Tuya v1 + ESPHome):** The logical core of the climate control. It continuously processes temperature sensor data over a low-level local UART (v1) and generates impulse control commands.
*   **"4CH" Module (Commutator - Sonoff 4CH Pro / Tuya WR3):** Features heavy-duty NT90TPNCE220CB relays (30/40A) configured in a COM-NO-NC cascade array. It receives network impulses directly from the Thermostat and physically switches power channels, toggling the series/parallel topology for heating arrays T1-T4.
*   **"Dual R2" Modules (Sonoff Dual R2 / Pow2):** Kept completely independent from the core switching logic. Equipped with industrial NT90 relays and CT current sensors, they are used exclusively for power holding loops, current safety monitoring, and continuous energy consumption logging.

| Component / Configuration File | Executive Mechanism | Commutating Mode | Technological Task |
| :--- | :--- | :--- | :--- |
| **Sonoff 4CH Pro / Tuya WR3** | Heavy-duty NT90TPNCE220CB relays (30/40A) | COM-NO-NC cascade array | Toggling series/parallel topology for heating arrays T1-T4 |
| **Tuya v1 Thermostats** | `tuya_uart` component (ESPHome) | Low-level local UART (v1) | Local climate zones control with hardware fail-safe limits |
| **Sonoff Dual R2 / Pow2** | Industrial NT90 relays | Dry contact / CT current sensor | Power holding loops and continuous energy consumption logging |

---

## 🧠 Inter-Module Interaction & The 0.5°C Trigger

Communication between the **Thermostat** and the **4CH Commutator** is handled directly over the local network via ESPHome Native API / HTTP embedding. This completely bypasses the central Home Assistant server for core operations and eliminates network "ping-pong".

*   **The Trigger Point (0.5°C Advance):** The Thermostat tracks the temperature threshold based on the formula: `Trigger = Target Temperature + 0.5°C`.
*   **Pre-heating Impulse:** As soon as the temperature drops to this 0.5°C buffer zone, the Thermostat fires a direct local network impulse to the 4CH Commutator. The Commutator immediately engages the relays to start pre-heating the ZEBRA PRO panels. By the time the actual target temperature is reached, the heavy heating elements are already at full operating capacity, completely smoothing out thermal inertia.
*   **Network Watchdog (Heartbeat):** The Thermostat sends these impulses periodically. Since there is no battery backup, if the 4CH Commutator stops receiving this heartbeat (due to a local Wi-Fi drop), a local software watchdog trips. The Commutator instantly switches to the hardware Fail-Safe mode to prevent the facility from freezing.

---

## 🔒 Fail-Safe, Topology & Reliability Protocols

1.  **Fail-Safe Relay Topology:** The electrical wiring of the load contacts is designed around a "normally-closed" emergency loop principle. In the event of a total control electronics failure, module hang-up, or network blackout, the Commutator relays naturally fall back to the **Maximum Parallel Heating Mode**. This ensures the facility remains protected from freezing (anti-freeze) under any circumstances.
2.  **Short-Circuit Prevention:** The physical wiring topology of the series/parallel relay bridge inherently prevents phase-to-neutral short circuits, even in the event of an unexpected microcontroller logic failure or simultaneous relay engagement.
3.  **Availability Filtering:** All Home Assistant automation routines employ strict entity state checking to filter out `unavailable` and `unknown` statuses, preventing erratic behavior during network transitions or router reboots.
4.  **HA Protection Automation (Anti-Hang):** To prevent physical actuators from getting stuck in an active state, a strict safety automation is implemented in Home Assistant. It monitors the leading edge of the state transition (`to: active`). Continuous operation is strictly capped at **30 minutes**. If exceeded, the automation overrides the system and kills the power until manual inspection.

---

## 🚀 Deployment & Maintenance

*   **Initial Flashing:** All hardware modules (Sonoff, Tuya, generic ESP8266) are initially flashed "via wire" using a standard USB-to-TTL UART adapter in bootloader mode. Detailed pinout instructions can be found in the official ESPHome documentation.
*   **Firmware Updates:** Once connected to the local network, all future configuration adjustments and firmware upgrades are performed wirelessly over-the-air (**OTA** via the ESPHome Native API).
*   **Security & Secrets:** All sensitive variables, such as Wi-Fi credentials, AP BSSIDs, and OTA passwords, are isolated inside the `secrets.yaml` file and safely ignored by Git version tracking using `.gitignore` rules.
*   **System Environment (Windows Network Bridge):** To ensure reliable access to network shares and ASUTP logs from engineering workstations running Windows, a baseline fix is applied: unauthenticated guest access is enabled via registry (`LanmanWorkstation\AllowInsecureGuestAuth = 1`) combined with an automatic File Explorer cache flush to eliminate network folder freezing within the local automation subnet.
