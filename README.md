![Version](https://img.shields.io/badge/version-0.1-white)
[![License](https://img.shields.io/badge/license-Apache--2.0-red.svg)](LICENSE)
![Status](https://img.shields.io/badge/status-needs--validation-blue)


ICE180LM - Behavioural SPICE model of the Infineon ICE180LM CoolSET SiP flyback controller
=========================================================================================

Simulator : LTspice (developed and tested with LTspice 24.0.12). Uses LTspice-specific syntax
            (behavioural B-sources with limit(), if(), time-varying PWL/PULSE sources, SW/D models).
            It is NOT portable to other SPICE simulators without changes.
Files     : ice1801_primary.lib   the model (subcircuit ICE1801LM, primary + secondary in one file)
            ICE1801LM.asy         LTspice schematic symbol (16 pins)
            README.txt            this file
            tests/                regression tests (Python + LTspice batch mode)
            eval_board/           EVAL_100W1_ZVS_180LM board netlist (validation work in progress)
            progress_notes.md     development log: datasheet references, assumptions, bugs found

*** THIS IS NOT AN OFFICIAL INFINEON MODEL AND HAS NOT BEEN VALIDATED OR APPROVED BY INFINEON. ***

It is an independent, unofficial, behavioural model built from the public datasheet (ICE1 100LM
series, R1.0, 2026-06-22) and the public EVAL_100W1_ZVS_180LM engineering report. It is not provided,
reviewed, supported or endorsed by Infineon Technologies AG. Infineon, CoolSET and CoolSiC are
trademarks of their owner and are named here only to identify the part being modelled.

The model reproduces the control behaviour of the IC, not its transistor-level circuit. Use it for
design-in and system-level simulation only. It is provided as is, without warranty, and its results
are no guarantee of real hardware behaviour: always verify a design on hardware and against the
official Infineon datasheet.


1. INSTALLATION
---------------
1. Put ice1801_primary.lib and ICE1801LM.asy in the same folder (for example next to your schematic,
   or LTspice's user lib\sub and lib\sym folders).
2. The symbol's ModelFile attribute currently holds an absolute path:
       C:\Users\90541\Desktop\ice1801_spice_model\ice1801_primary.lib
   Change it to a relative name (SYMATTR ModelFile ice1801_primary.lib) or to your own path before
   sharing the symbol.
3. Place the symbol (component X, value ICE1801LM). LTspice includes the library automatically.
   Alternatively add   .lib ice1801_primary.lib   to the schematic.
4. If LTspice stops with "Unknown SPICE device type" on the first line of the library, the file was
   saved with a UTF-8 byte-order mark. Re-save it as UTF-8 without BOM.


2. PINS (subcircuit order = symbol SpiceOrder)
----------------------------------------------
  #  Pin    Domain     Function / typical connection (EVAL_100W1_ZVS_180LM)
  1  DRAIN  primary    Internal power switch drain (CoolSiC, 83 mohm typ, Coer 60 pF)
  2  HV     primary    Startup cell / brown-in sensing, via RHV to the DC bus (R3+R4 = 2 x 50 k = 100 k)
  3  VCCP   primary    Primary supply. Startup cell charges it; aux winding + regulator hold it afterwards
  4  GNDP   primary    Primary ground
  5  ZCDP   primary    Aux-winding divider (RZCDPH 22 k to the aux diode, RZCDPL = 2 k to GNDP, 33 pF).
                       RZCDPL also selects the brown-in/brown-out option (section 3)
  6  VINP   primary    Bus-voltage sense for line OVP (divider from the DC bus through the ENP-controlled switch)
  7  ENP    primary    Output that switches the VINP divider (10 V when enabled, 0 V when off)
  8  CS     primary    Switch source / current sense: sense resistor (0.165 ohm on the eval board) to GNDP
  9  GNDS   secondary  Secondary ground
 10  GDSR   secondary  SR MOSFET gate drive (10 V)
 11  ZCDS   secondary  SR drain sensing through a resistor (15 k for NMAIN/NSEC = 8)
 12  VCCS   secondary  Secondary supply (from the output voltage)
 13  FB     secondary  Feedback input, reference 1.2 V
 14  EA     secondary  Error-amplifier output; external compensation network to GNDS
 15  CONF0  secondary  RSET0 to GNDS: transformer turns ratio NMAIN/NSEC
 16  CONF1  secondary  RSET1 to GNDS: hysteretic-mode parameter set

Model-internal signals (gate, latches, counters) are not pins. When you need to look at them, run the
netlist flat (see section 6) or probe them as X1:name in LTspice.


3. EXTERNAL CONFIGURATION (resistor values from the datasheet, Tables 4-6)
--------------------------------------------------------------------------
Primary side, RZCDPL from ZCDP to GNDP (read once, at start-up, during the configuration window):
   Option 1: 1.00 - 1.05 kohm   BI 2.00 mA  BO 1.40 mA  RHVshunt 0.5 kohm
   Option 2: 1.87 - 2.70 kohm   BI 1.00 mA  BO 0.70 mA  RHVshunt 1.0 kohm     (eval board: 2 kohm)
   Option 3: 4.30 - 5.00 kohm   BI 0.67 mA  BO 0.47 mA  RHVshunt 1.5 kohm
   Option 4: 9.20 - 9.50 kohm   BI 0.50 mA  BO 0.35 mA  RHVshunt 2.0 kohm
   Brown-in voltage ~ (RHV + RHVshunt) x BI current (101 V with 100 k and option 2).
   A value outside every window sets CONFIG_FAULT; ZCDP below 100 mV during the window triggers the
   ZCDP short-to-GNDP protection (auto-restart).

Secondary side (50 uA is forced into each CONFx pin):
   RSET0 (turns ratio):   3.9k=5   6.8k=6   12k=7   18k=8   27k=9   39k=10
   RSET1 (parameter set): 3.9k, 6.8k, 12k -> VEA_HMon 1.25 / 1.20 / 1.10 V, VEA_HMoff 0.95 / 0.9 / 0.8 V,
                          VEA_LHM 1.4 V; 18k, 27k, 39k -> same HMon/HMoff, VEA_LHM 1.6 V.
   A resistor outside the option windows sets CONF0_FAULT / CONF1_FAULT.


4. HOW TO CONNECT IT (isolation and grounds)
--------------------------------------------
- GNDP and GNDS are separate, isolated domains and nothing in the model connects them. Each needs a DC
  path to the solver reference, otherwise LTspice reports a singular matrix or floating nodes.
- Tie GNDP to node 0 with a small resistor (Rgndp GNDP 0 0.1m). Never use a 0-ohm resistor: LTspice
  cannot reliably solve one.
- For a full board tie GNDS to GNDP through the datasheet isolation resistance (Rio GNDS GNDP 1G) and add
  the board's Y capacitor (1.5 nF on the eval board). For a test of one side only, tie that side with 0.1m.
- Signals that cross the barrier (CT-Link) are re-driven by behavioural sources that read the secondary
  side as V(x,GNDS), so a large DC offset between GNDP and GNDS does not disturb the model.
- The junction temperature node TJ is internal (weak 25 C default). To sweep temperature, drive the TJ
  node of the instance from a testbench or edit the default in the library.


5. WHAT IS MODELLED
-------------------
Primary  : VCCP startup cell (drawn from the HV pin), UVLO, VCCP OVP, HV brown-in / brown-out, config
           readout (RZCDPL), ZCDP zero-crossing detection with ringing suppression, output OVP / UVP,
           VINP line OVP, CS current sense with OCP1 / OCP2, gate driver, soft-start, tOnMax / tOffMax,
           close-loop timeout, ENP, OTP, auto-restart (plain, non-switch and eight-skip classes),
           integrated power switch.
Secondary: VCCS management, FB amplifier (OTA) and EA clamp, open-loop protection, SR gate driver with
           ZCDS sensing, CONF0 / CONF1 decode, take-over of control, QR target-valley selection, valley
           counting, fsw_min / fsw_max limits, tCCMperiod, skip mode, PWM on-time generator with ZCDS
           feed-forward, LINE_HIGH detection, hysteretic mode, frequency jitter.
Link     : behavioural CT-Link (secondary to primary): take-over stop, closed-loop flag, turn-on and
           turn-off requests, hysteretic-sleep flag (delay 50 ns).


6. SIMULATION TIPS
------------------
- Use  .tran <tstop> 0 <tmax> uic  with tmax around 100 ns for a switching converter, and give the output
  and supply capacitors initial conditions (ic=...) to skip the start-up transient.
- Do not save every node of a flattened board: the raw file grows to gigabytes. Use .save for the nodes you
  need, or probe only the pins.
- The model alone (no power train) runs about 500 us of simulated time in a few seconds. Runs with the
  full power stage switching are much slower; run times have not been characterised yet.
- Avoid delay() in anything you add: LTspice limits the time step to delay/20 for the whole circuit.
- The regression suite needs Python with the spicelib and numpy packages:
      python tests/run_all.py
  Each test extracts the real block from the library, runs LTspice in batch mode and checks numeric
  pass/fail conditions. All 12 tests pass on the current library.


7. VALIDATION STATUS
--------------------
- Block level: 12 automated tests (CT-Link, fsw_min, fsw_max, tCCMperiod, valley counting, skip mode,
  PWM on-time generator, LINE_HIGH, hysteretic mode, jitter, ENP, ZCDP configuration readout), plus
  the earlier bench tests of the primary side recorded in progress_notes.md.
- System level: the EVAL_100W1_ZVS_180LM board netlist exists (eval_board/), but comparison with the
  engineering report (ER090656 V1.0) measurements is NOT complete. Do not treat efficiency, frequency,
  ripple or start-up timing results as validated yet.


8. KNOWN LIMITATIONS AND ASSUMPTIONS (where the datasheet is silent)
--------------------------------------------------------------------
- ZVS current-injection pulse (SR turn-on before the primary) is not modelled. High-line ZVS operation is
  therefore not reproduced; valley switching is.
- CT-Link delay 50 ns (no datasheet value). CT pulses must be wider than about 100 ns.
- Configuration window length 100 us, integrated RZCDP 1 Mohm, and the ZCDP sense current 200 uA are
  assumptions (a current below 130 uA would make option 1 trip the short-to-GNDP check).
- PWM on-time is linear in EA between VPWM_lclamp and VPWM_hclamp; the ZCDS feed-forward is normalised
  to unity at IZCDS_mid; the ZCDS clamp switch is 10 ohm.
- Jitter (4 kHz, +/-4 %) is applied to the on-time as a triangle; the datasheet does not say what is modulated.
- The SR gate is kept off in hysteretic mode.
- Internal Coss is a constant 60 pF (real value is voltage dependent); the body diode is generic.
- Startup-cell knee 2 V. Time constants of all RC / ramp timers are tuned to the typical datasheet values.
- Saturating currents use smooth tapers, so latch levels top out near 4.55 V and floors near 0.02 V.
  Do not add logic that compares a latch to exactly 5 V or 0 V.
- Every assumption is flagged in the library comments and in progress_notes.md.


Version : 0.1 (development)
Author  : SuBardagi (library header)
