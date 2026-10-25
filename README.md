# Smart-Heater-ASUTP
### Electric heating system with gradual power ramp-up and monitoring capabilities

An autonomous, distributed industrial-grade control system (ICS/ASUTP) for residential electric heating based on **ZEBRA PRO EVO300** heating elements. The project seamlessly integrates a robust hardware relay automation board, custom microcontroller firmware, and intelligent software logic to deliver maximum energy efficiency, fail-safe operation, and hardware redundancy.

---

## 🛠️ Key System Features

*   **Dynamic Topology Shifting (Series ⇄ Parallel):** A hardware relay bridge (relays K1–K6) dynamically toggles the heating element circuits between two states: an eco/soft mode (series connection) for low-power temperature maintenance and a forced/rapid mode (parallel connection) for quick room warm-up.
*   **Tuya Thermostat v1 Integration:** Room climate control is tied to **Tuya Thermostats version 1 (v1)**. High-speed physical data exchange with ESPHome runs over a low-level, local `tuya_uart` component (completely bypassing v3 modifications), ensuring instant responsiveness without any cloud dependency.
*   **Gradual Power Ramp-up:** A step-by-step sequential activation of executive systems (Heating ➔ Water Valves ➔ Sockets ➔ Lighting) utilizing precise time delays after a Home Assistant server boot. This eliminates dangerous inrush currents, power grid surges, and high CPU spikes.
*   **Edge Autonomy (ESPHome):** Internal firmware logic running on ESP8266 modules handles local timeouts and processes physical buttons or thermostats independently, ensuring the heating system remains functional even during a total loss of connection to the Home Assistant server.
*   **Intelligent Anti-Hang Control:** Automatic tracking of "frozen" states in wireless motion and presence sensors using sliding timers and software counters. Stuck sensors are programmatically reset to `off` on-the-fly via custom Python scripts.

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

## 📐 Hardware Specifications

| Component / Configuration File | Executive Mechanism | Commutating Mode | Technological Task |
| :--- | :--- | :--- | :--- |
| **Sonoff 4CH Pro / Tuya WR3** | Heavy-duty NT90TPNCE220CB relays (30/40A) | COM-NO-NC cascade array | Toggling series/parallel topology for heating arrays T1-T4 |
| **Tuya v1 Thermostats** | `tuya_uart` component (ESPHome) | Low-level local UART (v1) | Local climate zones control with hardware fail-safe limits |
| **Sonoff Dual R2 / Pow2** | Industrial NT90 relays | Dry contact / CT current sensor | Power holding loops and continuous energy consumption logging |

---

## 🚀 Deployment & Maintenance

*   **Initial Flashing:** All hardware modules (Sonoff, Tuya, generic ESP8266) are initially flashed "via wire" using a standard USB-to-TTL UART adapter in bootloader mode. Detailed pinout instructions can be found in the official ESPHome documentation.
*   **Firmware Updates:** Once connected to the local network, all future configuration adjustments and firmware upgrades are performed wirelessly over-the-air (**OTA** via the ESPHome Native API).
*   **Security & Secrets:** All sensitive variables, such as Wi-Fi credentials, AP BSSIDs, and OTA passwords, are isolated inside the `secrets.yaml` file and safely ignored by Git version tracking using `.gitignore` rules.

---

## 🔒 Fail-Safe & Reliability Protocols

1.  **Availability Filtering:** All Home Assistant automation routines employ strict entity state checking to filter out `unavailable` and `unknown` statuses, preventing erratic behavior during network transitions or router reboots.
2.  **Short-Circuit Prevention:** The physical wiring topology of the series/parallel relay bridge inherently prevents phase-to-neutral short circuits, even in the event of an unexpected microcontroller logic failure or simultaneous relay engagement.
3.  **Automation Resuscitation Alerts:** Any automated intervention—such as forcing a stuck motion sensor back to `off` after 30 minutes of continuous static holding—triggers an immediate telemetry alert dispatched to the administrator's service channel.
