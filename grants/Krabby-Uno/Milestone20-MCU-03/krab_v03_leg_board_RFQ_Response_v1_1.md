**Krab v0.3 Per-Leg Motor Controller Board**

**STM32G474 + three STSPIN9P21 full bridges + CAN-FD \| Hardware
proposal**

| ARCHITECTURE | SCHEDULE | FIXED PRICE |
|---|---|---|
| Six identical leg boards | 20 business days to Gerbers | $3,000 USD |

# **1. Objective and Scope**

Redesign the v0.2 power board into one self-contained controller
installed on each of the robot’s six legs. Each board will drive three
brushed actuators and integrate one STM32G474RET6, three STSPIN9P21 full
bridges, one CAN-FD transceiver, three current/position feedback
interfaces, XT30/XT60 power connections, and local 5 V/3.3 V regulation.
The Orin J4012 is the mid-bus leader; the two rear leg boards are the
physical CAN termination points.

> **Scope boundary:** Production firmware, CAN application protocol,
> motor-control algorithms, CAN bootloader code, Jetson/ROS software,
> harness/enclosure design, certification, fabrication, assembly, and
> shipping are not included.

# **2. Major Work Steps**

| **STEP** | **WORK PACKAGE**                       | **PLANNED OUTPUT**                                                                                                                                                   |
|----------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1        | **Requirements freeze**                | Confirm actuator current/feedback data, 48 V pack maximum/full-charge voltage, regeneration, mechanics, connector parts, CAN rate, and bootloader licensing.         |
| 2        | **Schematic architecture**             | Translate useful v0.2 signals; define MCU power/clock/reset/SWD/BOOT0, three-bit node ID, power tree, protection, and connector pinouts.                             |
| 3        | **MCU, CAN and update provisions**     | Integrate STM32G474 + MCP2562FD, dual parallel JST-GH CAN ports, ESD/termination, SWD recovery, and CAN-bootloader-compatible hardware.                              |
| 4        | **Motor, feedback and power**          | Replace six BTN7960B half bridges with three STSPIN9P21 channels; add current/pot/Hall conditioning, XT30/XT60, fusing, 100 V-class power parts, and 12–72 V buck.   |
| 5        | **PCB layout and design review**       | Four-layer layout with short switching loops, thermal copper/vias, separated analog/CAN routing, creepage, labels, test points, ERC/DRC, and DFM review.             |
| 6        | **Manufacturing release and closeout** | Release BOM, Gerbers, drills, pick-and-place, assembly drawings, JLC package, test checklist, one review-driven revision, and one agreed first-article hardware ECO. |

# **3. Schedule and Payment Milestones**

| MILESTONE | TARGET | DELIVERABLE | PAYMENT |
|---|---|---|---|
| 1\. Confirm Requirements | 1 day      | RQF                                                                                         | \$600  |
| 2\. Schematic design     | 7 day      | Complete reviewed schematic and preliminary BOM                                             | \$300  |
| 3\. PCB design           | 5 days     | Placement, routing, power/thermal review and manufacturing files                            | \$1300 |
| 4\. Final release        | 7 days     | Final BOM, Gerbers, drill files, pick-and-place, assembly drawings, hardware test checklist | \$800  |

# **4. RFQ Design Questions**

**1. STM32G474-to-CAN wiring**

Yes. FDCAN1 PA12/TX drives MCP2562FD TXD; PA11/RX receives RXD; PA15 controls STBY. MCP2562FD uses 5 V VDD and 3.3 V VIO, with local decoupling, ESD/TVS, optional common-mode choke, and selectable 120 Ω termination. References: STM32G474 NUCLEO schematic, MCP2562FD data sheet, and Seeed J401 manual.

**2. CAN firmware-update hardware**

The G474 factory ROM bootloader does not support CAN/FDCAN. Initial programming therefore uses SWD; an OpenBLT-style custom bootloader can update over CAN afterward. I will provide SWD, NRST, BOOT0, HSE, three-bit board ID, and safe transceiver-state hardware. Before PCB release, hardware compatibility can be bench-checked using NUCLEO-G474RE + MCP2562FD and the published OpenBLT G474 CAN demo. Production bootloader code/licensing are outside scope.

**3. Pin capacity and peripheral compatibility**

Yes. The preliminary map uses hardware FDCAN, six timer PWM outputs, six Hall timer inputs, six ADC inputs, three enables, three faults, three DAC references, SWD, and a three-bit leg address. The 64-pin device has sufficient pins; the schematic review will complete the final alternate-function conflict check.

**4. Current sense and Hall/pot feedback**

Yes. Three STSPIN SNSO outputs and three potentiometers route to protected/filterable ADC channels; three Hall pairs route to timer-capable inputs. Scaling, pull-ups, hysteresis and filters depend on the actuator sensor voltage/output type and harness length, which must be confirmed before schematic freeze.

**5. CAN commands to three motor channels**

The hardware path is CAN frame → FDCAN peripheral → MCU logic → local timer PWM/enable to each STSPIN9P21. CAN does not carry a physical PWM waveform; firmware interprets commands and generates local voltage/direction control. The electrical path is included; CAN protocol and control firmware are excluded. Existing PWM L/R and EN semantics will be retained where technically appropriate.

