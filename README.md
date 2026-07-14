
<img width="1526" height="811" alt="Screenshot 2026-07-14 121441" src="https://github.com/user-attachments/assets/723a40ac-2b18-421c-940f-3daf6f0ad367" />


# TECHNICAL DOCUMENTATION REPORT

## Real-Time EV Dashboard and ADAS Warning System

**System Architecture, Firmware Design, and Multi-Threaded Telemetry Visualization**

**Author:** Sripadapudeekshith

**Target Hardware Core:** STM32F103C8T6 (ARM Cortex-M3 Blue Pill)

**Development Framework:** Bare-Metal C / Asynchronous Python Ecosystem

---

## 1. Executive Summary

This document serves as the formal technical specification for the **Real-Time EV Dashboard and Advanced Driver Assistance System (ADAS)**. In safety-critical automotive environments, the synchronization of real-time telemetry processing with driver assistance systems demands highly deterministic, low-latency execution loops.

This project implements an end-to-end simulation blueprint for an Electric Vehicle Electronic Control Unit (ECU). Using bare-metal C on the **STM32 Blue Pill** microcontroller, the system samples analog driver controls, processes a physics-based vehicle inertia engine, runs an explicit hazard management finite state machine, and streams packed serial frames to an asynchronous, dual-threaded Python visualization interface at **115,200 bps**.

---

## 2. System Architecture & Topology

The engineering topology of the platform is strictly divided into four functional operational layers to guarantee maximum decoupling, structural modularity, and error isolation:

### 2.1 Hardware Input Layer (Sensor Emulation)

Since physical vehicle drivetrains present extreme deployment hazards during early prototyping phases, driver commands and physical environments are emulated via a high-fidelity hardware input layer:

* **Driver Input Controls:** Potentiometer banks isolate linear voltages representing the exact deflection of the Accelerator Pedal and Brake Pedal ($0\% \text{ to } 100\%$).
* **System Environment Factors:** Separate voltage networks emulate ambient battery State of Charge (SOC) degradation and Motor Temperature fluctuations ($0^\circ\text{C} \text{ to } 150^\circ\text{C}$).
* **Spatial Proximity Sensors:** A multi-axis sensor array maps physical obstructions along the Front, Left, and Right perimeters of the vehicle.

### 2.2 Processing Layer (Bare-Metal Firmware)

The core computational heavy lifting resides entirely inside the ARM Cortex-M3 MCU registers. The firmware acts as the main vehicle controller, containing:

1. An Analog-to-Digital Converter (ADC) scanning engine.
2. A vehicle kinematics physics engine.
3. A safety interlock Finite State Machine (FSM).
4. A diagnostic fault supervisor.

### 2.3 Data Pipeline (Serial Transport)

Data is cast from internal multi-variable C-structures into a single compressed ASCII transmission string. Communication is driven asynchronously over the Universal Asynchronous Receiver-Transmitter (UART) peripheral, linked across virtual COM boundaries via a low-overhead software bridge.

### 2.4 Presentation Layer (Python Visual Dashboard)

The host PC environment runs a decoupled UI application that reads incoming UART frames via standard expression parsers, populating a real-time responsive graphical cockpit cluster.

---

## 3. Firmware Design & Real-Time Scheduling

Automotive microcontrollers must fulfill strict deterministic execution timing boundaries. Traditional programming paradigms relying on blocking loops like `HAL_Delay()` are unacceptable, as they freeze CPU processing cycles and introduce significant latency, leaving the vehicle blind to overlapping safety threats.

### 3.1 Hardware Cooperative Scheduler

To achieve high execution predictability, this architecture implements a strict hardware-driven **Cooperative Scheduler** utilizing native timer interrupts rather than an external RTOS overhead.

* The internal main system clock is driven at its maximum capacity of **72 MHz**.
* A timer Prescaler value of **71** steps down the internal counter frequency to exactly **1 MHz** (resulting in a stable $1\,\mu\text{s}$ baseline clock tick).
* The Auto-Reload Register (ARR) is configured to **9,999**, generating a periodic hardware interrupt flag exactly every **10 milliseconds** ($100\text{ Hz}$ execution rate).

