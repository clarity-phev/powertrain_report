# powertrain_report
Generate an Electric Powertrain Report for a Honda Clarity

The purpose of this code is to communicate with the Clarity through the OBD2 port, and read parameters associated with the Electric Powertrain.  These parameters include the HV Battery Capacity, as well as detailed information about each individual battery cell (2 Banks of 84 cells).

This method is very convenient because it produces a 'written' record of the battery measurements every time it is used.  The log files produced are timestamped, and can be preserved and re-processed later.  The output reports may also be saved as .pdf documents.

Any comments / contributions are welcome.

Credits:
This started as a collaborative effort on the Honda Clarity Forum here: https://www.insideevsforum.com/community/index.php?forums/clarity.53/
between users @MrFixit, and @lincomatic.  The original concept for this was done in 'c++' by @lincomatic and ported to this web-based version by @MrFixit.