**6. Additional pins and service interfaces**

Provide Tag-Connect/SWD, NRST/BOOT0 pads, three-bit node-ID straps or DIP option, selectable termination, UART/service pads, CAN/ADC/fault test points, driver nFAULT/nOL, VREF test points, and spare GPIO where routing permits. Only required field interfaces will use production connectors.

**7. 72 V-rated power redesign**

Feasible for the RFQ’s stated 48 V maximum system pack using 80/100 V-class capacitors, TVS/clamp, reverse-polarity stage, buck and spacing. STSPIN9P21 is recommended for 7–75 V and 80 V absolute maximum, so it is not suitable for a true 72 V nominal/84 V charged pack with transients. Full-charge voltage, regenerative energy, motor stall current and cable inductance are design gates.

**8. Connector changes**

No fundamental concern. Exact XT30/XT60 manufacturer part numbers, gender, polarity, orientation, cable exit, strain relief and ratings must be confirmed. J401 CAN is 4-pin JST-GH: GND, CAN_L, CAN_H, 5 V. Because each leg generates local 5 V, the J401 5 V pin will be isolated/no-connect by default to avoid backfeed unless an agreed pass-through option is required.

**9. EMI control**

Use a four-layer stack with continuous ground reference, compact high-current loops, driver-local bulk/ceramic decoupling, separated power/analog/CAN zones, controlled returns, filtered sensor inputs, adjustable edge rate, and optional TVS/common-mode choke/snubber footprints. CAN and feedback will not run beside bridge outputs or VM switching loops.

**10. Heat management**

Use large exposed-pad copper areas, thermal-via arrays to internal/bottom planes, driver spacing, and 2 oz outer copper as the initial construction. Add a PCB temperature/NTC provision and reserve mechanical access for a heat spreader if analysis requires it. Final copper and first-article validation depend on stall/running current, duty, ambient, airflow, and simultaneous-axis loading.

**11. Protection against wiring faults**

Planned protection: XT60 input fuse, three motor fuses, reverse-polarity ideal-diode stage, TVS/surge clamp, inrush provision, STSPIN current limit/OCP/thermal/UVLO, CAN/sensor ESD protection, keyed connectors, and motor snubber footprints. Reversing a motor pair changes direction; feedback inputs are protected for expected field faults, not arbitrary traction-bus voltage.

**12. Status indicators**

Retain VM present (renamed from 24 V), 5 V and 3.3 V LEDs; add MCU heartbeat/status, CAN activity, and per-driver or aggregate fault. Remove or buffer high-frequency PWM L/R LEDs to avoid loading/noise, while retaining labeled test points for PWM, enable, current sense and feedback.

**13. CAN daisy chain and transceiver**

Use one MCP2562FD-E/SN per board and two parallel JST-GH ports labeled CAN IN/OUT; these are bussed connectors, not two transceivers. Keep internal stubs short. Enable 120 Ω only at Rear Left and Rear Right; the four intermediate leg boards are unterminated and Orin taps the bus mid-line. Retain a DNP alternate footprint if sourcing changes.

**14. JLCPCB selection and BOM**

The architecture removes the shield and six bridge ICs but adds MCU, CAN, wide-input buck, protection and higher-voltage parts. I will target a neutral/lower system BOM, but cannot guarantee it until motor current and sourcing are frozen. STSPIN9P21, STM32G474RET6, the buck, 100 V passives and genuine XT parts require JLC/LCSC availability checks and may need extended parts or consignment.

**15. Other concerns and kickoff inputs**

Required: exact actuator models; rated/running/stall current and duty; feedback pinouts/levels; battery chemistry, full-charge voltage and regeneration; board outline/mounting; harness lengths; connector part numbers; CAN nominal/data rates; and bootloader license choice. The baseline BOM appears to use six BTN7960B ICs—please confirm the RFQ’s BTS7960 wording refers to them. Six leg addresses require a three-bit node ID.

# **Preliminary STM32G474RET6 Pin Allocation**

| FUNCTION | PINS | FUNCTION | PINS |
|---|---|---|---|
| **FDCAN1**     | PA12 TX / PA11 RX / PA15 STBY       | **PWM M1**       | PC6 / PC7 — TIM3 CH1/CH2 |
| **PWM M2**     | PC8 / PC9 — TIM3 CH3/CH4            | **PWM M3**       | PA8 / PA9 — TIM1 CH1/CH2 |
| **Driver EN**  | PB10 / PB11 / PB12                  | **Driver fault** | PB13 / PB14 / PB15       |
| **Pot ADC**    | PC0 / PC1 / PC2                     | **Current ADC**  | PC3 / PB0 / PB1          |
| **Hall pairs** | PA0/1; PA2/3; PB6/7                 | **DAC VREF**     | PA4 / PA5 / PA6          |
| **Debug/boot** | PA13 SWDIO / PA14 SWCLK / PB8 BOOT0 | **Board ID**     | PB4 / PD2 / PB9          |

**This allocation is preliminary** and may move during schematic/layout
work while preserving the required peripheral functions.
