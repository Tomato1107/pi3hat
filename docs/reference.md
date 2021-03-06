# mjbots pi3 hat #

This is the reference documentation for the mjbots pi3 hat.  What is it?

The mjbots pi3hat is a daughterboard for a Raspberry Pi (tm), which
provides the following:

 * Power input from 8-34V, provides 5V for Raspberry Pi
 * 4x independent 5Mbps CAN-FD buses
 * 1x 125kbps CAN bus
 * A 1kHz IMU and attitude reference system
 * A spread spectrum nrf24l01 interface

# Pinouts #

## J1/J2 ##

XT30-M power input/output.  The chamfered side of the connector is
negative.

## JC1/JC2/JC3/JC4 ##

These are JST PH-3, 5Mbps CAN-FD connectors with 12V common mode
range.  Each has a physical 100 ohm terminator permanently installed.

* Pin 1: CANH
* Pin 2: CANL
* Pin 3: Ground

## JC5 ##

This is a JST PH-3 125 kbps CAN connector with 34V common mode range.
It is suitable for connecting to the mjbots power dist board or
another low speed CAN operation where high common mode range is
necessary.

* Pin 1: CANH
* Pin 2: CANL
* Pin 3: Ground

## J3 ##

This 8 pin connector allows a nrf24l01 module to be connected.

Pin 1 is closest to the J3 label.

* Pin 1: IRQ
* Pin 2: CIPO
* Pin 3: COPI
* Pin 4: SCK
* Pin 5: CS
* Pin 6: CE
* Pin 7: 3.3V
* Pin 8: GND

# Usage  with client-side C++ library#

The pi3hat can be used in two ways, the first being easy but less
flexible, the second being much harder but giving the user arbitrary
flexibility.

The easiest way to use the pi3hat is to do so using the C++ library
 provided in this repository at:
 https://github.com/mjbots/pi3hat/blob/master/mjbots/pi3hat/pi3hat.h

It consists of a single standalone C++11 header and source file, which
depends upon nothing but the standard library.  The library provides a
blocking interface that allows all the ports and features to be used
in an efficient manner.  Internally, it bypasses the linux kernel
drivers (which requires it to be run as root), and sometimes
busy-waits the CPU, in order to achieve tight timing performance.

There is a sample user of this library, `pi3hat_tool` in the same
directory which provides a simple command line program that exercises
all of its functionality.

# Board level documentation #

If you wish to use the existing linux kernel SPI drivers, or write a
custom driver, the board theory of operation and SPI register mappings
are described here.

Internally, the board has 3 separate STM32 processors, each attached
to a SPI bus on the Raspberry Pi:

* *Processor 1* The first processor provides the high speed CAN-FD
  interfaces for CAN connectors JC1 and JC2.  It is connected to the
  Raspberry Pi SPI1 bus, CS0.  Those are the following pins on the 40
  pin GPIO header:

  * Pin 35: CIPO (BCM 19)
  * Pin 38: COPI (BCM 20)
  * Pin 40: SCLK (BCM 21)
  * Pin 12: CS   (BCM 18)
  * Pin 37: IRQ  (BCM 26)

* *Processor 2*: This processor provides the CAN-FD interfaces for JC3
  and JC4.  It is connected to the Raspberry Pi SPI1 bus, CS1.  Pins:

  * Pin 35: CIPO (BCM 19)
  * Pin 38: COPI (BCM 20)
  * Pin 40: SCLK (BCM 21)
  * Pin 11: CS   (BCM 17)
  * Pin 29: IRQ  (BCM 5)

* *Processor 3*: This processor provides the low-speed CAN interface,
  the IMU, and the nrf24l01 logic.  It is connected to the Raspberry
  Pi SPI0 bus, CS0.

  * Pin 21: CIPO (BCM 9)
  * Pin 19: COPI (BCM 10)
  * Pin 23: SCLK (BCM 11)
  * Pin 24: CS   (BCM 8)
  * Pin 15: IRQ  (BCM 22)
  * Pin 16: IRQ  (BCM 23)
  * Pin 18: IRQ  (BCM 24)

## Debugging connectors ##

### JP5 ###

This connector directly breaks out 3 pins from the Raspberry Pi:

 * 1: Pin 32 (BCM 12)
 * 2: Pin 33 (BCM 13)
 * 3: Ground

### JP4 ###

Reserved debugging connector for processor 1.

### JP3 ###

Reserved debugging connector for processor 2

### JP2 ###

Reserved debugging connector for processor 3

## SPI Protocol ##

All of the processors use the same basic SPI protocol, with unique
registers for each of the various functions.

Maximum frequency: 10MHz

The following sequence is used for all SPI transactions (write or
read).

