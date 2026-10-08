# R36S-V20-rf3536k3ka
A repository for information regarding R36S V20 clones under revision 2025-05-18 and the rf3536k3ka.dtb.

After purchasing multiple Temu-special R36S, I was cursed with receiving a R36S-V20-2025-05-18 clone. Unfortunately, this device uses the rf3536k3ka.dtb, which is not currently found at https://r36s.dpdns.org/dtbTools.html. 

There are multiple issues with this particular device and I've spent some time trying to diagnose and repair what I can. This repository will serve as a central location for anything related to this device. 

Initial Problems:
- Cannot flash SD Card to standard ARKOS installation
- Cannot run AROKS4Clones well
- Unsupported .dtb file (rf3536k3ka.dtb)
- Audio issues are present
- SD Cards cannot be recognized by file system managers (Linux & Windows)

Bonus problem:
- SD cards are, as expected at 30 USD, counterfeit and falsely marked as 128GB. They are in fact 16GB containing both an installation of ARKOS (08232024) and roms on the partitions. The ROMS partition has corrupted game files that I have not yet addressed.

(Partially) Resolved Issues
- Systems come with corrupted SD cards that cannot be viewed in file managers (partially resolved)
  - You can find documentation here on how I was able to get the SD card in a readable state.
- Stable installation of ArkOS_K36_v2.0_08112025, however the battery percentage is not accurate. Power management seems to be running fine.
  - See the ArkOS_K36-Installation for installation documentation and troubleshooting progress
