# M20 Design Notes — Client Watch List

Things that came up during v0.2 bring-up that need to carry forward into the v0.3 design. This is a running list of specific hardware decisions, lessons from the existing boards, and open questions that should be resolved and signed off during the schematic and layout reviews — not a replacement for the RFQ, but a supplement to it.

---

1. **LED placement.** All indicator LEDs need to move off the side edge of the board — they currently protrude slightly and catch on the 3D-printed chassis, getting ripped off easily. Alternatively, use a larger pad footprint for the LEDs so they're more securely attached.

2. **LED brightness parity.** The 24 V LED indicator is much brighter than the 5 V LED. The 24 V brightness is good; the 5 V indicator should match it.

3. **Remove variable resistors R20/R29/R38.** The upstream 4.6 kΩ resistor is sufficient; the variable resistor is not needed and can be eliminated.

4. **Size for 250 W motors.** All thermal design, capacitors, and power path components should be sized to support up to 250 W per motor. Current highest-wattage motor is 130 W; see `krabby-research/hardware/Uno-v0.2/motors` for the motor specs.

5. **Power path copper weight.** Pay close attention to copper thickness on the high-current path: input connector → bulk caps → automotive fuses → full bridges → XT30 output connectors. This path must sustain continuous load without burning out.

6. **Heat sink selection.** Review the current heat sink (sourced from JLC) and flag if an upgrade is needed. Please identify a specific part number orderable through JLCPCB or Mouser so we can order it.

7. **CAN daisy chain — two connectors per board.** Each board needs CAN in and CAN out connectors (dual JST-GH). Ground and power must be shared across all boards over the CAN cable.

8. **3.3 V power source — open question.** Unclear whether 3.3 V should come from the Orin (over the CAN connector) or from the battery via onboard regulation. Preference is Orin-sourced for a cleaner supply (18 motors on the battery line with occasional stalls will cause significant current fluctuation), but please advise if you see a reason to do otherwise. If Orin-sourced, it needs to be brought over the CAN connector.

9. **Shared ground across all boards.** Make sure a clean shared ground is carried over the CAN connector and maintained across all six boards.

10. **Unused pin audit.** List all unused pins on the STM32G474 and STSPIN9P21 and flag each one: should it be pulled out to a header for optional firmware use, or does it need to be tied to a fixed level per the datasheet?

11. **Floating pin handling.** Any pins not driven by the circuit should be explicitly pulled up or pulled down as appropriate — no floating inputs.

12. **1 kΩ resistor on POT signal lines.** Add a 1 kΩ series resistor on each motor's potentiometer signal wire. A miswired motor (control wire shorted to voltage) previously burned out two POT inputs by applying 5 V directly to the signal line; the 1 kΩ resistor should limit current and prevent this.

13. **HALL input protection.** Check whether HALLA/HALLB are vulnerable to the same miswiring scenario as POT (noting that HALLA/HALLB share a connector with POT). Add protection as appropriate.

14. **Qwiic connector on the STM32.** Add one Qwiic (JST-SH 4-pin) connector wired to an I2C-capable pin pair on the STM32G474. The existing Arduino shield manually breaks out D20/D21 to an OLED and IMU; this replaces that with a proper on-board connector. Use the same ground and power rail as the rest of the board.

15. **Debug interface selection.** Prefer the standard debug/flash interface used on STM32 Nucleo boards (e.g. compatible with ST-LINK-V3E; see the [STM32G4 Nucleo-64 user manual](https://www.st.com/resource/en/user_manual/dm00556337-stm32g4-nucleo-64-boards-mb1367-stmicroelectronics.pdf) for reference). Please confirm the interface you'll use and make sure it's brought out to a header.

16. **Reference a production STM32 dev board.** Review an STM32G474 Nucleo or equivalent reference design to confirm we're not missing any standard wiring, decoupling, or connectors that are easy to overlook.

17. **Apply manufacturer specs fully.** Follow the full STM32G474 and STSPIN9P21 datasheet recommendations for pin configuration and any required bypass/protection passives — don't leave anything from the "recommended application circuit" out.

18. **Hardware recovery from bricked state.** Make sure there is a clear, physical way to force the board into bootloader/debug mode if the firmware is bricked — typically a BOOT0 button.

19. **Onboard flash for config storage.** The STM32's internal flash will be used to store settings and calibration data. Ensure the hardware fully supports reflashing over CAN or the debug port with no additional provisions needed.

20. **No onboard input fuse needed.** There is an upstream 30 A fuse for the whole board; a redundant onboard fuse on the power input is not required (automotive power output fuses should remain as is).

21. **Miswiring burn-out audit.** Review all connector interfaces (Orin/CAN, motor power input, motor output, POT/HALL) and identify every realistic miswiring scenario that could damage the board, the Orin, or the motors. Add protection for each, and let me know if there are any non-protectable situations.
