/\*\*  
 \* MIL-STD-2045-47001D/E & MIL-STD-188-220 Variable Message Format (VMF) Core Engine  
 \* Implements bit-level Application Header packing, Presence Bit management (GPB/FPB),  
 \* 23-bit/24-bit coordinate range scaling, and K-Series message serialization (K02.35 9-Line & K05.1 PPLI).  
 \*/

// VMF Urgency Levels (MIL-STD-2045-47001 Table 5-I)  
const VMF\_URGENCY \= {  
  ROUTINE: 0,  
  PRIORITY: 1,  
  IMMEDIATE: 2,  
  FLASH: 3  
};

// VMF K-Series Message Identifiers  
const VMF\_K\_MESSAGES \= {  
  K01\_1: { id: 'K01.1', name: 'Free Text Message' },  
  K02\_1: { id: 'K02.1', name: 'Air Support Request' },  
  K02\_33: { id: 'K02.33', name: 'CAS 9-Line Briefing (Legacy)' },  
  K02\_35: { id: 'K02.35', name: 'CAS 9-Line Briefing (Modern DACAS)' },  
  K05\_1: { id: 'K05.1', name: 'Friendly Position Report (PPLI / BFT)' }  
};

// Target Mark Types  
const VMF\_MARK\_TYPES \= {  
  NO\_MARK: 0,  
  SMOKE\_WHITE: 1,  
  SMOKE\_RED: 2,  
  LASER\_DESIGNATOR: 3,  
  IR\_POINTER: 4,  
  BEACON: 5  
};

class VMFEngine {  
  constructor(originatorUrn \= 123456, recipientUrn \= 654321\) {  
    this.originatorUrn \= originatorUrn; // 24-bit Unit Reference Number (URN)  
    this.recipientUrn \= recipientUrn;   // 24-bit Destination URN  
    this.messageSequence \= 0;  
  }

  getNextMessageNumber() {  
    this.messageSequence \= (this.messageSequence \+ 1\) & 0xFFFF;  
    return this.messageSequence;  
  }

  /\*\*  
   \* Scales WGS-84 Latitude (-90 to \+90 deg) into 23-bit integer  
   \* Resolution: 180 / (2^23 \- 1\) \~= 0.00002146 deg \~= 2.38 meters  
   \*/  
  static encodeLatitude23(latDeg) {  
    const clamped \= Math.max(-90.0, Math.min(90.0, latDeg));  
    const maxVal \= (1 \<\< 23\) \- 1; // 8388607  
    return Math.round(((clamped \+ 90.0) / 180.0) \* maxVal) & 0x7FFFFF;  
  }

  static decodeLatitude23(latVal23) {  
    const maxVal \= (1 \<\< 23\) \- 1;  
    return \-90.0 \+ (latVal23 / maxVal) \* 180.0;  
  }

  /\*\*  
   \* Scales WGS-84 Longitude (-180 to \+180 deg) into 24-bit integer  
   \* Resolution: 360 / (2^24 \- 1\) \~= 0.00002146 deg \~= 2.38 meters at equator  
   \*/  
  static encodeLongitude24(lonDeg) {  
    const clamped \= Math.max(-180.0, Math.min(180.0, lonDeg));  
    const maxVal \= (1 \<\< 24\) \- 1; // 16777215  
    return Math.round(((clamped \+ 180.0) / 360.0) \* maxVal) & 0xFFFFFF;  
  }

  static decodeLongitude24(lonVal24) {  
    const maxVal \= (1 \<\< 24\) \- 1;  
    return \-180.0 \+ (lonVal24 / maxVal) \* 360.0;  
  }

  /\*\*  
   \* BitBuffer helper class for writing arbitrary bitfield widths  
   \*/  
  static createBitWriter() {  
    let bits \= \[\];  
    return {  
      writeBits: function(val, numBits) {  
        for (let i \= numBits \- 1; i \>= 0; i--) {  
          bits.push((val \>\> i) & 1);  
        }  
      },  
      writeBit: function(b) {  
        bits.push(b ? 1 : 0);  
      },  
      getBytes: function() {  
        const bytes \= \[\];  
        for (let i \= 0; i \< bits.length; i \+= 8\) {  
          let byteVal \= 0;  
          for (let j \= 0; j \< 8; j++) {  
            if (i \+ j \< bits.length) {  
              byteVal \|= (bits\[i \+ j\] \<\< (7 \- j));  
            }  
          }  
          bytes.push(byteVal);  
        }  
        return bytes;  
      },  
      totalBits: function() {  
        return bits.length;  
      }  
    };  
  }

  /\*\*  
   \* Encodes MIL-STD-2045-47001 Application Header (Section 5\)  
   \*/  
  encodeApplicationHeader(urgency \= VMF\_URGENCY.IMMEDIATE, ackRequest \= 1\) {  
    const writer \= VMFEngine.createBitWriter();  
    const msgNum \= this.getNextMessageNumber();

    // Version: 0x05 (MIL-STD-2045-47001D/E) \- 4 bits  
    writer.writeBits(0x05, 4);  
    // URG: 2 bits  
    writer.writeBits(urgency & 0x03, 2);  
    // ACK: 2 bits (1 \= Machine ACK requested)  
    writer.writeBits(ackRequest & 0x03, 2);  
    // Message Number: 16 bits  
    writer.writeBits(msgNum, 16);  
    // Retransmit Indicator: 1 bit (0 \= Original)  
    writer.writeBit(0);  
    // Originator URN: 24 bits  
    writer.writeBits(this.originatorUrn & 0xFFFFFF, 24);  
    // Recipient URN Count: 1 (4 bits)  
    writer.writeBits(1, 4);  
    // Recipient URN: 24 bits  
    writer.writeBits(this.recipientUrn & 0xFFFFFF, 24);

    return {  
      writer: writer,  
      msgNum: msgNum,  
      urgency: urgency,  
      originatorUrn: this.originatorUrn,  
      recipientUrn: this.recipientUrn  
    };  
  }

