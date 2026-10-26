# MIL-STD-2045-47001D/E & MIL-STD-188-220 (VMF) Technical Architecture

## 1\. Executive Summary & Protocol Overview

Variable Message Format (VMF), standardized under MIL-STD-2045-47001 and MIL-STD-188-220, defines a bit-oriented, connectionless tactical message protocol. Designed for tactical battlefield command, control, communications, and close air support, VMF achieves high data efficiency over bandwidth-limited Combat Net Radio (CNR) networks.

### Key Architectural Concepts:

1. **Application Header Structure (MIL-STD-2045-47001 Section 5\)**:  
   - `Version` (4 bits): Identifies standard edition (e.g. 0x05 for D/E).  
   - `Urgency` (2 bits): Routine (0), Priority (1), Immediate (2), Flash (3).  
   - `Acknowledge Request` (2 bits): Machine ACK and Operator ACK flags.  
   - `Message Number` (16 bits): Monotonically increasing sequence tracker.  
   - `URN Addressing` (24 bits): Unit Reference Numbers uniquely identifying tactical operators and aircraft.  
2. **Bit-Oriented Syntax & Dynamic Presence Bits**:  
   - Fields are mapped down to individual bit boundaries.  
   - Employs **Group Presence Bits (GPB)** and **Field Presence Bits (FPB)** so that unused tactical parameters consume zero wire bandwidth.  
3. **K-Series Message Suite**:  
   - **K02.35 (Close Air Support 9-Line Brief)**: Standardized CAS digital brief conveying initial points, headings, distances, target elevations, target coordinates, marking methods, friendly locations, and egress corridors.  
   - **K05.1 (Friendly Position Report)**: Digital situational awareness (PPLI) with GPS coordinates and velocity vectors.  
   - **K01.1 (Free Text)**: Variable-length 7-bit ASCII tactical communications.  
4. **Range Scaling of Tactical Coordinates**:  
   - Compresses WGS-84 Latitude into 23 bits and Longitude into 24 bits, providing \$\\approx 2.4\\text{ m}\$ spatial resolution on the digital battlefield.