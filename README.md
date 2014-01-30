ADC5G Testing
=============
This repository contains scripts and bitcodes for testing the 
ASIAA 5 GSps ADC (https://casper.berkeley.edu/wiki/ADC1x5000-8) 
on ROACH1 and ROACH2.

Requirements
------------
* Python >= 2.7
* numpy >= 1.6.2 (only for the 'array' object)
* corr >= 0.6.9 (only for katcp_wrapper.FpgaClient)

Getting Started
---------------
Begin with connecting to your ROACH and programming it:
```python
from corr import katcp_wrapper
roach = katcp_wrapper.FpgaClient(host, port)
roach.is_connected() # should return True
roach.progdev(bitcode)
roach.est_brd_clk()
```
The last line should return the approximate clock rate of the FPGA 
fabric; this should be close to `adc_clk/8` where `adc_clk` is the 
frequency of the sine wave you provide the ADC.

Calibrating data-to-clk
-----------------------
Glitches may occur if the rising-edge of the iSerDes capture clock 
(which is generated by a PLL locked to the externally provided ADC clock) 
does not occur within the data "eye", which can occur due to routing delays 
or component delays. Luckily the e2v EV8AQ160 provides a ramp test-vector 
which can be used to detect these glitches. The phase-relation of the 
input/capture clocks can then be adjusted until these glitches disappear. 
The 'adc5g' package provides a function for doing so:
```python
from adc5g import *
set_test_mode(roach, 0)
set_test_mode(roach, 1)
sync_adc(roach)
opt0, glitches0 = calibrate_mmcm_phase(roach, 0, ['scope_raw_0_snap',])
opt1, glitches1 = calibrate_mmcm_phase(roach, 1, ['scope_raw_1_snap',])
unset_test_mode(roach, 0)
unset_test_mode(roach, 1)
```
The calibration steps through all 56 phases and attempts to find the 
optimal one, i.e. the one with zero glitches furthest away from the peak. 
The function returns the optimal phase it found and an array containing 
the number of glitches detected in the ramp at each phase step.

Taking a Snapshot
-----------------
If you simply wish to take a quick snapshot you can use the following 
command with `snap_name` being the name of the snapshot block in your 
design:
```python
import numpy
raw = numpy.array(adc5g.get_snapshot(roach, snap_name))
```
If you are using the digicom bof files provided in the 'boffiles' 
directory then `snap_name="raw_%z"%z` where `z` is the ZDOK number 
of the ADC you wish to capture data from, either `0` or `1`.

Running the Test Suite
----------------------
If you want to run the full test suite from a remote machine over 
TCP/IP using tcpborphserver do the following from the root directory 
of this repository:
```bash
python2.7 test_adc5g.py -v -r roach_name 
```
where `roach_name` is the IP or hostname of the ROACH on the network.
By default this will test only the ADC in ZDOK 0 of the ROACH; to test 
the one in ZDOK 1 add `-z 1`.
If you'd like to see a list of options you can use `--help`.

Using the Stand-alone Binary
----------------------------
A stand-alone binary test-script has been compiled for the PowerPC 
platform and requires the Borph kernel to be running on the ROACH.
This binary is located in the 'bin' folder and can be run directly 
on the ROACH using:
```bash
./bin/test_adc5g -v
```
The stand-alone binary takes all the same options as the Python 
test suite since it's simply a "frozen" version of that suite.