### 3.2 Dual-Rate Task Slicing

The interrupt handler governs two cleanly decoupled operational tasks depending on their urgency:

* **High-Priority Task Thread (100 Hz / 10ms execution loop):** Executes immediate hardware ADC round-robin conversions to ensure the system reads accelerating or braking inputs with zero perceivable latency.
* **Low-Priority Task Thread (10 Hz / 100ms execution loop):** Coordinates spatial ultrasonic trigger waves, updates internal EV physics parameters, runs diagnostic interlocks, and formats the serialization frame to be pushed out over the UART bus.

---

## 4. Hardware Peripheral Ingestion & Processing

### 4.1 12-Bit Analog-to-Digital Conversion (ADC)

The STM32 features a built-in 12-bit Successive Approximation ADC. During operation, analog voltage signals varying from $0\text{V to } 3.3\text{V}$ are discretized by the peripheral hardware into high-precision unsigned integers ranging from **0 to 4095**:


$$\text{Digital Value} = \left( \frac{\text{Analog Voltage}}{3.3\text{V}} \right) \times 4095$$

These raw numeric values are read sequentially across four analog channels via continuous polling inside the 10ms scheduler interval, then scaled using software normalization filters to true engineering values:

* $\text{Pedal Deflection} = \left(\frac{\text{Raw ADC}}{4095}\right) \times 100\%$
* $\text{Motor Temperature} = \left(\frac{\text{Raw ADC}}{4095}\right) \times 150^\circ\text{C}$

---

## 5. EV Core Control Logic & Kinematics Engine

### 5.1 Physics-Based Vehicle Inertia Modeling

Rather than updating speeds based on arbitrary lookup variables, the firmware solves fundamental vehicle dynamics inside the controller loops. The scaled accelerator percentage translates directly into target **Motor Torque** based on selected drive maps (Eco, Normal, Sport).

Instantaneous linear vehicle acceleration is calculated dynamically using a foundational physics balance equation:


$$\text{Acceleration } (a) = \frac{\text{T}_{\text{motor}} - \text{F}_{\text{drag}}}{\text{Mass}}$$


Where $\text{F}_{\text{drag}}$ accounts for wind resistance coefficients proportional to the current velocity squared, and $\text{Mass}$ represents total simulated vehicle weight. The system integrates this acceleration factor to continuously compute the vehicle's true forward speed ($km/h$).

### 5.2 Regenerative Braking Model

Energy efficiency optimization loops are deeply integrated within the core torque calculation blocks. The firmware continually tests driver intent via an explicit mechanical deadband check:

* When the accelerator drops to zero and the brake pedal crosses a **5% initial engagement threshold**, positive motor torque drops to zero.
* The system actively commands a computed **negative torque** state—bounded by an operational safety ceiling of **$-80\,\text{Nm}$**.
* This models the kinetic energy of the turning wheels converting the electric motor into a generator. The vehicle velocity rapidly retards, and the system translates the captured mechanical forces into power, causing the battery **State of Charge (SOC)** variable to increment positively.

---

## 6. Advanced Driver Assistance System (ADAS) Specifications

### 6.1 Spatial Buffer Ingestion

The spatial protection layer tracks physical clearances via simulated high-frequency ultrasonic rangefinder metrics across the Front, Left, and Right sectors of the vehicle.

### 6.2 Time-to-Collision (TTC) Diagnostics

While static proximity checks provide baseline alerts, true safety critical computing relies heavily on **Time-to-Collision (TTC)** metrics. The firmware actively divides the physical real-time distance gap to a forward object by the vehicle's instantaneous operational speed:


$$\text{TTC } (\text{seconds}) = \frac{\text{Distance to Obstacle } (\text{meters})}{\text{Current Velocity } (\text{meters/second})}$$

### 6.3 Multi-Tier Hazard Escalation Matrix

The ADAS algorithm passes calculated values through a strict multi-tier safety validation framework:

