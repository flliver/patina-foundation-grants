This contains some notes/details about Task 3, where we will get 3x arduino, each controlling 6x joints (i.e. a pair of legs each) to talk to each other using a leader/follower architecture.

#1 - Instead of using HW jumper for leader/follower, have firmware send out SYNC message on both serial, then the unit that gets back 2x SYNC should send FOLLOWER(FRONT) to 16/17 and FOLLOWER(REAR) to 14/15. Listeners when they receive FOLLOWER(FRONT/REAR) should send ACK. When leader receives 2x ACK, it stops sending and begins normal command proxy. All three devices should do this every boot, and should blink LED while sending SYNC commands so it's obvious if they are unable to sync.
#2 - Please start code from EXACTLY the code on krabby-research right now, and do not rename directories/files, as it is time consuming for me to re-merge things when the files names or content is unnecessarily modified. This includes comments and line structures, don't change other lines in the file, and before sending to me, do a diff and confirm that changes are minimized.
#3 - Joint telemetry can utilize JointTelemetry data class, and leader can just forward telemetry exactly as is, i.e. krabby_mcu.py will receive three lines, w/ six joints each. All leader should need to do is receive JT lines and resend on different serial.
#4 - Leader and follower should initialize ACT_LIST after they are synced, depending on which role they have (i.e. the unit receiving FOLLOWER(FRONT) should initialize FLHY, FLHL, FLKL, FRHY, FRHL, FRKL). Use four character notation in all appropriate places (e.g. FLHY, MLHY, RLHY)
#5 - krabby_mcu.py is already almost perfect, just need very light update for four digit actuator names on 'send neutral' I think. 
#6 - controller.ino loop(), cmdType == 'T' is the place where it needs to change to pull the appropriate actuators out by name and resend commands for other actuators to the followers. parseCommands needs to be modified to return middleCmdBuf, frontCmdBuf, rearCmdBuf, then send front/rear to new serial, and in dump telemetry section needs to read/resend telemetry it receives on other serials.
#7 - Calibration command needs to resend C to followers (joints can calibrate all at once for now, but I may change in the future)
#8 - Jog should identify if joint is local and resend if not
#9 - I think actuator_manager.h, command.h, joint_telemetry.h, joint_telemetry.py are pretty much untouched

$200 deposit
$200 final delivery 

Final delivery requirements:
1) Calibrate runs on all three leg pair at same time
2) Jog can work on all 18 joint with 4 digit code
3) Telemetry returns from all joint pairs (on three lines, 6x joint per line with proper 4 digit joint names)
4) T command list can work with any mix of joint commands across all three units, and properly splits/resends commands to appropriate unit(s). Write simple #4 test function that moves front/middle/rear left joints out all at same time, then back, then front/middle/rear right joints all out at same time, then back, for 2s each. 