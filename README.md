# linux-intel-undervolt
Guide to linux undervolting for Haswell and never Intel CPUs

## Disclaimers

I am not responsible for any damage you do by following the instructions found here. There is no official documentation about the MSR interface used, everything was reverse engineered by watching [Throttlestop](http://forum.notebookreview.com/threads/the-throttlestop-guide.531329) system calls and trial and error.

This is applicable to all Intel CPUs with an integrated voltage controller (FIVR). In general this includes all mobile Haswell and newer CPUs. I only tested this on one machine, a laptop with a i7-6700HQ.


## HOW TO
__TODO__ : write an undervolting guide (for now just a guide for people who already know about undervolting)

### Finding voltage offsets

#### Using windows
If you are already undervolting using Throttlestop in Windows, you can easily copy the voltage HEX values from *ThrottleStop.ini*. They look like this:
```
...
FIVRVoltage00=0xECC00000
UnlockVoltage00=1
FIVRVoltage01=0xECC00000
UnlockVoltage01=1
FIVRVoltage02=0xECC00000
UnlockVoltage02=1
FIVRVoltage03=0xECC00000
UnlockVoltage03=1
FIVRVoltage10=0xF0000000
UnlockVoltage10=1
FIVRVoltage11=0xF0000000
UnlockVoltage11=1
FIVRVoltage12=0xF0000000
UnlockVoltage12=1
FIVRVoltage13=0xF0000000
...
```

The first number after `FIVRVoltage` is the index of the voltage plane and they correspond to the order in which they are displayed in Throttlestop. On my machine the order is as follows:

0 - CPU Core

1 - Intel GPU

2 - CPU Cache

3 - System Agent

4 - Analog I/O

5 - Digital I/O

On my machine index 5 is greyed out in Throttlestop, so I have not played with it.
For more information about these planes, check out the Throttlestop guide thread (linked in the beginning). There are some limitations that are model specific, like CPU Core and Cache sharing the voltage plane, meaning that only the higher voltage of the two settings is applied, again the Throttlestop thread is right now the best reference for this.

The second number after `FIVRVoltage` is another index which I don't really know what it means. Right now my *.ini* has all three set to the same value, but changing the slider only the voltage at index *_2* changes. My guess is this is some feature of Throttlestop, that stores old values that get restored in case of a crash.

If you are unsure, you can always change the slider in Throttlestop, save the configuration, look at the *.ini* and figure out the correct values.

You should now have a hex value for each of the voltage planes. You can skip the next section (or go along and check the numbers).

#### Calculating the offset manually
If you skipped the windows section, please read it anyway as there is information there I don't want to repeat here.

At this point I would like to remind you that this is reverse engineering / guess work. Since I only have one machine that is new enough for this, I have no way to check if the same principles work on other CPUs. It is possible that the actual offsets and plane indexes are different across CPUs (especially across generations of cpus). 

The offset I got from Throttlestop for the CPU Core for my  -150.4 mV offset is `0xECC00000`. My GPU has an offset of -125.0 mV and a HEX offset of `0xF0000000`.
The HEX value is actually a 12 bit number (first 3 characters, each hex character is 4 bits). `0xECC` is -308 and `0xF00` is -256.
The offset has about 0.5 mV increments. 

To calculate the actual offset you multiply the offset in mV with -2.048, round to the nearest integer, then take the 12 least significant bits (or last three charcters in hex). 

Example 50mV undervolt:

1. First multiply 50 by -2.048 and get `-102.4`
2. Round to `-102`
3. Convert to HEX and get ‭`‭FFFFFFFFFFFFFF9A‬‬`
4. Take the last 12 bits (last three characters) and get `0xF9A`
5. Pad with zeros to get the right format `0xF9A00000`

Since we rounded in step 2, the actual offset is 49,8 mV. 

### Setting the voltages
You need to be able to write to MSR registers. The easiest way is to use `wrmsr` from `msr-tools` Check with your distro to see how to get it. These settings reset on reboot or S3 sleep, so if you set the voltage too low and crash, it resets. To keep the undervolt after sleep you must create a script that runs after resume.

Example usage (-50mV undervolt of the CPU Core plane):

`wrmsr 0x150 0x80000011F9A00000`

Explanation:

`0x150` is the MSR register, `0x80000011F9A00000` is the value we are setting it to.

The value can be deconstructed to 5 parts:

| constant | plane index | constant | write/read | offset     |
|----------|-------------|----------|------------|------------|
| `80000`  | `0`         | `1`      | `1`        | `F9A00000` |

I do not know what the constants mean, you only need to change the plane index and offset.

More examples:

Undervolt GPU by 125mV and everything else by 150mV:
```
wrmsr 0x150 0x80000011ecc00000          
wrmsr 0x150 0x80000111f0000000          
wrmsr 0x150 0x80000211ecc00000          
wrmsr 0x150 0x80000311ecc00000          
wrmsr 0x150 0x80000411ecc00000 
```

### Reading from the OC mailbox
To read (and check if you set the voltages correctly) you must first write to `0x150` and then read it again.
Set the plane index to the plane you want to read, write/read to 0 and offset to 0.

Example (reading the first three planes on my machine):
```
# wrmsr 0x150 0x8000001000000000
# rdmsr 0x150
ecc00000
# wrmsr 0x150 0x8000011000000000
# rdmsr 0x150
f0000000
# wrmsr 0x150 0x8000021000000000
# rdmsr 0x150
ecc00000
```