1. CS is pulled low
2. A >= 3 us hold time
3. An 8 byte address is written MSB first
4. A >= 3 us hold time
5. One or more bytes can now be read or written.  When reading, 0's
   should be clocked out.  When writing, the return value can be
   ignored.  Each "address" logically consists of sequence of bytes.
6. A >= 3 us hold time
7. CS is pulled high
8. A minimum 3us turnaround time is required before CS can be pulled
   low again.

## CAN Register Mapping ##

These addresses are present on processor 1, 2, and 3.

* *0* Protocol version: A constant byte 0x02
* *1* Interface type: A constant byte 0x01
* *2* Receive status
  * byte 0-5: Size of up to 6 received frames.  0 means no frame is
    available.  This size includes header information, so is the total
    number of bytes that must be read from address 3 to get a complete
    frame.
* *3* Received frame
  * Reading this address consumes exactly one frame from the received
    queue.
  * byte 0: (0x00 means no data)
    * bit 7: which port this was received on 0=JC1/JC3/JC5 1=JC2/JC4
    * bit 6-0: size of payload + 1 (1-65)
  * byte 1-4: CAN ID, MSB first
  * byte 5+: payload
* *4* Transmit frame with 4 byte ID
  * Writing to this address enqueues one CAN frame to be sent.
  * byte 0:
    * bit 7: which port to send on 0=JC1/JC3/JC5 1=JC2/JC4
    * bit 6-0: size of payload
  * byte 1-4: CAN ID, MSB first
  * byte 5+: payload
* *5* Transmit frame with 2 byte ID
  * Writing to this address enqueues one CAN frame to be sent.
  * byte 0:
    * bit 7: which port to send to 0=JC1/JC3/JC5 1=JC2/JC4
    * bit 6-0: size of payload
  * byte 1-2: CAN ID, MSB first
  * byte 3+: payload

## IMU Register Mapping ##

These addresses are present only on processor 3.

* *32* Protocol version: A constant byte 0x20
* *33* Raw IMU data
  * uint16 _present_
  * float _gx dps_
  * float _gy dps_
  * float _gz dps_
  * float _ax mps2_
  * float _ay mps2_
  * float _az mps2_
* *34* Attitude Data
  * uint8 _present_
  * float _w_
  * float _x_
  * float _y_
  * float _z_
  * float _x dps_
  * float _y dps_
  * float _z dps_
  * float _ax mps2_
  * float _ay mps2_
  * float _az mps2_
  * float _bias x dps_
  * float _bias y dps_
  * float _bias z dps_
  * float _uncertainty w_
  * float _uncertainty x_
  * float _uncertainty y_
  * float _uncertainty z_
  * float _uncertainty bias x dps_
  * float _uncertainty bias y dps_
  * float _uncertainty bias z dps_
  * uint32 _error count_
  * uint32 _last error_
* *35* Read configuration
  * float _yaw deg_
  * float _pitch deg_
  * float _roll deg_
  * uint32t _rate Hz_
* *36* Write configuration
  * The same structure as for address 35.

# RF Register Mapping #

If a nrf24l01 module is attached, the pi3hat implements a spread
spectrum receiver.  This receiver must be configured with the same ID
as the transmitter for communication to take place.

Each "slot" is transmitted independently and repeatedly according to
the given bitmask schedule.  The individual RF frames occur at 50Hz,
and each bit in the bitmask controls whether the data should be sent
in that particular frame.  So 0xaaaaaaaa would result in the data
being sent every other frame.

* *48* Protocol version: A constant byte 0x11
* *49* Read ID
  * byte 0-3: Common RF ID
* *50* Write ID
* *51* Set data for transmission
  * byte 0: Slot number (0-15)
  * byte 1-4: Transmission schedule (little endian)
  * byte 5+: Data
* *52* Read status
  * byte 0-3: uint32 rx slot counter, 2 bits per slot (little endian)
    * each time new data is received from the transmitter, the 2 bit
      counter for that slot is incremented
  * byte 4-7: uint32 age since last lock from tx (little endian)
* *64-79* Read data from remote transmitter
  * byte 0-3: uint32 age in ms (little endian)
  * byte 4: size (0-16)
  * byte 5-20: data


# Optional Raspberry Pi SD card image #

The optional Raspberry Pi with SD card comes with RealtimePi flashed
on it from https://github.com/guysoft/RealtimePi/releases
Additionally, it is configured to present as a 5GHz wifi access point.
The SSID is MjMech-xxxxxx, and the password is "WalkingRobots".  The
user name and password are the default for the rpi, pi/raspberry

The IP address is 192.168.16.47, you can ssh to it from linux:

```
ssh pi@192.168.16.47
```

In the home directory is a copy of the compiled `pi3hat_tool`, you can
run it:

```
pi@realtimepi:~ $  sudo bash

root@realtimepi:/home/pi# ./pi3hat_tool --read-att
```

Which will then display the current attitude information.
