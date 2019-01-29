# Toyota_TPMS Signal Decoding

## Decodings of available signals of this type

- [Example Set](ANALYSIS/README.md)
- [Example Set 02](02/ANALYSIS/README.md)
- [Example Set 03](03/ANALYSIS/README.md)
- [Example Set 04](04/ANALYSIS/README.md)

## Device

Seen on a Toyota Auris (Corolla).

|||
|:-|:-|
| Device Type | TPMS Device |
| Company | Pacific Industrial Co., Ltd. |
| Manufacturer | Pacific Industrial Co., Ltd. (Pacific Manufacturing Ohio, Inc.) |
| Model | PMV-C210 |
| Date of FCC Application | 2010-03-19 |
| Toyota Part Number | 42607-02031 |


## References:

- [rtl_433/src/devices/tpms_toyota.c](https://github.com/merbanan/rtl_433/blob/master/src/devices/tpms_toyota.c)
- [Toyota TPMS sensor #706](https://github.com/merbanan/rtl_433/issues/706)
- [RAV4.4 - Receiving and Decoding TPMS Valve Data](http://forum.rav4driversclub.com/thread/74/rav4-receiving-decoding-tpms-valve)
- [FCC ID PCX-PMV-C210](https://fccid.io/PCX-PMV-C210)

## Approximate Signal Info

|||
|:-|:-|
| Frequency          | 433-434 MHz         |
| Modulation         | FSK                 |
| Deviation          | ?                   |
| Symbol Bit Rate    | 20,000 bps          |
| Symbol Length      | 50 μs               |
| Symbols Per Signal | 160                 |
| Frames Per Signal  | 1                   |

## Decoding

There are 160 symbols in the signal:

Assume the higher mark frequency of the FSK signal is a "1" symbol, and the lower space frequency is a "0" symbol.

|||
|:-|:-|
| Preamble             | 8 symbols: 01010101        | 
| Start Delimiter      | 6 symbols: 001111          | 
| Data Symbols         | 146 symbols                |

Encoding is 0-Transition Differential Manchester Coding.  This variation is essentially Bi-Phase Space Coding ("BP-S" or "FM0") and defines a signal 
transition when encoding the logical "0" bits, no transition for logical "1" bits, between logical bit periods.

Each logical bit period consists of pair of two symbol bits, including the knowledge of whether or not a transition exists immediately before the period. 
There will always be a transition between the pair of symbols by definition.  The transition immediately before the period is the data transition and
the transition in the middle of the period is the clock transition.

The decoded logical data bit of the period is determined by the presence ("0") or absense ("1") of a transition immediately before the first symbol 
of the period.

There exists a Differential Manchester decoding efficiency if three things are known about the signal:

1. The initial symbol transition is to be considered a mid-period clock transition.

2. That initial decoded period will result in a logical data bit with a predetermined relationship to the initial symbol bit (instead of an
   unknown existence or not of a transition before the period).

3. The final transition before the end of the signal must be a clock transition in order to result in a decoded logical bit for that period.

These requirements are met in this signal and it is likely not a coincidence.  In this scenario, the resulting decoded data will have a length equal to 
the number of clock transitions found within the signal (the beginning and end of the signal itself are not considered clock transitions).  This is also
why although there are 146 symbols in the signal, there will be only 72 decoded logical bits, not 73, as there are only 72 clock transitions inside this 
signal.  And, due to our predetermined selection of "0-Transition Differential Manchester Coding" (we'll refer to this as "BP-S"), the predetermined 
relationship to the initial symbol bit to the initial decoded logical data bit is to be the inverse value.  Thus, our initial symbol bit in the will
always be "0" (and it must be due to our known Start Delimiter) and our initial decoded logical data bit will always be "1".

To understand this algorithm, it is helpful to consider how the Universal Radio Hacker functions decode a Differential Manchester encoded signal.

## Universal Radio Hacker Interpretation and Decoding:

| Decoder Function    | Information and Options   | Description |
|:-                   |:-                         |:-           |
| Cut before/after    | Cut Before Sequence: 1111 | Remove symbols before the 1111 in the start delimiter |
| Cut before/after    | Cut Before Positon: 4     | Remove the 1111 |
| Cut before/after    | Cut After Position: 145    | Result is 146 bits in length
| Edge Trigger        |                           | Bi-Phase Manchester I, 01 -> 1 and 10 -> 0 
| Differential Encoding |                         | First bit copied as is, subsequent transition -> 1, no transition -> 0

Note that the combination of the Edge Trigger and Differential Encoding functions in combination essentially result in a BP-S 
decoding.  The success of this algorithm to properly decode our data takes advantage of the fact that the first and last 
transitions in the symbol data (not into and out of the signal itself) are clock transitions and that the first transition is 
`0` to `1` (and not a `1` to `0`) resulting in a final decoded logical bit of '1'.

For example:

```
0011001100101100101010101101001011   Original symbol bits (34 symbol bits)
  1 0 1 0 1 1 0 1 1 1 1 1 0 0 1 1    Result of Edge Trigger function (16 logical data bits)
  1 1 1 1 1 0 1 1 0 0 0 0 1 0 1 0    Result of Differential Encoding function (16 logical data bits)
```

## Data Interpretation

After decoding data symbols using BP-S coding, we're left with a logical data message of 72 bits (9 bytes)

| Offset Bit | Length | Description | Value           | Notes |
|:-----------|:-------|:------------|:----------------|:------|
| 0          | 4      | FixedValue  | 1111  (0xf)     | Always 1111 in observed signals and not technically considered part of Serial ID |
| 4          | 28     | Serial_ID   | *               | Sensor Serial ID is 7 hex digits and is printed on TPMS sensor |
| 32         | 1      | Status_80   | 1               | Unknown (1 = Battery OK?)
| 33         | 8      | Pressure    | 4 * ( PSI + 7 ) | PSI = ( Value / 4 ) - 7 |
| 41         | 8      | Temperature | Celsius + 40    | Celsius = Value - 40, resulting in a range from -40C to +215C |
| 49         | 1      | Status_40   | 0               | Unknown
| 50         | 1      | Status_20   | 0               | Unknown
| 51         | 1      | Status_10   | 0               | Unknown
| 52         | 1      | Status_08   | 0               | Unknown
| 53         | 1      | Status_04   | 0               | Unknown (1 = Rapid Deflation?)
| 54         | 1      | Status_02   | 0               | Unknown
| 55         | 1      | Status_01   | 0               | Unknown
| 56         | 8      | PressureInv | Pressure ^ 0xff | Pressure as inverted value |
| 64         | 8      | CRC         | *               | CRC8 over bits 0-63, 0x07 truncated polynomial, initial value 0x80 |