1. **Nominal Status:** All tracking channels report adequate distance fields; vehicle operates normally.
2. **Advisory Warning:** Triggered if spatial parameters are fine, but the vehicle velocity drifts beyond safe cruise limits.
3. **Proximity Alert Warning:** Automatically activated if any external boundary wall or obstacle enters a close **50 cm** parameter window.
4. **Critical Collision Warning:** Imposed instantly if an obstacle breaches the immediate **20 cm** defensive perimeter OR the calculated structural Time-to-Collision falls below **1.5 seconds**.

---

## 7. Deterministic Finite State Machine (FSM) & Fail-Safe Loops

### 7.1 FSM State Topology

To completely rule out uncommanded transitions or erratic physical performance, the system architecture operates as a strict, state-isolated Finite State Machine:

* `PARK`: Initial boot state. The powertrain is completely isolated; motor torque output is mathematically blocked at absolute zero.
* `READY`: Safety systems arm. Transitions to Ready occur only when pre-flight background register checks are completely validated.
* `DRIVING`: Active control state. Driver inputs directly dictate positive motor torque and energy depletion.
* `REGENERATION`: Energy recovery tracking mode. Activated via brake deflection to feed current variables back into battery capacity.

### 7.2 Fault Management & Powertrain Isolation

Operating parallel to the FSM is an absolute diagnostic supervisor loop. The system continuously benchmarks performance metrics against emergency parameters. If any of the following **Critical Fault Conditions** are met:

* **Thermal Overrun:** Motor Temperature crosses a critical **$120^\circ\text{C}$** safety threshold.
* **Undervoltage Collapse:** Battery capacity drops below an emergency **$5\%$** minimum floor.
* **Proximity Violation:** Front physical clearance drops below an absolute **$30\,\text{cm}$** safety line.

The fault manager instantly fires a hardware-level override, sets the corresponding diagnostic hexadecimal fault code (`0x01` for Thermal, `0x02` for Battery, `0x04` for Proximity), overrides all FSM execution chains, forces a shift into a secure `FAULT` state, and cuts motor speed and torque paths to absolute zero to safeguard human operators and system components.

---

## 8. Data Pipelines & Visual Analytics Interface

### 8.1 Serialization Protocol

Streaming numerous high-precision variables continuously can place massive computational loads on lightweight embedded cores. To bypass heavy floating-point calculation demands, the firmware packs active variables into an optimized, space-separated ASCII line package before broadcasting:

```text
SPD:45 SOC:88 TRQ:32 TMP:74 RNG:210 ACC:40 BRK:0 FWD:120\n

```

This payload is pushed out across the UART TX pin at a fast **115,200 bits per second** exactly every 100 milliseconds.

### 8.2 Asynchronous Multi-Threaded Host Application

At the presentation layer, the visualization platform splits operations into a dual-threaded processing model to maintain high visual frame rates without risking the loss of incoming data packets:

* **Thread 1 (Serial Ingestion Engine):** Operates as a non-blocking background worker. It continuously samples characters from the incoming serial buffer, isolates string boundaries, validates data integrity using optimized Regular Expression (Regex) search filters, and updates a shared internal system dictionary.
* **Thread 2 (GUI Animation Loop):** A graphical loop executing strictly at a decoupled **$10\text{ Hz}$ frequency (every 100ms)**. It extracts variables from the shared data dictionary to seamlessly refresh a five-axis digital instrument cluster, containing an arc speedometer, an SOC tracking meter, historical line trends, and an interactive **Top-Down Bird's-Eye ADAS Radar plot** that flashes active warning regions to the driver.

---

## 9. Engineering Summary & Verification

This project successfully demonstrates the implementation of complex automotive electronics using bare-metal architecture. Through hardware cooperative interrupt task slicing, the system achieves predictable, deterministic real-time data acquisition with minimal CPU overhead.

The integration of physics models, low-latency ASCII serialization, an absolute state-machine structure, and an isolated background diagnostic fault supervisor provides a highly scalable blueprint for production-grade vehicle control electronics.
