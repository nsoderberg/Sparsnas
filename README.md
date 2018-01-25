# Sparsnas
Note: This is work in progress and content may change at any time.

# Radio Signal Analysis
This section describes how to decode the radio transmission.

The IKEA Sparsnäs consists of a sensor measuring energy led blinks. It uses a TexasInstruments CC115L transmitter, and the display-enabled receiver uses a TexasInstruments CC113L.

 * [Texas Instruments CC115L](Docs/TexasInstruments.CC115L-RF.Transmitter.On.Sensor.pdf) - Transmitter datasheet
 * [Texas Instruments CC113L](Docs/TexasInstruments.CC113L-RF.Receiver.On.Display.pdf) - Receiver datasheet
 

## Recording the signal

First, you need to have some sort of Software Defined Radio (SDR) installed on your system. There are many out there; [RTL-SDR](https://www.rtl-sdr.com/rtl-sdr-blog-v-3-dongles-user-guide/), [HackRF-One](https://greatscottgadgets.com/hackrf/), [AirSpy](https://airspy.com/products/) just to name a few.

XXX TODO: insert photo of them here

Second, you need some software to record the signal. You will find any alternatives ranging from simple commandline apps to more advanced guis. The Ikea Sparsnäs sends a signal on the 868 MHz band, and here are a few alternatives to record on that band.

```
rtl_sdr -f 868000000 -s 1024000 -g 40 - > outfile.cu8

hackrf_transfer -r outfile.cs8 -f 868000000 -s 2000000

osmocom_fft -a airspy -f 868000000 -v
```

This text will not go into the details on how to install them. This text will continue assuming that you managed to record a signal to file on disk using one of the command lines above. Note: different applications stores the signal data in different formats such as *.cu8, *.cs8, *.cu16, *.cfile, etc. Common to all these formats is the sample form called ["IQ"](http://whiteboard.ping.se/SDR/IQ).

##### Open the recorded signal file in a graphical interface for analysis

 There are many different techniques and softwares for doing signal analysis. Here we will be using [Inspectrum](https://github.com/miek/inspectrum) and [DspectrumGUI](https://github.com/tresacton/dspectrumgui).

```
 inspectrum -r SampleRateInHz Filename.Ext
 inspectrum -r 1024000        outfile.cu8
```

Now begins the process of locating the signal. Browsing the horizontal timeline we finally find the following colorful signal:
![Locating the signal](Docs/00.Start.by.scrolling.until.we.find.some.signal.png?raw=true "Locating the signal")

By modifying the sliders on the left in the GUI, we can zoom in into the signal. The Y-axis is the 'FFT size' and the X-axis is the 'Zoom' which corresponds to the timeline of your signal recording. By doing this, we end up with the following result:

![Zooming into the signal](Docs/01.Isolate.the.FSK.signal.png?raw=true "Zooming into the signal")
The recording frequency of 868 MHz is represented at zero on the Y-axis. The short signal in the middle is an artifact called "DC-Spike" which is an anomaly generated by the SDR-device. But here we see two frequencies along the recording frequency. This is typical for signal modulation type ["FSK"](https://en.wikipedia.org/wiki/Frequency-shift_keying). Lets zoom in even further:

![Zooming into the signal ever further](Docs/02.Zoom.in.and.identify.upper.and.lower.FSK.frequencies.png?raw=true "Zooming into the signal ever further")
Now this certainly looks like a FSK-signal.

By right-clicking on the FSK-signal in Inspectrum we can do a "Frequency Plot":

![Right-clicking and enable frequency plot](Docs/03.RightClick.and.enable.frequency.plot.png?raw=true "Right-clicking and enable frequency plot")
![Frequency Plot](Docs/04.Frequency.plot.png?raw=true "Frequency Plot")

The frequency plot (see the green-lined graph above) show how the two frequencies changes and creates binary 1's and 0's. Right-clicking on the frequency plot graph enables us to do a "Threshold Plot" which modifies the somewhat buzzy frequency plot into a nice binary form:

![Right-clicking and enable threshold plot](Docs/05.RightClick.and.enable.threshold.plot.png?raw=true "Right-clicking and enable threshold plot")
![Threshold Plot](Docs/06.Inspect.the.binary.stream.png?raw=true "Threshold Plot")

Next up in the analysis is to determine signal characteristics like data rate of the 1's and 0's. Inspectrum helps us with that by using "cursors" which are enabled by hitting the checkbox on the left in the gui. We can here graphically position the cursors such that they align as perfectly as possible with the edges of the 1's and 0's. We do this for as long as we have what seems as valid data, ending up with 208 symbols (i.e. 1's or 0's).

![Position cursors to define the whole message](Docs/08.Position.cursors.to.define.the.whole.message.png?raw=true "Position cursors to define the whole message")

Now, if we look in the reference documentation of the transmitter CC115L, we find this packet description:
![Lookup packet format in the TI-CC113L documentation](Docs/09.Lookup.packet.format.in.the.TI-CC113L.receiver.docs.png?raw=true "Lookup packet format in the TI-CC113L documentation")

So, our green binary bit-stream should fit into this packet.

Working with binary streams can be inefficient. A more preferable form is hexadecimal. We could start counting the binary string, 8-bits at a time, but instead we use the application DspectrumGui which automates that process for us. Right-click and send the data to stdout. (For this to work it is required that you have started Inspectrum from within DspectrumGUI).
![RightClick and send symbol data back to DspectrumGui](Docs/10.RightClick.and.send.symbol.data.back.to.DspectrumGui.png?raw=true "RightClick and send symbol data back to DspectrumGui")
![Review the decoded binary stream in the DspectrumGUI](Docs/11.Review.the.decoded.binary.stream.in.the.DspectrumGui.png?raw=true "Review the decoded binary stream in the DspectrumGUI")

Here in DspectrumGUI we see the binary stream and the "Raw Binary To Hex" conversion. Now its easier to map the data into the packet format:

![Mapping the values from DspectrumGUI to the Texas Instruments packet format](Docs/12.Mapping.the.values.from.DspectrumGUI.to.the.receivers.defined.paket.format.png?raw=true "Mapping the values from DspectrumGUI to the Texas Instruments packet format")

We now want to verify that our analysis is correct. We do this by looking up the CRC-algorithm in the Texas Instruments documentation and test our values:

![Look up the CRC algorithm used in the Texas Instruments documentation and test the values](Docs/13.Look.up.the.CRC.algorithm.used.in.TI-docs.and.test.the.values.png?raw=true "Look up the CRC algorithm used in the Texas Instruments documentation and test the values")

We do a quick implementation of the algorithm in an online c++ compiler/debugger environment, and when executing it we end up with "crc checksum: 0x1204" which matches the expected crc value.

We can now go on to the next step in the analysis which is recording more data. Now since the sender and transmitter are of the Texas Instruments family CCxxxx, we use a usb hardware dongle called ["Yard Stick One"](https://greatscottgadgets.com/yardstickone/). It consists of a CC1111 chip which can be controlled using the Python-library RfCat.

To start doing this, we need to feed the things we have seen so far in the analysis into the CC1111-tranceiver. Here's how we do that:

![Determine CC1111 tranceiver parameters in RfCat](Docs/16.Determine.CC1111.tranceiver.parameters.in.RfCat.png?raw=true "Determine CC1111 tranceiver parameters in RfCat")

By starting RfCat, defining the function init(d) and calling it we have configured the CC1111-chip. To start listening we call d.RFlisten() and as you can see we start to get some packets.

However, to be able to test the packet content better, we write a small Python script. Take a moment to read it, in order to get an understanding of whats going on:

```python
#=============================================================================
# Ikea Sparsnas packet decoder using Yard Stick One along with RfCat
#=============================================================================
import sys
import readline
import rlcompleter
readline.parse_and_bind("tab: complete")
from rflib import *                           

#-----------------------------------------------------------------------------
#------------------------------ Global variables -----------------------------
#-----------------------------------------------------------------------------
d = RfCat()
Verbose = False


#-----------------------------------------------------------------------------
# Initialize radio
#-----------------------------------------------------------------------------
def init(d):
    d.setFreq(868000000)            # Main frequency
    d.setMdmModulation(MOD_2FSK)    # Modulation type
    d.setMdmChanSpc(40000)          # Channel spacing
    d.setMdmDeviatn(20000)          # Deviation
    d.setMdmNumPreamble(32)         # Number of preamble bits
    d.setMdmDRate(38391)            # Data rate
    d.setMdmSyncWord(0xD201)        # Sync Word
    d.setMdmSyncMode(1)             # 15 of 16 bits must match
    d.makePktFLEN(20)               # Packet length
    d.setMaxPower()

#-----------------------------------------------------------------------------
# Crc16 helper
#-----------------------------------------------------------------------------
def culCalcCRC(crcData, crcReg):
    CRC_POLY = 0x8005
    
    for i in xrange(0,8):
        if ((((crcReg & 0x8000) >> 8) ^ (crcData & 0x80)) & 0XFFFF) :
            crcReg = (((crcReg << 1) & 0XFFFF) ^ CRC_POLY ) & 0xFFFF
        else:
            crcReg = (crcReg << 1) & 0xFFFF
        crcData = (crcData << 1) & 0xFF
    return crcReg

#-----------------------------------------------------------------------------
# crc16
#-----------------------------------------------------------------------------
def crc16(txtBuffer, expectedChksum):
    CRC_INIT = 0xFFFF
    checksum = CRC_INIT

    hexarray = bytearray.fromhex(txtBuffer)
    for i in hexarray:
        checksum = culCalcCRC(i, checksum)

    if checksum == int(expectedChksum, 16):
        #print "(CRC OK)"
        return True
    else:
        #print "(CRC FAIL) Expected=" + expectedChksum + " Calculated=" + str(hex(checksum))
        return False

#-----------------------------------------------------------------------------
# "main"
#-----------------------------------------------------------------------------
print "Initialize modem..."
init(d)

print "Waiting for packet..."
#d.RFlisten()

#-----------------
# Read packet loop
#-----------------
while True:
    capture = ""
    
    #---------------------------------
    # Wait for a packet to be captured
    #---------------------------------
    try:
        y,z = d.RFrecv()
        capture = y.encode('hex')
        #print capture

    except ChipconUsbTimeoutException:
        pass

    #------------------------
    # When we got a packet...
    #------------------------
    if capture:

        # Extract packet content to the formal TexasInstruments packet layout
        pkt_length  = capture[0:0+2]
        pkt_address = capture[2:2+2]
        pkt_data    = ""
        for x in xrange(4, len(capture) - 4, 2):
            currElement = capture[x:x+2]
            pkt_data += currElement + " "
        pkt_crc     = capture[36:36+2] + " " + capture[38:38+2]

        # Verify crc16 accordingly to the TexasInstruments implementation
        crcBuf_str  = (pkt_length + pkt_address + pkt_data).replace(" ","")
        expectedCrc = capture[36:36+2] + capture[38:38+2]
        crcOk       = crc16(crcBuf_str, expectedCrc)

        if Verbose:
            print "pkt.length        = " + pkt_length
            print "pkt.address_field = " + pkt_address
            print "pkt.data_field    = " + pkt_data
            print "pkt.crc16         = " + pkt_crc + " (CRC verification: " + str(crcOk) + ")"
            print ""
        else:
            if crcOk:
                print "Pkt: " + pkt_length + " ",
                print  pkt_address + " ",
                print  pkt_data.replace(" ","") + " ",
                print  pkt_crc.replace(" ","")
```

When we run the script and start to get some data, we quickly identify that the packet content does not match what is shown on the receiving display. We can therefore conclude that the packet content is scrambled in some way. However, since the sensor is a small battery powered device with limited computational resources it is a fair assumption that we're dealing with some kind of simplistic XOR obfuscation of sorts.

![First attempt to look for patterns in packet content](Docs/17.First.attempt.to.look.for.patterns.in.packet.content.png?raw=true "First attempt to look for patterns in packet content")
At this point, we know nothing of the internal packet layout, but we can start to identify patterns. This is a creative process which can be time consuming. First we need to list possible entities that may, or not may, be in the Data Field-part of the signal.

### Constants

 * Variable identifiers (such as data for variable X is always prepended with the constant Y)
 * Length fields (packet length, length of individual fields in the packet, etc)
 * Sync words or other "magics" 
 * Sender identifiers (addresses, serials, hardware or software versions/revisions)
 * etc

### Variables

 * Timestamps
 * Things we (in this case) see on the display
   * Current power usage seen on the display
   * Accumulative power consumption seen on the display
   * Battery life properties
 * Signal strength/RSSI if we're dealing with a two-way communication protocol
 * Extra crc's or other hashes
 * etc

The list goes on an on, but lets start with those elements for now. When identifying element-patterns we need to control the signal being sent as much as possible. Therefore we build a simple led-blinker with an Arduino board. The led is flashing at a predetermined rate which we control. Knowing the stable flash rate we can observe what kWh they translate into on the receiving display. This is a good starting point for our analysis. Other things we may consider could be to hook up the sensor to a voltage cube and vary the transmitters battery voltage. A third option is to purchase several Sparsnäs devices, and decode signals from the different senders which may have different sender properties or identifiers. 

## Led blink helper tool
<img src="LedBlinkerHelperTool/LedFlasher.png"  width="200" />
<img src="LedBlinkerHelperTool/LedFlasher2.jpg"  width="246" />

You can find the source code [here](LedBlinkerHelperTool/LedFlasher.ino).

We hook up the Sparsnäs sensor to the red led on the right in the image above. Using the yellow and green push buttons we can increase or decrease the delay between led blinks, allowing us to experiment while running our RfCat on the side.

## Experiment 1: Finding counters
![Experiment 1](Docs/Experiment1.jpg?raw=true "Experiment 1")
In the first experiment, we isolate the sensor in total darkness (using some black electrical tape). Any changing fields would not be related to measured data, but rather counters such as unique packet identifiers, timestamps etc. In this case, we use a sender with ID 400-565-321, and by looking at the hexdump we can identify some patterns. To better view them, we insert spaces to form columns.

```
 len  ID  Cnt Fix  Fixed    Cnt2 Data Fixed      Crc16
 11   49   00 070f a276170e cfa2 8148 47cfa27ed3 f80d
 11   49   01 070f a276170e cfa3 8148 47cfa27ed3 6e0e
 11   49   02 070e a276170e cfa0 c6b7 47cfa27ed3 be8c
 11   49   04 070f a276170e cfa6 6db7 47cfa27ed3 6a3d
 11   49   05 070f a276170e cfa7 6877 47cfa27ed3 f9a2
 11   49   06 070f a276170e cfa4 6437 47cfa27ed3 4f25
 11   49   07 070f a276170e cfa5 60f7 47cfa27ed3 5da9
 11   49   08 070f a276170e cfaa 5cb7 47cfa27ed3 302e
 11   49   09 070f a276170e cfab 5b77 47cfa27ed3 2192
 11   49   0a 070f a276170e cfa8 5737 47cfa27ed3 9715
 11   49   0b 070f a276170e cfa9 53f7 47cfa27ed3 8599
 11   49   0c 070f a276170e cfae 4fb7 47cfa27ed3 7a1e
 11   49   0d 070f a276170e cfaf 4a77 47cfa27ed3 e981
 11   49   0e 070f a276170e cfac 4637 47cfa27ed3 5f06
 11   49   0f 070f a276170e cfad 42f7 47cfa27ed3 4d8a
 11   49   10 070f a276170e cfb2 3eb7 47cfa27ed3 8408
 11   49   11 070f a276170e cfb3 3d77 47cfa27ed3 11f7
 11   49   12 070f a276170e cfb0 3937 47cfa27ed3 2ff3
 11   49   13 070f a276170e cfb1 35f7 47cfa27ed3 b5fc
 11   49   14 070f a276170e cfb6 31b7 47cfa27ed3 53fb
 11   49   15 070f a276170e cfb7 2c77 47cfa27ed3 d9e4
 11   49   16 070f a276170e cfb4 2837 47cfa27ed3 e7e0
 11   49   17 070f a276170e cfb5 24f7 47cfa27ed3 7def
 ..   ..   .. .... ........ .... .... .......... ....
 ..   ..   .. .... ........ .... .... .......... ....
 ..   ..   .. .... ........ .... .... .......... ....
 11   49   40 070f a276170e cfe2 8ab7 47cfa27ed3 5b53
 11   49   41 070f a276170e cfe3 8977 47cfa27ed3 ceac
 11   49   42 070f a276170e cfe0 8537 47cfa27ed3 782b
 11   49   43 070f a276170e cfe1 81f7 47cfa27ed3 6aa7
 11   49   44 070f a276170e cfe6 8177 47cfa27ed3 082f # Column 'Data' becomes stable
 11   49   45 070f a276170e cfe7 8177 47cfa27ed3 9e2c
 11   49   46 070f a276170e cfe4 8177 47cfa27ed3 a42c
 11   49   47 070f a276170e cfe5 8177 47cfa27ed3 322f
 11   49   48 070f a276170e cfea 8177 47cfa27ed3 e02f
 11   49   49 070f a276170e cfeb 8177 47cfa27ed3 762c
 ..   ..   .. .... ........ .... .... .......... ....
 ..   ..   .. .... ........ .... .... .......... ....
 ..   ..   .. .... ........ .... .... .......... ....
 11   49   7a 070f a276170e cfd8 8177 47cfa27ed3 ec26
 11   49   7b 070f a276170e cfd9 8177 47cfa27ed3 7a25
 11   49   7c 070f a276170e cfde 8177 47cfa27ed3 9826
 11   49   7d 070f a276170e cfdf 8177 47cfa27ed3 0e25
 <this packet (7e) was lost in sniffing>
 11   49   7f 070f a276170e cfdd 8177 47cfa27ed3 a226
 11   49   00 070f a276170e cf22 8177 47cfa27ed3 5302 # Column 'Cnt' wraps
 11   49   01 070f a276170e cf23 8177 47cfa27ed3 c501
 11   49   02 070f a276170e cf20 8177 47cfa27ed3 ff01
 11   49   03 070f a276170e cf21 8177 47cfa27ed3 6902
 11   49   04 070f a276170e cf26 8177 47cfa27ed3 8b01
 11   49   05 070f a276170e cf27 8177 47cfa27ed3 1d02
 11   49   06 070f a276170e cf24 8177 47cfa27ed3 2702
 ```

* This experiment results in
    * Len = This column matches the number of payload bytes. In the Texas Instruments-case, the payload starts with the column after the length column (namely the 'ID' column) and ends where the CRC16 column begins.
    * ID = The signal analysis we performed in DspectrumGUI (previously) was done using a different sensor. Here we see that the 2nd byte is changed when we're using another sensor. We assume that this is some sort of sensor ID, and therefore name the column 'ID'.
    * We find what looks like two counters and name them 'Cnt'. 
        * As for the first, it isn't scrambled and continues to increase until it reaches 0x7F. Then it restarts at 0x00 again.
        * The second 'Cnt2' is scrambled, and its easily mixed with the column next to it named 'Data'. However, when scrolling down until packet 0x45, we see that the 'Data' column stabillizes at '8177'. This makes it very likely we're dealing with two separate columns.
    * The remaining columns contain fixed values and we leave them as is (for now).
    * Another important finding:
        * Power-cycling the sensors' battery, will make the sequences repeat *exactly* as the previous testrun.
        -> This makes our analysis much more doable.

* We concluded earlier that it is likely that we're dealing with some sort of XOR-obfuscation. Therefore; it is a good time to review the characteristics of the XOR-operation (denoted with the ^ character). Some handy facts:

    * Fact 01: Any byte ^ 0x00 will result in the original value. That is, 0xAA ^ 0x00 = 0xAA. We can use this fact when identifying counters which starts from zero and then increases. Lets say we have a 32-bit counter (i.e. 4 bytes) which we find it reasonably that is starts from zero, but is XOR'ed with an unknown key:

    | Packet ID  | Clear text before send | XOR'ed data read in the air  |
    | ---------- |:----------------------:| -----:|
    | packet 01  | 00 00 00 00 | 11 22 33 44 |
    | packet 02  | 00 00 00 01 | 11 22 33 45 |
    | packet 03  | 00 00 00 02 | 11 22 33 46 |
    | packet 04  | 00 00 00 03 | 11 22 33 47 |
    | packet 05  | 00 00 00 04 | 11 22 33 40 |

    * Knowing that a number xor'ed with 0x00 results in the original value, we can conclude that the unknown XOR-key for the packets above is 11 22 33 44.

    * Fact 02: How XOR works when dealing with increasing value series in relation to each other. Consider the following set of values:
```
    | Packet ID  | In the air  | Packet 01 XOR with Packet 0* | Packet 01   ^ Current Val = Unscrambled Cnt
    | ---------- |:-----------:| ----------------------------:| -------------------------------------------
    | packet 01  | 11 22 33 44 | --+--+--+--+                 | 11 22 33 44 ^ 11 22 33 44 = 00 00 00 00
    | packet 02  | 11 22 33 45 | <-+  |  |  |                 | 11 22 33 44 ^ 11 22 33 45 = 00 00 00 01
    | packet 03  | 11 22 33 46 | <----+  |  |                 | 11 22 33 44 ^ 11 22 33 46 = 00 00 00 02
    | packet 04  | 11 22 33 47 | <-------+  |                 | 11 22 33 44 ^ 11 22 33 47 = 00 00 00 03
    | packet 05  | 11 22 33 40 | <----------+                 | 11 22 33 44 ^ 11 22 33 48 = 00 00 00 04
```
    * In counter series starting with *zero*, xor'ing the start value with the following elements, results in the counter sequence in clear text.

### Applying what we now know about XOR to our own data

Lets assume that the 'Cnt2' counter starts at 0x000. To verify this assumption we do the following operation. Xor the starting value with each following value in the column and see what we get:

```
 len  ID  Cnt Fix  Fixed    Cnt2 Data Fixed      Crc16
 11   49   00 070f a276170e cfa2 8148 47cfa27ed3 f80d
 11   49   01 070f a276170e cfa3 8148 47cfa27ed3 6e0e
 11   49   02 070e a276170e cfa0 c6b7 47cfa27ed3 be8c
 11   49   04 070f a276170e cfa6 6db7 47cfa27ed3 6a3d
 ..   ..   .. .... ........ .... .... .......... ....
 
  cfa2 ^ cfa2 = 0000
  cfa2 ^ cfa3 = 0001
  cfa2 ^ cfa0 = 0002
  cfa2 ^ cfa6 = 0003
  .... ^ .... = ....

  Great, it looks like our assumption is valid. The XOR key for
  those two bytes in the 'Cnt2' colum is definitly 'cfa2'.
```

### Look for repetitions in the XOR-data

We now know that given the value of zero, the xor-key *at some positions* in the dataset is 'cfa2'. But if we're lucky, there can be other columns which also begins with the value of zero, but isn't counters. Lets consider the top row:

```
 len  ID  Cnt Fix  Fixed    Cnt2 Data Fixed      Crc16
 11   49   00 070f a276170e cfa2 8148 47cfa27ed3 f80d

 Can we detect the 'cfa2'-sequence anywhere else? 
```
Well, yes, we have one hit in the last 'Fixed' column. If we now make the following assumptions:
    * That column (at least at those positions) also starts with 0x000.
    * Repeating sequences indicates a rolling XOR-key, where we have a shorter key compared to the longer data to be scrambled.

Lets measure the byte-distance:
```
 len  ID  Cnt Fix  Fixed    Cnt2 Data Fixed      Crc16
 11   49   00 070f a276170e cfa2 8148 47cfa27ed3 f80d
                            ____ ____ __

We seem to have 5 bytes before the values repeat. If our assumptions are correct, this means that we have a 5-byte long XOR-key.
```

Lets measure if a 5-byte XOR-key would go fit into the packet. We saw in the long packet dump above, that the three first columns (Len, ID, Cnt) was most likely in clear text. So, the scrambled data begins with the first 'Fix' column and ends where the Crc16 begins. Attempt to fit a 5-byte XOR-key based on that:
```
 len  ID  Cnt Fix Fixed   Cnt2DataFixed      Crc16
 11   49   00 070fa276170ecfa2814847cfa27ed3 f80d
              |-5-byte-||-5-byte-||-5-byte-|

It aligns perfectly, which strengthens our assumption.
```

The assumption is now that we have the following XOR-key: ??cfa2???? 

## Experiment 2: Controlling input data
![Experiment 2](Docs/Experiment2.jpg?raw=true "Experiment 2")
Next up is to use our Arduino-based "Led blink helper tool" we built earlier to see what happens to the columns. We remove the black electrical tape and attach the sensor to the led on the breadboard. Looking at the packet dump above we can very easily conclude that the sensor sends one packet every 15'th second. If we configure our helper-tool to blink once every minute, we would have four packets per blink. (To reduce space I have removed duplicate 'NewCnt' packets. Thats why the 'Cnt' column isn't sequential.)

```
Len ID Cnt Fix Fixed    PCnt Data NewCnt      Crc16
 11 49 00 070f a276170e cfa2 8148 47cfa27e d3 f80d
 11 49 02 070e a276170e cfa0 c6b7 47cfa27e d3 be8c
 11 49 04 070e a276170e cfa6 99eb 47cfa27f d3 b625
 11 49 08 070e a276170e cfaa 916f 47cfa27c d3 bc2b
 11 49 0d 070e a276170e cfaf 8ef3 47cfa27d d3 4a4d
 11 49 0f 070e a276170e cfad 9129 47cfa27a d3 5a6a
 11 49 19 070e a276170e cfbb 917d 47cfa278 d3 223c
 11 49 1b 070e a276170e cfb9 9179 47cfa279 d3 e83a
 11 49 22 070e a276170e cf80 9161 47cfa276 d3 0c2b
 11 49 24 070e a276170e cf86 9167 47cfa277 d3 6e2d
 11 49 27 070e a276170e cf85 9161 47cfa274 d3 ce28
 11 49 2c 070e a276170e cf8e 914d 47cfa275 d3 6206
 11 49 31 070e a276170e cf93 8ecf 47cfa272 d3 807b
 11 49 33 070e a276170e cf91 9117 47cfa273 d3 f45f
 11 49 37 070e a276170e cf95 8e83 47cfa270 d3 5830
 11 49 3b 070e a276170e cf99 9101 47cfa271 d3 5848
```

### Initial observations
* The firsted fixed column is now 070e instead of 070f. This could be a status-field which indicates that the sensor is receiving led-blinks. One could speculate that either 070e or 070f represents a TRUE/FALSE value.
* The 'NewCnt' and the following value d3 seems to be different columns, since d3 is constant and NewCnt behaves like a 32-bit counter, so we space them apart.

### Power cycling
Now lets pull the battery out and put it back in again. The interesting thing is to determine whether any values are persistent over the power cycle, or if they are being reset back to default values. At the same time, we take our assumption of a 5-byte XOR-key length into account. Watch what happens:

```
First captured packet after power cycling:

Len ID Cnt Fix Fixed    PCnt Data NewCnt      Crc16
 11 49 00 070f a276170e cfa2 8148 47cfa27e d3 f80d
          |-5-bytes-||--5-bytes-| |-5-bytes-|
```
The first packet is identical to the one previous to the power cycling. Now, having determined that the 'NewCnt' column is a counter, and its being reset on boot, we can make the assumtion that NewCnt starts with the value zero. We can test if this assumption is reasonably.
Remember what we stated in "Fact 01" earlier, that is, 0xAA ^ 0x00 = 0xAA. This would mean that we have the have the 4 first bytes XOR-key if our assumption is correct. The remaining value 'd3' remains to be figured out. Enough, lets test our assumption:

```
Len ID Cnt Fix Fixed    PCnt Data NewCnt      Crc16      Xor-Key      NewCnt      Result
 11 49 00 070f a276170e cfa2 8148 47cfa27e d3 f80d     # 47cfa27e  ^  47cfa27e  = 00000000 
 11 49 04 070e a276170e cfa6 99eb 47cfa27f d3 b625     # 47cfa27e  ^  47cfa27f  = 00000001
 11 49 08 070e a276170e cfaa 916f 47cfa27c d3 bc2b     # 47cfa27e  ^  47cfa27c  = 00000002
 11 49 0d 070e a276170e cfaf 8ef3 47cfa27d d3 4a4d     # 47cfa27e  ^  47cfa27d  = 00000003
 11 49 0f 070e a276170e cfad 9129 47cfa27a d3 5a6a     # 47cfa27e  ^  47cfa27a  = 00000004
 11 49 19 070e a276170e cfbb 917d 47cfa278 d3 223c     # 47cfa27e  ^  47cfa278  = 00000006
 11 49 1b 070e a276170e cfb9 9179 47cfa279 d3 e83a     # 47cfa27e  ^  47cfa279  = 00000007
 11 49 22 070e a276170e cf80 9161 47cfa276 d3 0c2b     # 47cfa27e  ^  47cfa276  = 00000008
 11 49 24 070e a276170e cf86 9167 47cfa277 d3 6e2d     # 47cfa27e  ^  47cfa277  = 00000009
 11 49 27 070e a276170e cf85 9161 47cfa274 d3 ce28     # 47cfa27e  ^  47cfa274  = 0000000A
 11 49 2c 070e a276170e cf8e 914d 47cfa275 d3 6206     # 47cfa27e  ^  47cfa275  = 0000000B
 11 49 31 070e a276170e cf93 8ecf 47cfa272 d3 807b     # 47cfa27e  ^  47cfa272  = 0000000C
 11 49 33 070e a276170e cf91 9117 47cfa273 d3 f45f     # 47cfa27e  ^  47cfa273  = 0000000D
 11 49 37 070e a276170e cf95 8e83 47cfa270 d3 5830     # 47cfa27e  ^  47cfa270  = 0000000E
 11 49 3b 070e a276170e cf99 9101 47cfa271 d3 5848     # 47cfa27e  ^  47cfa271  = 0000000F
```

Well, what a nice counter! Thus, we can (with some certainty) conclude the XOR-key being:
```
    47 cf a2 7e ??
```
Now we only need to figure out the last byte. The bytes affected by the missing XOR-byte are:

```
Len ID Cnt Fix Fixed    PCnt Data NewCnt      Crc16
 11 49 00 070f a276170e cfa2 8148 47cfa27e d3 f80d
                   ^^          ^^          ^^
          |-5-bytes-||--5-bytes-| |-5-bytes-|

Testing an unscramble operation by XOR'ing the first packet with the assumed XOR-key:

Len ID Cnt Fix Fixed    PCnt Data NewCnt      Crc16
 11 49 00 070f a276170e cfa2 8148 47cfa27e d3 f80d
          47cf a27e??47 cfa2 7e?? 47cfa27e ??
 -------------------------------------------------
          40C0 0008??49 0000 FF?? 47cfa27e ??
```
The first and last values (17 & d3) have been static during our whole analysis so far. This makes them very dificult to work with. However, the byte in the middle column 'Data', with the value of 48, has been fluctuating since we started feeding the sensor with led blinks. If we could figure out what this column is used for, perhaps we can solve it. Let summarize what we assumed (and partially have know) so far:
  * Len    - Length of payload bytes, starting with column Fix (070f) and ending befire the Crc16
  * ID     - Seems to be a sender ID of some sort
  * Cnt    - A 8-bit packet counter, wrapping at 0x7F (which makes it 7-bits acutally)
  * Fix    - Some sort of flag/status column with true/false like properies stating if the sensor is detecting any blinks.
  * Fixed  - 5 bytes of static data. At present, it is hard to make something of it. (See note on 'PCnt' column description.)
  * PCnt   - A 16-bit packet counter.
  * Data   - Only modified when we prove led blinks, so it should have something to do with the measurement process.
  * NewCnt - A 32-bit led blink counter. Increases by one for every blink.
  * d3     - At present, it is hard to make something of it.
  * Crc16  - The standard Texas Instruments Crc16

### Figuring out the last byte in the XOR-key
Lets take a step back and reason a little bit. The only two columns which are changed relative to led blinks are 'Data' and 'NewCnt'. That means they are the only two columns which can affect the Watt-value printed on the receiving display. Now, the 'NewCnt' column only measures the total amounts of led blinks. However, the receiving display also shows the *current* power usage in Watts. We should look into the theory of how that works. Infact, this is commonly described as the process of converting led impulses to Watts in modern domestic electricity consumption and microgeneration meters.

XXX TODO: Insert math theory here
watts = watt-hours / hours = watt-hours / (seconds / 3600 ) = watt-hours * 3600 / seconds

### Default value assumption
In our test-unscramble operation above we received the following result:

```
    Data column: 8148
    XOR-key:     7e??
                 ----
                 FF??
```
When we see FF as the highbyte, we can start to reason. We know the lowbyte must be somewhere between 00 & FF, right? FFFF would translate into -1 in decimal form, which would be a plausable intialization value. Other values, such as FF00, translates into  -256 (or 65280) which may be valid but seems less likely. So we start with and assumption that the Data column starts with the unscrambled value of 0xFFFF.

How do we figure out the XOR-key? Well, this is what we're asking: 48 ^ ?? = FF  which in XOR-math translates into ?? = 48 ^ FF, which in turn equals B7. 

How do we verify this XOR-key? Well, lets capture some data. When we receive a packet look at the values on the receiving display and write them down. Also, we speed up the blink-rate on the Arduino-connected led to one blink per second in order to get more dynamic values. Here's a few selected lines:

```
Len ID Cnt Fix Fixed    PCnt Data NewCnt     Crc16     Data ^ Key           Watt on display
--- -- -- ---- -------- ---- ---- ---------- ----      ------------------   ---------------
 11 49 1e 070e a276170e cfbc 7aa5 47cfa3aed3 9143    # 7aa5 ^ 7EB7 = 0412   3537
 11 49 24 070e a276170e cf86 7aa3 47cfa054d3 a17f    # 7aa3 ^ 7EB7 = 0414   3531
 11 49 45 070e a276170e cfe7 7aa7 47cfa667d3 bd13    # 7aa7 ^ 7EB7 = 0410   3544
```

XXX TODO: Insert some photos here

XXX TODO: Write the finish of using these values in the formulas above to verify our assumption.





XXX TODO: continue to document the analysis here

# Ideas for the future
* Connect a logic analyzer on the sender to retrieve the CC115L settings sent from the micro-controller.
* Connect a programmer to the micro-controller and see if we can dump the flash memory.


