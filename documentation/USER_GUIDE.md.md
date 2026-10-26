# VMFForge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your web browser.  
2. In the "JTAC Digital 9-Line CAS Config" panel, enter your Originator URN (`123456`) and Destination Aircraft URN (`654321`).  
3. Select the target message (`K02.35: Close Air Support 9-Line Brief` or `K05.1: Friendly Position Report`).  
4. Enter target coordinates (Latitude e.g. `34.5281`, Longitude e.g. `69.1723`), Target Elevation (`1250 ft`), Target Type (`T-72 MBT`), and Laser Code (`1688`).  
5. Click **Transmit Digital 9-Line** to pack the bit-level Application Header, serialize K02.35 bitfields with Presence Bits, scale coordinates into 23b/24b integers, and inspect the bitstream hex dump.  
6. Click **Run Normative Audit** to evaluate all 10 MIL-STD-2045-47001 protocol compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 vmf-cli.py \--audit \--orig 123456 \--dest 654321