  /\*\*  
   \* Serializes a K02.35 Close Air Support (CAS) 9-Line Briefing  
   \*/  
  serializeK0235Cas9Line(casData \= {}) {  
    const headerInfo \= this.encodeApplicationHeader(VMF\_URGENCY.FLASH, 1);  
    const writer \= headerInfo.writer;

    // K-Series Message Standard Identifier: K02.35 (12 bits)  
    writer.writeBits(0x0235, 12);

    // Group Presence Bit: Line 1-3 IP/Heading/Distance  
    writer.writeBit(1);  
    const headingDeg \= casData.headingDeg \|\| 145;  
    const distanceNm \= casData.distanceNm \|\| 8.5;  
    writer.writeBits(Math.round(headingDeg) & 0x1FF, 9); // Heading: 9 bits (0-360)  
    writer.writeBits(Math.round(distanceNm \* 10\) & 0x3FF, 10); // Distance: 10 bits (0.1 NM res)

    // Line 4: Target Elevation in feet MSL (14 bits, 0-16383 ft)  
    const elevationFt \= casData.elevationFt \|\| 1250;  
    writer.writeBit(1); // Field Presence Bit  
    writer.writeBits(Math.round(elevationFt) & 0x3FFF, 14);

    // Line 5: Target Description (Code: 1 \= Tank, 2 \= Bunkers, 3 \= Troops, 4 \= Artillery)  
    const targetDescCode \= casData.targetDescCode \|\| 1;  
    writer.writeBits(targetDescCode & 0x0F, 4);

    // Line 6: Target Coordinates WGS-84 (Lat 23b, Lon 24b)  
    const targetLat \= casData.targetLat \|\| 34.5281;  
    const targetLon \= casData.targetLon \|\| 69.1723;  
    const lat23 \= VMFEngine.encodeLatitude23(targetLat);  
    const lon24 \= VMFEngine.encodeLongitude24(targetLon);  
    writer.writeBit(1); // Field Presence  
    writer.writeBits(lat23, 23);  
    writer.writeBits(lon24, 24);

    // Line 7: Mark Type (Laser 3, Smoke 1, etc.)  
    const markType \= casData.markType \|\| VMF\_MARK\_TYPES.LASER\_DESIGNATOR;  
    writer.writeBits(markType & 0x07, 3);  
    if (markType \=== VMF\_MARK\_TYPES.LASER\_DESIGNATOR) {  
      writer.writeBits(casData.laserCode \|\| 1688, 12); // 12-bit PRF laser code  
    }

    // Line 8: Friendlies Distance (10 bits) & Bearing (9 bits)  
    writer.writeBit(1);  
    writer.writeBits(Math.round((casData.friendliesDistNm \|\| 1.8) \* 10\) & 0x3FF, 10);  
    writer.writeBits(Math.round(casData.friendliesBearingDeg \|\| 270\) & 0x1FF, 9);

    // Line 9: Egress Direction (Cardinal 0=N, 1=NE, 2=E, 3=SE, 4=S, 5=SW, 6=W, 7=NW)  
    writer.writeBits(casData.egressCode \|\| 4, 3); // South

    const bytePayload \= writer.getBytes();

    return {  
      messageId: VMF\_K\_MESSAGES.K02\_35.id,  
      name: VMF\_K\_MESSAGES.K02\_35.name,  
      msgNumber: headerInfo.msgNum,  
      originatorUrn: this.originatorUrn,  
      recipientUrn: this.recipientUrn,  
      totalBits: writer.totalBits(),  
      totalBytes: bytePayload.length,  
      coordinates: { lat: targetLat, lon: targetLon, lat23: lat23, lon24: lon24 },  
      targetElevationFt: elevationFt,  
      wireBytes: bytePayload,  
      hexDump: bytePayload.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
    };  
  }

  /\*\*  
   \* Serializes a K05.1 Friendly Position Report (PPLI / Blue Force Tracking)  
   \*/  
  serializeK051PositionReport(posData \= {}) {  
    const headerInfo \= this.encodeApplicationHeader(VMF\_URGENCY.ROUTINE, 0);  
    const writer \= headerInfo.writer;

    // K-Series ID: K05.1  
    writer.writeBits(0x0501, 12);

    const lat \= posData.latitude \|\| 34.5200;  
    const lon \= posData.longitude \|\| 69.1600;  
    const lat23 \= VMFEngine.encodeLatitude23(lat);  
    const lon24 \= VMFEngine.encodeLongitude24(lon);

    writer.writeBits(lat23, 23);  
    writer.writeBits(lon24, 24);  
    writer.writeBits(Math.round(posData.speedKts \|\| 250\) & 0x3FF, 10);  
    writer.writeBits(Math.round(posData.courseDeg \|\| 180\) & 0x1FF, 9);

    const bytePayload \= writer.getBytes();

    return {  
      messageId: VMF\_K\_MESSAGES.K05\_1.id,  
      name: VMF\_K\_MESSAGES.K05\_1.name,  
      msgNumber: headerInfo.msgNum,  
      totalBits: writer.totalBits(),  
      totalBytes: bytePayload.length,  
      wireBytes: bytePayload,  
      hexDump: bytePayload.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
    };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    VMFEngine,  
    VMF\_URGENCY,  
    VMF\_K\_MESSAGES,  
    VMF\_MARK\_TYPES  
  };  
}  
