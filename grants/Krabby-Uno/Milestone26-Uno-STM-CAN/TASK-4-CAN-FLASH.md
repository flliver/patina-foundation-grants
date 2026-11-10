# Task 4 — Flash over CAN

Goal: Update firmware on any leg over CAN using the OpenBLT bootloader, confirmed by updating all six boards from the Orin and by recovering a board after a power cut during a flash.

Outputs
- `firmware/stm/bootloader/`: OpenBLT configured for this board.
- An application linked above the bootloader, with a reset on XCP connect in `can_handler.c`.
- `update` over CAN in `krabby-firmware-stm`, and `krabby firmware --stm` on the Orin.
- S-record files published next to the `.elf`.
- A task report covering the power-cut test and the six-board update.

(Acceptance criteria and time estimate for this task live in [OVERVIEW.md](OVERVIEW.md#task-4---flash-over-can).)

---

Note: line numbers below point at the right function at the time of writing and may drift.

---

## 1. Bootloader

The G474's built-in ROM bootloader cannot flash over CAN, which is why the board needs OpenBLT. Start from OpenBLT's Nucleo-G474RE demo and change it for this board: the 8 MHz crystal, FDCAN2 on PB12 and PB13 at 1 Mbit/s, and 128 KB of flash, since the demo's chip has 512 KB.

Check the demo's flash layout table first, because the RB part's flash is smaller and its sector list must match. Put the bootloader at `0x08000000` and give it the smallest whole number of flash pages it fits in, with a limit of 32 KB. Link the application directly above it, set `SCB->VTOR` to the application's start address, and let the linker fail if the application outgrows its space. Install the bootloader once per board with `STM32_Programmer_CLI` over SWD.

The bootloader must keep the motors off. `M*_SLEEP` stays low through its pull-down resistors as long as the bootloader leaves those pins alone.

## 2. CAN IDs

Each leg needs its own bootloader IDs, because all the boards on a bus would otherwise respond to the same connect frame. Use `0x600 + leg` for commands to the board and `0x680 + leg` for its responses, and add these to `krabby.dbc` so they appear by name in `candump`.

OpenBLT's port expects these IDs as fixed configuration, so its CAN code needs a small edit to read the node ID switch at startup (PC13, PB4, PB5) and compute them. The application and the bootloader then use the same switch, so one bootloader image works on all six boards.

## 3. Entering the bootloader

The application watches for the XCP connect frame, which is the two bytes `FF 00` on the board's command ID. Add `0x600 + node_id` to the FDCAN filter. When `can_handler_on_frame()` receives this frame, it sets all three motors to 0 and resets the MCU with `NVIC_SystemReset()`. OpenBLT's demo application does the same thing in its `boot.c`, so check the frame format there.

After a reset, OpenBLT waits briefly for a connect before starting the application. The host keeps sending connect frames until it gets a response, so the bootloader catches one during this wait and stays active. If no connect arrives, the application starts as normal. Keep the wait at OpenBLT's default of about 500 ms, unless testing shows it needs changing.

OpenBLT writes the application's checksum last and only starts an application whose checksum is valid. An interrupted flash therefore leaves the board in the bootloader, ready for another attempt.

## 4. CLI

OpenBLT's host library, LibOpenBLT, includes a Python package and reads S-record files. Add `arm-none-eabi-objcopy -O srec` to the STM build, and publish `krabby_uno.srec` with an `srec_url` entry, the same way as `elf_url` in Task 1.

`update` gains `--leg N` and `--all`, and flashes over CAN when either is given. The existing `update <branch>` with neither flag keeps flashing over SWD, for new boards and for recovering a board whose bootloader is damaged. The over-CAN flow is:

1. Download the S-record file for the branch.
2. Send zero duty to all legs, because a leg in the bootloader stops responding and the robot should not be moving during an update.
3. Listen for BoardInfo to learn which bus each target leg is on. Stop with an error if a target leg is silent.
4. Flash each target leg with LibOpenBLT, using that leg's bus and IDs. Flash one leg at a time.
5. Wait for BoardInfo from each flashed leg, and print its old and new versions. Report an error if a version does not match the build.

## 5. Orin

LibOpenBLT is a compiled library, and the Orin is ARM64, so build it from source in `images/locomotion/Dockerfile.release` and install its Python package there. Add a `--stm` option to `krabby firmware` in `krabby/firmware.py` and to `firmware_cmd()` in `krabby/_docker.py` (~L130), which selects `krabby-firmware-stm` as the container's entrypoint in place of `krabby-firmware`. The container already has host networking on the Jetson, so it can reach `can0` and `can1`.

## 6. Testing

On a bench board, flash build A over SWD, then update to build B over CAN and confirm BoardInfo reports B. Update back to A and confirm it reports A. Check that the board still drives its motors after each update, and that a second leg on the same bus keeps sending telemetry while the first one flashes.

Cut the board's power during a flash, once while erasing and once while programming. On power-up, confirm that the board stays in the bootloader with its motors off and that running `update` again completes the flash. Then cancel a flash from the host with Ctrl-C and run `update` again.

Finish with `krabby firmware --stm update --all <branch>` on the robot, run from the Orin, and confirm that `show` lists the new version on all six boards. Record the results in the task report.
