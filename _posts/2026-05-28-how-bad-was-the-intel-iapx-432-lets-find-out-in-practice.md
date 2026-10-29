---
title: "How bad was the Intel iAPX 432 - let's find out in practice"
date: 2026-05-28 08:10:03
slug: how-bad-was-the-intel-iapx-432-lets-find-out-in-practice
layout: post
---

Back in the 80s there was such processor - the Intel iAPX 432. It was developed as a successor to the 8080 and initially even carried the codename 8800. Intel packed an enormous amount into it - a completely new architecture, nothing like its predecessors, and even some OS concepts baked right into silicon - support for object-oriented programming, garbage collection, a process scheduler, asynchronous communications, multiple levels of fault tolerance, and much more.
But due to its complexity, the architecture failed. There are several post-mortems describing the problems and the reasons for the failure, but the short version is that the technology of the time severely limited how complex a physical chip could be. Intel made a number of compromises that badly hurt performance. The CPU had to be split across two chips, since fitting all the logic into one wasn't possible. And even that wasn't enough to include all the features they wanted - even something as useful as a register file.
I am not kidding - the iAPX 432 had exactly one indirectly accessible general-purpose register (a 16-bit top-of-stack), and every other variable access went through memory. On top of that, the system was positioned by Intel as capability-based, which meant data access was far more complex than simply reading or writing a value at a specific address. I'll come back to this, but the decision made the architecture's problems considerably worse.
There were a few other questionable design choices, some of which were revised in later revisions. The changes were quite sweeping and partially fixed the iAPX 432's issues, but by then the train had already left the station and the market buried Intel's innovative brainchild.
Fortunately, the company had a Plan B. While the brightest minds were focused on the breakthrough system, another team was working on a stopgap - the 8086, which was supposed to cover people's immediate needs. The "temporary" x86 architecture ended up dominating for several decades, while the iAPX 432 lives on only in the memory of computer enthusiasts. Yeah, that's how it goes.
I enjoy poking around with old processors, so there was no way I was going to pass up the chance to run something on this peculiar thing. What made it even more interesting was that, as far as I know, nobody had touched a working 432 system in the past couple of decades.
## Hardware

The processor (a.k.a. GDP, general data processor) came to me as part of an iSBC 432/100 board. It's a single board computer with a Multibus interface, intended for use with the Intel Intellec MDS. But of course, I wanted much more control over the processor's signals and a friendlier interface to work with. So I did what I usually do - designed a simple board with an FPGA and SRAM on board, which would cover everything the processor needs.
<figure class="wp-block-image"><img alt="my design of SBC" class="aligncenter size-full wp-image-179" height="1857" src="../../../images/2026/05/01.png" width="1943"/></figure>

Besides power, I needed to take care of signal level translation (a lot of chips back then ran on 5V). And that's pretty much it - the board is quite simple and routed on just 2 layers.
A couple of noteworthy details. I put a TPS63002 on there to convert the floating voltage from the USB connector into a stable 5V, but it didn't last very long. Either it's not suited for this kind of use, or there's a mistake somewhere on my end.
The second feature of the board is sharing a single connector for both flashing the SPI flash with the FPGA bitstream and for UART communication with the host. Normally I'd put a UART-USB bridge on there, so the USB connection handles both power and the communication channel. But in this case I played it safe - my PC can't deliver decent current through USB 2.0, and according to the spec the iAPX 432 can be quite power-hungry, so the USB cable is connected to a power supply instead. At the same time I didn't want a bundle of wires hanging off it, so I combined the two functions into one connector.
To bring the ft232h back into UART mode after it's been used to flash the chip, all you need to do is restart the kernel module:
```
sudo modprobe -r ftdi_sio
sudo modprobe ftdi_sio
```

## Gateware

For the FPGA I went with a Lattice iCE40HX, mainly because of the open-source toolchain for synthesizing bitstreams. I picked the specific chip based on having a solderable package and enough pins.
For memory I chose a synchronous parallel SRAM rated at 250MHz (access time was specified at 2.6ns). I wasn't able to hit the maximum clock speed here (even though the same combination worked at 250MHz in another project), but 125MHz turned out to be more than enough to respond to the processor within a single cycle (without introducing extra wait states), so I didn't bother hunting for the right timings to push it higher.
<figure class="wp-block-image"><img alt="debugging bus transactions" class="aligncenter size-full wp-image-180" height="1920" src="../../../images/2026/05/02-scaled.jpg" width="2560"/></figure>

In my design the FPGA acted as the memory controller (bus slave), clock generator for the iAPX 432 (it needs 3 clock signals), and communicated with the control software running on the PC. For debugging purposes I wanted a log of all memory accesses from the GDP, to study the logic of its operation.
On the Verilog side, the only real problem was trying to get the SRAM running at 250MHz. Yosys (the synthesis tool) kept inserting SB_DFFE elements (D flip-flops with a Clock Enable input) that completely blew the timing budget (and at 250MHz that budget isn't very generous). In the end I put together a clean module that synthesized successfully and even worked - at lower frequencies - but unfortunately not at 250MHz in the current topology.
<figure class="wp-block-image"><img alt="bus read timings" class="aligncenter size-full wp-image-181" height="1407" src="../../../images/2026/05/03.png" width="2130"/></figure>

The bus has a pretty straightforward interface. The only thing worth mentioning is that at almost any moment a signal can arrive indicating that someone has initiated an IPC message (inter-processor communication).
The request packet from the bus master (GDP) is 32 bits - 24 bits of address and 8 bits describing what the processor wants.
<figure class="wp-block-image"><img alt="specification field for bus operation" class="aligncenter size-full wp-image-182" height="1152" src="../../../images/2026/05/04.png" width="2130"/></figure>

The most obvious field is the operation type - read or write. The length is self-explanatory too. The modifiers are mostly informational and don't affect the behavior of my memory controller. The Access bit is a bit more interesting - the 432 operates with two address spaces: regular device memory and external registers for inter-processor communication (that's where things like the device ID live). Since we have no Interface Processor (another processor from 432 family to manage communications between 432 system and more classic environment) in our system, we'll mostly be using just the regular memory space.
The last flag (RMW) is also quite neat. It provides bus-level atomicity - short for Read-Modify-Write. A read with the RMW flag locks the memory at that address - until a write command arrives (or a timeout expires), all other reads to that address will hang without a response. In my simplified system with a single GDP and a passive slave memory controller there aren't going to be any competing memory requests anyway, so we can skip implementing the logic for this feature.
## Getting the processor ready to start

I haven't mentioned this before, but you can't build a functional system with just 432-family processors. Due to its object-oriented nature, the iAPX 432 expects someone to have already set up a bunch of structures in memory. There always has to be some attached processor (like 8080 or even 8086) that handles memory preparation and send initialization signal to the main actors in the distributed 432 system. Before it can execute any user code, the 432 processor goes through a whole lot of activity reading system objects - the segment object tables, processor info (registers in that inter-processor space and structures in regular memory), the current process, code and data segments, and so on.
So we need to build a memory snapshot and load it into the SRAM before starting the processor.
After the INIT/ signal is asserted, the GDP starts hammering out read requests for inter-processor register 0x02. This register holds the IPC status - whether there's an external message waiting to be handled. If an IPC arrives, the processor kicks off its full wake-up sequence. In theory, memory can be shared between multiple devices, so you can't rely on hardcoded absolute addresses for reading the various system structures. Only one address is baked into the microcode - the address of the global object directory. The processor uses its device ID (obtained by reading register 0x00) as an index into this global table to read a Processor object.
From there it starts fetching the addresses of objects specific to that particular processor. Before executing the first user instruction, the GDP performs around 150 memory operations. And we need to be able to respond to all of those correctly.
<details><summary>Access log example</summary>
<pre>[+] Connected to SBC
[+] SBC is online
[~] Building image...
[+] ROM image has been written to SBC, size = 1110 bytes
[+] GDP has been started
[~] Read access log after 2s of execution.
[+] Access log (skipped 0 entries):
  [000] GDP initialization
  [001] spec:  addr: 0x0002
  [002] spec:  addr: 0x0000
  [003] spec:  addr: objectTableDirectory/objectTableProcessor (0x0018)
  [004] spec:  addr: objectTableDirectory/objectTableProcessor (0x0018)
  [005] spec:  addr: objectTableDirectory/objectTableProcessor (0x0018) &lt;58d7&gt;
  [006] spec:  addr: objectTableDirectory+0x18 (0x0020)
  [007] spec:  addr: objectTableDirectory+0x14 (0x001c)
  [008] spec:  addr: objectTableProcessor/processorAccess (0x0068)
  [009] spec:  addr: objectTableProcessor/processorAccess (0x0068)
  [010] spec:  addr: objectTableProcessor/processorAccess (0x0068) &lt;78df&gt;
  [011] spec:  addr: processorAccess+0x10 (0x0088)
  [012] spec:  addr: objectTableDirectory/objectTableDirectory (0x0038)
  [013] spec:  addr: objectTableDirectory/objectTableDirectory (0x0038)
  [014] spec:  addr: objectTableDirectory/objectTableDirectory (0x0038) &lt;08d7&gt;
  [015] spec:  addr: objectTableDirectory+0x38 (0x0040)
  [016] spec:  addr: objectTableDirectory+0x34 (0x003c)
  [017] spec:  addr: objectTableDirectory/objectTableDirectory (0x0038)
  [018] spec:  addr: processorAccess+0x00 (0x0078)
  [019] spec:  addr: objectTableDirectory/objectTableMain (0x0048)
  [020] spec:  addr: objectTableDirectory/objectTableMain (0x0048)
  [021] spec:  addr: objectTableDirectory/objectTableMain (0x0048) 
  [022] spec:  addr: objectTableDirectory+0x48 (0x0050)
  [023] spec:  addr: objectTableDirectory+0x44 (0x004c)
  [024] spec:  addr: objectTableMain/processorData (0x00e8)
  [025] spec:  addr: objectTableMain/processorData (0x00e8)
  [026] spec:  addr: objectTableMain/processorData (0x00e8) 
  [027] spec:  addr: processorData+0x00 (0x01e8)
  [028] spec:  addr: processorData+0x00 (0x01e8) &lt;0005&gt;
  [029] spec:  addr: processorAccess+0x00 (0x0078)
  [030] spec:  addr: objectTableMain/processorData (0x00e8)
  [031] spec:  addr: processorData+0x02 (0x01ea) &lt;0102&gt;
  [032] spec:  addr: processorAccess+0x08 (0x0080)
  [033] spec:  addr: objectTableMain/processorLocalComms (0x00f8)
  [034] spec:  addr: objectTableMain/processorLocalComms (0x00f8)
  [035] spec:  addr: objectTableMain/processorLocalComms (0x00f8) &lt;78d7&gt;
  [036] spec:  addr: processorLocalComms+0x00 (0x0278)
  [037] spec:  addr: processorLocalComms+0x00 (0x0278) &lt;0005&gt;
  [038] spec:  addr: processorLocalComms+0x02 (0x027a)
  [039] spec:  addr: processorLocalComms+0x04 (0x027c) &lt;0000&gt;
  [040] spec:  addr: processorLocalComms+0x00 (0x0278)
  [041] spec:  addr: processorLocalComms+0x00 (0x0278) &lt;0000&gt;
  [042] spec:  addr: processorAccess+0x00 (0x0078)
  [043] spec:  addr: objectTableMain/processorData (0x00e8)
  [044] spec:  addr: processorData+0x02 (0x01ea) &lt;0102&gt;
  [045] spec:  addr: processorAccess+0x18 (0x0090)
  [046] spec:  addr: objectTableMain/delayCarrierAccess (0x0138)
  [047] spec:  addr: objectTableMain/delayCarrierAccess (0x0138)
  [048] spec:  addr: objectTableMain/delayCarrierAccess (0x0138) 
  [049] spec:  addr: objectTableMain+0x49 (0x0121) &lt;0001&gt;
  [050] spec:  addr: processorAccess+0x24 (0x009c) &lt;004f 004f&gt;
  [051] spec:  addr: objectTableMain/delayPortAccess (0x0118)
  [052] spec:  addr: objectTableMain/delayPortAccess (0x0118)
  [053] spec:  addr: objectTableMain/delayPortAccess (0x0118) &lt;9adf&gt;
  [054] spec:  addr: delayPortAccess+0x00 (0x029a)
  [055] spec:  addr: objectTableMain/delayPortData (0x0108)
  [056] spec:  addr: objectTableMain/delayPortData (0x0108)
  [057] spec:  addr: objectTableMain/delayPortData (0x0108) &lt;82d7&gt;
  [058] spec:  addr: delayPortData+0x00 (0x0282)
  [059] spec:  addr: delayPortData+0x00 (0x0282) &lt;0005&gt;
  [060] spec:  addr: delayPortData+0x06 (0x0288)
  [061] spec:  addr: delayPortData+0x00 (0x0282)
  [062] spec:  addr: delayPortData+0x00 (0x0282)
  [063] spec:  addr: delayPortData+0x00 (0x0282) &lt;0000&gt;
  [064] spec:  addr: processorAccess+0x14 (0x008c)
  [065] spec:  addr: objectTableMain+0x89 (0x0161) &lt;0001&gt;
  [066] spec:  addr: processorAccess+0x28 (0x00a0) &lt;008f 004f&gt;
  [067] spec:  addr: objectTableMain/normalCarrierAccess (0x0158)
  [068] spec:  addr: objectTableMain/normalCarrierAccess (0x0158)
  [069] spec:  addr: objectTableMain/normalCarrierAccess (0x0158) 
  [070] spec:  addr: normalCarrierAccess+0x00 (0x02f2)
  [071] spec:  addr: objectTableMain/normalCarrierData (0x0148)
  [072] spec:  addr: objectTableMain/normalCarrierData (0x0148)
  [073] spec:  addr: objectTableMain/normalCarrierData (0x0148) 
  [074] spec:  addr: normalCarrierData+0x02 (0x02e4)
  [075] spec:  addr: normalCarrierData+0x02 (0x02e4) &lt;0008&gt;
  [076] spec:  addr: normalCarrierAccess+0x1c (0x030e)
  [077] spec:  addr: objectTableMain+0xa9 (0x0181) &lt;0001&gt;
  [078] spec:  addr: processorAccess+0x04 (0x007c) &lt;00af 004f&gt;
  [079] spec:  addr: processorAccess+0x00 (0x0078)
  [080] spec:  addr: objectTableMain/processorData (0x00e8)
  [081] spec:  addr: processorData+0x02 (0x01ea) &lt;0103&gt;
  [082] spec:  addr: objectTableMain+0xa9 (0x0181) &lt;0001&gt;
  [083] spec:  addr: processorAccess+0x28 (0x00a0) &lt;00af 004f&gt;
  [084] spec:  addr: objectTableMain/processCarrierAccess (0x0178)
  [085] spec:  addr: objectTableMain/processCarrierAccess (0x0178)
  [086] spec:  addr: objectTableMain/processCarrierAccess (0x0178) &lt;26df&gt;
  [087] spec:  addr: processCarrierAccess+0x00 (0x0326)
  [088] spec:  addr: objectTableMain/processCarrierData (0x0168)
  [089] spec:  addr: objectTableMain/processCarrierData (0x0168)
  [090] spec:  addr: objectTableMain/processCarrierData (0x0168) &lt;16d7&gt;
  [091] spec:  addr: processCarrierData+0x04 (0x031a)
  [092] spec:  addr: processCarrierData+0x00 (0x0316)
  [093] spec:  addr: processCarrierData+0x00 (0x0316) &lt;0005&gt;
  [094] spec:  addr: processCarrierAccess+0x20 (0x0346)
  [095] spec:  addr: objectTableMain/processAccess (0x0198)
  [096] spec:  addr: objectTableMain/processAccess (0x0198)
  [097] spec:  addr: objectTableMain/processAccess (0x0198) 
  [098] spec:  addr: processAccess+0x00 (0x03da)
  [099] spec:  addr: objectTableMain/processData (0x0188)
  [100] spec:  addr: objectTableMain/processData (0x0188)
  [101] spec:  addr: objectTableMain/processData (0x0188) &lt;4ad7&gt;
  [102] spec:  addr: processData+0x00 (0x034a)
  [103] spec:  addr: processData+0x00 (0x034a) &lt;0005&gt;
  [104] spec:  addr: processData+0x20 (0x036a)
  [105] spec:  addr: processAccess+0x14 (0x03ee)
  [106] spec:  addr: processCarrierAccess+0x0c (0x0332) &lt;0000 0000&gt;
  [107] spec:  addr: processAccess+0x04 (0x03de)
  [108] spec:  addr: objectTableMain/processContext0Access (0x01a8)
  [109] spec:  addr: objectTableMain/processContext0Access (0x01a8)
  [110] spec:  addr: objectTableMain/processContext0Access (0x01a8) &lt;0adf&gt;
  [111] spec:  addr: processContext0Access+0x00 (0x040a)
  [112] spec:  addr: objectTableMain/processContext0Data (0x01b8)
  [113] spec:  addr: objectTableMain/processContext0Data (0x01b8)
  [114] spec:  addr: objectTableMain/processContext0Data (0x01b8) &lt;32d7&gt;
  [115] spec:  addr: processContext0Access+0x14 (0x041e)
  [116] spec:  addr: processData+0x32 (0x037c) 
  [117] spec:  addr: processContext0Access+0x14 (0x041e) &lt;0000 0000&gt;
  [118] spec:  addr: processContext0Access+0x18 (0x0422)
  [119] spec:  addr: processData+0x34 (0x037e) 
  [120] spec:  addr: processContext0Access+0x18 (0x0422) &lt;0000 0000&gt;
  [121] spec:  addr: processContext0Access+0x1c (0x0426)
  [122] spec:  addr: processData+0x36 (0x0380) 
  [123] spec:  addr: processContext0Access+0x1c (0x0426) &lt;0000 0000&gt;
  [124] spec:  addr: processContext0Access+0x24 (0x042e)
  [125] spec:  addr: processContext0Data+0x00 (0x0432)
  [126] spec:  addr: processContext0Access+0x20 (0x042a)
  [127] spec:  addr: objectTableMain/processContext0Domain (0x01c8)
  [128] spec:  addr: objectTableMain/processContext0Domain (0x01c8)
  [129] spec:  addr: objectTableMain/processContext0Domain (0x01c8) &lt;409f&gt;
  [130] spec:  addr: processContext0Domain+0x00 (0x0440)
  [131] spec:  addr: objectTableMain/processContext0Instruction0 (0x01d8)
  [132] spec:  addr: objectTableMain/processContext0Instruction0 (0x01d8)
  [133] spec:  addr: objectTableMain/processContext0Instruction0 (0x01d8) &lt;4497&gt;
  [134] spec:  addr: processorAccess+0x00 (0x0078)
  [135] spec:  addr: objectTableMain/processorData (0x00e8)
  [136] spec:  addr: processorData+0x02 (0x01ea) &lt;0104&gt;
  [137] spec:  addr: processContext0Instruction0+0x0e (0x0452)
  [138] spec:  addr: processContext0Access+0x08 (0x0412)
  [139] spec:  addr: processAccess+0x0c (0x03e6)
  [140] spec:  addr: objectTableDirectory:Header (0x0008)
  [141] spec:  addr: processData+0x7c (0x03c6) &lt;0000 0000 0000 0000 7fff&gt;
  [142] spec:  addr: processData+0x86 (0x03d0) &lt;0004 010f 010f 0000 7fff&gt;
  [143] spec:  addr: processData+0x74 (0x03be) &lt;00cd 7417 000c 0000&gt;
  [144] spec:  addr: processData+0x68 (0x03b2) &lt;0000 0076 0070 0000&gt;
  [145] spec:  addr: processData+0x70 (0x03ba) &lt;0000 0001&gt;
  [146] spec:  addr: processAccess+0x10 (0x03ea)
  [147] spec:  addr: processorAccess+0x00 (0x0078)
  [148] spec:  addr: objectTableMain/processorData (0x00e8)
  [149] spec:  addr: processorData+0x54 (0x023c) &lt;0000 0001 0001 0000 7fff&gt;
  [150] spec:  addr: processorData+0x5e (0x0246) &lt;0000 0000 0000 0000 7fff&gt;
  [151] spec:  addr: processorData+0x4c (0x0234) &lt;00cd 7a00 000c 0000&gt;
  [152] spec:  addr: processorData+0x40 (0x0228) &lt;0000 0070 0070 0000&gt;
  [153] spec:  addr: processorData+0x48 (0x0230) &lt;0000 0001&gt;
  [154] spec:  addr: processorAccess+0x14 (0x008c)
  [155] spec:  addr: objectTableMain/normalCarrierAccess (0x0158)
  [156] spec:  addr: normalCarrierAccess+0x00 (0x02f2)
  [157] spec:  addr: objectTableMain/normalCarrierData (0x0148)
  [158] spec:  addr: normalCarrierData+0x02 (0x02e4)
  [159] spec:  addr: normalCarrierData+0x02 (0x02e4) &lt;000c&gt;
  [160] spec:  addr: processorAccess+0x00 (0x0078)
  [161] spec:  addr: objectTableMain/processorData (0x00e8)
  [162] spec:  addr: processorData+0x02 (0x01ea) &lt;0105&gt;
  [163] spec:  addr: processContext0Data+0x00 (0x0432) &lt;0000 0000 0000 0070&gt;
  [164] spec:  addr: processData+0x22 (0x036c)
  [165] spec:  addr: processData+0x24 (0x036e) &lt;0043 0000 0000&gt;
  [166] spec:  addr: processData+0x00 (0x034a)
  [167] spec:  addr: processData+0x00 (0x034a) &lt;0000&gt;
  [168] spec:  addr: processorAccess+0x00 (0x0078)
  [169] spec:  addr: objectTableMain/processorData (0x00e8)
  [170] spec:  addr: processorData+0x02 (0x01ea) &lt;0105&gt;
  [171] spec:  addr: processorAccess+0x04 (0x007c)
  [172] spec:  addr: objectTableMain/processCarrierAccess (0x0178)
  [173] spec:  addr: processCarrierAccess+0x00 (0x0326)
  [174] spec:  addr: objectTableMain/processCarrierData (0x0168)
  [175] spec:  addr: processCarrierData+0x00 (0x0316)
  [176] spec:  addr: processCarrierData+0x00 (0x0316) &lt;0001 0000 0000 0000 0000&gt;
  [177] spec:  addr: processorAccess+0x00 (0x0078)
  [178] spec:  addr: objectTableMain/processorData (0x00e8)
  [179] spec:  addr: processorData+0x02 (0x01ea) &lt;0135&gt;
  [180] spec:  addr: processorAccess+0x4c (0x00c4)
  [181] spec:  addr: processorAccess+0x14 (0x008c) &lt;0000 0000&gt;
  [182] spec:  addr: processorAccess+0x00 (0x0078)
  [183] spec:  addr: objectTableMain/processorData (0x00e8)
  [184] spec:  addr: processorData+0x02 (0x01ea) &lt;0132&gt;
  [185] spec:  addr: processorAccess+0x18 (0x0090)
  [186] spec:  addr: objectTableMain/delayCarrierAccess (0x0138)
  [187] spec:  addr: objectTableMain+0x49 (0x0121) &lt;0001&gt;
  [188] spec:  addr: processorAccess+0x24 (0x009c) &lt;004f 004f&gt;
  [189] spec:  addr: objectTableMain/delayPortAccess (0x0118)
  [190] spec:  addr: delayPortAccess+0x00 (0x029a)
  [191] spec:  addr: objectTableMain/delayPortData (0x0108)
  [192] spec:  addr: delayPortData+0x00 (0x0282)
  [193] spec:  addr: delayPortData+0x00 (0x0282) &lt;0005&gt;
  [194] spec:  addr: delayPortData+0x06 (0x0288)
  [195] spec:  addr: delayPortData+0x00 (0x0282)
  [196] spec:  addr: delayPortData+0x00 (0x0282)
  [197] spec:  addr: delayPortData+0x00 (0x0282) &lt;0000&gt;
  [198] spec:  addr: processorAccess+0x14 (0x008c)
  [199] spec:  addr: processorAccess+0x28 (0x00a0) &lt;0000 0000&gt;
  [200] Fatal signal is raised by GDP
</pre>
</details>

Here's a rough visualization of the object hierarchy that makes up the minimal set needed to run our code.
<figure class="wp-block-image"><img alt="object graph, required to run any user code" class="aligncenter size-full wp-image-183" height="1485" src="../../../images/2026/05/05-scaled.png" width="2560"/></figure>

Here is a code that builds necessary objects:
```javascript
  const processorObjectTable = new ObjectTable('objectTableProcessor');
  // empty, would not be used
  const tempDirObjectTable = new ObjectTable('objectTableTemp');
  const mainObjectTable = new ObjectTable('objectTableMain');
  const directoryObjectTable = new ObjectTable('objectTableDirectory');
  const objectDirectory = new ObjectTableDirectory(directoryObjectTable);
  objectDirectory.addObjectTable(processorObjectTable);
  objectDirectory.addObjectTable(tempDirObjectTable);
  objectDirectory.addObjectTable(directoryObjectTable);
  objectDirectory.addObjectTable(mainObjectTable);
  // processors object table contains only processor access segments
  processorObjectTable.addObject(new ProcessorAccessSegment('processorAccess', { directoryObjectTable }));
  // interconnect segment for UART output
  mainObjectTable.addInterconnectSegment('uartInterconnect', 0x1000, 0x10);
  // here is all objects, except processor access segments
  mainObjectTable.addObject(new ProcessorDataSegment('processorData'));
  mainObjectTable.addObject(new LocalCommunicationSegment('processorLocalComms'));
  // delay port
  mainObjectTable.addObject(new PortDataSegment('delayPortData', { messageQueueSize: 1, portType: PORT_TYPE.DELAY }));
  mainObjectTable.addObject(new PortAccessSegment('delayPortAccess', { directoryObjectTable, messageQueueSize: 1 }));
  mainObjectTable.addObject(new CarrierDataSegment('delayCarrierData', { carrierType: CARRIER_TYPE.PROCESSOR }));
  mainObjectTable.addObject(new CarrierAccessSegment('delayCarrierAccess', { directoryObjectTable }));
  // actual process objects
  mainObjectTable.addObject(new CarrierDataSegment('normalCarrierData', { carrierType: CARRIER_TYPE.PROCESSOR, hasMessage: true }));
  mainObjectTable.addObject(new CarrierAccessSegment('normalCarrierAccess', { directoryObjectTable, messageRef: 'processCarrierAccess' }));
  mainObjectTable.addObject(new CarrierDataSegment('processCarrierData', { carrierType: CARRIER_TYPE.PROCESSOR, hasMessage: true }));
  mainObjectTable.addObject(new CarrierAccessSegment('processCarrierAccess', { directoryObjectTable, carriedObjectRef: 'processAccess' }));
  mainObjectTable.addObject(new ProcessDataSegment('processData'));
  mainObjectTable.addObject(new ProcessAccessSegment('processAccess', { directoryObjectTable }));
  const contextParams = { directoryObjectTable, objectsRefs: ['uartInterconnect', 'processContext0Vars'] };
  mainObjectTable.addObject(new ContextAccessSegment('processContext0Access', contextParams));
  mainObjectTable.addObject(new ContextDataSegment('processContext0Data', { sp: 0 }));
  const stackParams = { size: stack.size, data: stack.data, type: SEGMENT_TYPE.OPERAND_STACK_DATA };
  mainObjectTable.addObject(new GenericDataSegment('processContext0Stack', stackParams));
  mainObjectTable.addObject(new GenericDataSegment('processContext0Vars', { data: varsData, type: SEGMENT_TYPE.GENERIC_DATA }));
  const domainParams = { directoryObjectTable, instructionsRefs: ['processContext0Instruction0'] };
  mainObjectTable.addObject(new DomainSegment('processContext0Domain', domainParams));
  const instructionsParams = { directoryObjectTable, instructions: bytecode, contextIdx: 0 };
  mainObjectTable.addObject(new InstructionSegment('processContext0Instruction0', instructionsParams));
```

Rather than walk through every single memory access, I'll take a high-level approach and just cover the interesting bits and quirks of the iAPX 432.
First, let's talk about how object pointer translation to physical addresses works. Every pointer consists of two parts - an object table index and an index of the object within that table. So to compute a physical address, the processor first has to find the table's address (by reading the central directory, which holds the addresses of the individual tables), and only then reads the descriptor from that table, which contains the physical address.
<figure class="wp-block-image"><img alt="two-level pointers" class="aligncenter size-full wp-image-185" height="1027" src="../../../images/2026/05/06.png" width="1474"/></figure>

Why so complex? Because several features needed to be implemented:
1. Every pointer, in addition to the address, carries the access rights that pointer holds - whether it can be used for reading, writing, etc. The set of available rights depends on the object type: if a pointer to Processor object can permit high-level operations like sending a message to the processor or querying its cycle counter (that info doesn't live in regular memory). A Process object has its own set of rights. So the object type has to be checked regularly (it's a field in the descriptor).
2. Garbage collection. It was still the 80s, so many GC concepts of the time were fairly naive. And cramming complex algorithms into the severely constrained microcode space wasn't easy either. So memory management isn't all that complicated here. Nesting level could be assigned to an object. When a "function" (of course in 432 terms this is called something entirely different, but I'm simplifying) finishes executing and returns to the caller, the processor frees all objects matching that function's nesting level. The descriptor also contains a flag indicating whether the object is currently in use by anyone.

On top of that, the descriptor stores various housekeeping metadata (I'm not sure whether the GDP itself uses it or whether it's meant for external systems): flags for whether the object was accessed, whether it was written to, whether it contains any data or is all zeros, and so on. If the processor doesn't actually need any of this, than it is an additional source of performance drag - those fields get synchronized as execution proceeds, which means extra memory accesses.
By the way, if you look at the code, you'll notice that some objects have two segments - Data and Access. For example, `ProcessorDataSegment` / `ProcessorAccessSegment` . What's that about? It's one of Intel's more controversial decisions (later reversed in the third revision of the iAPX 432). An object is split into 2 parts - the access segment holds only pointers, while the data segment holds everything else. This means that when working with a system object, the processor typically ends up touching 2 physical segments. Rarely does any operation need only pointers or only scalar fields. This doubles the number of memory accesses, since reading data from a segment requires a two-step journey through the object table address lookup and then the table entries themselves. And on top of that there may be housekeeping data to write back as well. No wonder Intel ended up with such a sluggish processor...
Not all, but many objects can be locked - either by hardware or programmatically (via the `LOCK OBJECT` instruction). The implementation is fairly straightforward - the data segment of the object has a field containing lock metadata: the lock type and the identifier of whoever is holding it. Also a fascinating concept in a 50-year-old processor, but what would you say about hardware-level message queues with priority support, TTL, non-blocking operations, and a bunch of other bells and whistles?
This mechanism is used not only for passing messages between different processes running on the GDP, but also by the scheduler itself. There's a set of processors and a set of processes. The scheduler is responsible for finding a process to run on a given processor. Message queues are used for this. Processors act as consumers, processes are the messages, and various sources - both internal and external - can publish messages to the queue.
<figure class="wp-block-image"><img alt="queues and how they are used for process scheduling" class="aligncenter size-full wp-image-186" height="1069" src="../../../images/2026/05/07.png" width="1402"/></figure>

There are actually several such queues (just like in a modern Linux kernel, but in hardware) - for normal tasks, reconfiguration tasks, urgent processes, and diagnostic ones. There's even a queue for sleeping processes that are supposed to wake up at a specific time (the scheduler just moves them back into the normal process queue). And the queue can be either FIFO or priority-based. I'll be honest - I didn't experiment much in this area. With only single process running on single processor in my system, a lot of this I know only in theory rather than from hands-on experience. The one thing I did bump into was the scheduler trying to kick in after the process's time quantum expired, which I didn't want. I had to cheat by running the processor's tick counter clock at a very low frequency - the iAPX 43202 needs a separate clock signal for its internal purposes, which can be arbitrary and is used purely by the scheduler.
To sum up: to run a user program you need to prepare an initial directory containing the object table addresses, the tables themselves, and define a number of core objects - the processor (which actually consists of several linked segments), a process, queues, messages in those queues, and code/data segments. That's quite a bit, but by using the memory access log you can figure out which fields are actually critical to the processor and only fill in those, ignoring a chunk of the architectural complexity.
## Organizing user code and data

A process's executable code is spread across several Context objects. Think of them as functions - or more accurately, procedures. A context is made up of code segments (Instruction objects) and 4 lists of data segments (those lists are called "entry access segments", EAS). Code can only access variables that live in segments defined in the EASes. This is how isolation is achieved - to access data from another context, you have to call the appropriate instruction that imports an EAS from that context (assuming the rights allow it, of course).
<figure class="wp-block-image"><img alt="context and its entry access segments" class="aligncenter size-full wp-image-187" height="721" src="../../../images/2026/05/08.png" width="1201"/></figure>

Calling a procedure (context) is an expensive operation. The iAPX 432 needs to prepare a bunch of objects describing the context (not just the corresponding data and access segments, but also the stack segment, for example), which takes around 20-30 memory accesses. This is, incidentally, one of the bottlenecks in Ada programs written for this system. The compiler distributed code quite suboptimally - the ISA has instructions for transferring control to code residing inside a context, so it wasn't always sensible to create a new context and pay the heavy price of a subroutine call.
In my low-level code I use just one context and one code segment (I didn't write any large programs - everything fit comfortably within 64KB). One data segment was also sufficient (plus the stack), although an EAS can hold 16384 references, which allows addressing 4 * 16k * 64kb = 4GB. Not bad for a chip from the 80s.
Since I wanted to explore the raw performance of the system, I planned to write code in "assembly" rather than trying to coax decent output from Ada compilers. Which means you need to understand the 432's programming architecture before writing your programs and the compiler to translate them into machine code.
I've already mentioned some things - no accessible registers except a single 16-bit top-of-stack, and a stack that (interestingly) grows upward rather than downward like ARM or x86. There are no familiar PUSH/POP instructions either - the GDP modifies the stack pointer itself whenever an instruction references data from the stack.
The processor supports various operand types - both regular integers (up to 32 bits!) and floating-point numbers (the iAPX 432 was one of the first to support the floating-point format that would later be standardized in the IEEE 754 spec). And of course there are instructions for working with various objects: system-level (like the `LOCK OBJECT` mentioned earlier) and user-defined.
The machine code format doesn't support encoding immediate values, which means most operands are references to values in memory. So how does variable addressing work? It's not simple, unfortunately - every reference consists of two parts: a selector for the data segment containing the variable, and an offset within that segment. In its simplest form, the selector is encoded as an EAS index in the context (one of the 4) and an index of the pointer to the target segment. So even the most trivial use of a constant in code turns into multiple memory accesses. "Efficiency". There's also indirect segment addressing - where we take a selector from the machine code, compute the address of a memory cell, which in turn contains another selector that finally points to the segment holding our variable.
<figure class="wp-block-image"><img alt="overcomplicated access to variables" class="aligncenter size-full wp-image-188" height="883" src="../../../images/2026/05/09.png" width="1057"/></figure>

The variable's offset within the data segment can also be more complex than just a number. A basic example - accessing an array element by an index that is itself a variable. That's supported.
As you can see, nothing extraordinary - we can go ahead and start writing a simple compiler. But I had performance in mind from the start and took a few optimization steps upfront. First and foremost, you want to make maximum use of the single register, and when that's not possible, go for the simplest addressing modes with the fewest memory reads.
Another important consideration when writing the assembler is that the machine code uses a variable-length instruction format, and the instructions aren't even byte-aligned! An instruction can be encoded in as (6!) bits or for instance as two hundred (200!). From a programming standpoint this matters because shorter instructions mean more compact code and fewer memory accesses when the processor fetches it.
## Programs for the iAPX 432

As always, let's start with Hello World. But how do we actually get output to the screen? We have an FPGA sitting on the bus with the GDP, but how do we detect the processor's command to send text to the console? The standard way is to initiate IPC via the "BROADCAST TO PROCESSORS" or "SEND TO PROCESSOR" instructions, but then we'd need to emulate another processor in the system for the GDP to send that IPC message to. Luckily there's an easier way - the "MOVE TO INTERCONNECT" instruction simply writes a value to an inter-processor register, without requiring any complex pre-built structures in memory. It's literally just one bus transaction, and the FPGA can intercept a write to a specific address (the GDP only uses 0x0 and 0x02) and forward the data over UART to our PC.
```python
.stack {
  size = 0x10
  data = []
}
.data {
  msgIdx = { size = 2, data = [0x06, 0x00] }
  # reversed, because we start sending data from the end
  msg = { size = 12, data = [0x21, 0x64, 0x6c, 0x72, 0x6f, 0x57, 0x20, 0x6f, 0x6c, 0x6c, 0x65, 0x48] }
  # variables for sending data via UART
  interconnectRegUart = { size = 2, data = [0x02, 0x00] }
  interconnectSegmentSelector = { size = 2, data = [0x28, 0x00] }
}
sendTwoChars:
  MOVE_TO_INTERCONNECT interconnectSegmentSelector interconnectRegUart $data[msgIdx]
  # array is iterated in range [msgLen ... 1], because we want to reference uart payload from base 0
  # and need to skip element at index 0 (it's reserved for uart payload length)
  DEC_2U msgIdx msgIdx
  EQUAL_ZERO_2U msgIdx $st0
  BRANCH_FALSE $st0 sendTwoChars
  RETURN_FROM_CONTEXT
```

The code is trivial even for people unfamiliar with the iAPX 432. Let me just clarify a couple of things.
I introduced the pseudo-variable $data specifically to optimize code size. Addressing a variable from the start of the segment lets us avoid encoding the array's base offset. Using `msg[msgIdx]` instead would add an extra 7 bits to encode 0x02 (the offset of the msg variable in the data segment). In this case it's penny-pinching, but I wanted to show a concrete example.
You'll also notice that a conditional branch takes 2 instructions - first we compare the variable to zero and write the comparison result into another variable (in this case the stack, specifically the top-of-stack register), then we branch based on the value of that result.
<figure class="wp-block-image"><img alt="successful hello world" class="aligncenter size-full wp-image-189" height="653" src="../../../images/2026/05/10.png" width="1467"/></figure>

`BRANCH_TRUE msgIdx sendTwoChars` would not work because `BRANCH_TRUE` operates on 8-bit values and `msgIdx` is 16 bits. It's not that the iAPX 432 checks the width - it's just basic arithmetic: casting a 16-bit value to 8-bit drops information and breaks the logic. We definitely don't want an early exit at `msgIdx = 0x100` :)
Now we finally get to the reason all this was started - writing a benchmark. As usual I went with a Pi digit computation. Specifically, the spigot algorithm. It's extremely simple to implement but does a good job of exercising ALU performance.
```python
.stack {
  size = 0x20
  data = []
}
.data {
  idx = { size = 2 }
  arr = { size = 55000 }                       # iteration in range [1 .. (LEN - 1)],
                                               # array length for 8192 digits is 27307, size should be 54614
  ### global variables
  toPrint = { size = 2, data = [0x00, 0x20] }        # amount of digits to print
  # toPrint = { size = 2, data = [0x00, 0x08] }
  # toPrint = { size = 2, data = [0x00, 0x01] }
  # toPrint = { size = 2, data = [0x0A, 0x00] }
  LEN = { size = 2, data = [0x00, 0x00] }            # length of array - 1
  nineCount = { size = 2, data = [0x00, 0x00] }      # count of consecutive 9s
  previousDigit = { size = 2, data = [0x02, 0x00] }  # previous digit
  ### local variables for inner loops
  carry = { size = 4 }
  denominator = { size = 4 }
  numerator = { size = 4 }
  digitFromCarry = { size = 2 }
  nextDigit = { size = 2 }
  ### constants
  c10 = { size = 4, data = [0x0A, 0x00, 0x00, 0x00] }  # constant 10
  c3 = { size = 4, data = [0x03, 0x00, 0x00, 0x00] }   # constant 3
  c2 = { size = 2, data = [0x02, 0x00] }   # constant 2
  c9 = { size = 4, data = [0x09, 0x00, 0x00, 0x00] }
  ### variables for sending data via UART
  interconnectRegTiming = { size = 2, data = [0x00, 0x00] }
  interconnectRegUart = { size = 2, data = [0x02, 0x00] }
  interconnectSegmentSelector = { size = 2, data = [0x28, 0x00] }
}
  MOVE_TO_INTERCONNECT interconnectSegmentSelector interconnectRegTiming c2
### initialization
  MUL_4U toPrint c10 $st0    # stk[0] = toPrint * 10, sp = 4 (toPrint is 2b, so LEN would be used as high part for operation)
  DIV_4U c3 $st0 $st0        # stk[0] = stk[0] / 3
  SAVE_4U LEN                # LEN = stk[0] (LEN is 2b, so high part, which is 0x0000, would be saved to nineCount)
  MOVE_4U $st0 idx           # idx = stk[0], sp = 0 (idx is 2b, so high part would be saved as first element for an array)
  MOVE_2U c2 $st0            # stk[0] = 2, sp = 2
init_array:
  SAVE_2U $data[idx]              # arr[idx] = stk[0]
  DEC_2U idx idx                  # idx--
  EQUAL_ZERO_2U idx $st0          # stk[2] = (idx == 0), sp = 4
  BRANCH_FALSE $st0 init_array    # if (stk[2] === false) goto init_array, sp = 2
  # XXX: only way to pop value from stack without extra access to memory
  BRANCH_TRUE $st0 main_loop     # sp = 0
main_loop:
  ZERO_4U carry                  # carry = 0
### computation loop
  MOVE_2U LEN denominator                       # denominator = LEN
  ADD_2U denominator denominator $st0           # stk[0] = denominator + denominator, sp = 2
  INC_2U $st0 numerator                         # numerator = stk[0] + 1, sp = 0
update_loop:
  CONVERT_2U_4U $data[denominator] $st0         # stk[0] = arr[denominator], sp = 4
  MUL_4U $st0 c10 $st0                          # stk[0] = stk[0] * 10
  ADD_4U $st0 carry $st0                        # stk[0] = stk[0] + carry
  SAVE_4U $st0                                  # stk[4] = stk[0], sp = 8
  REMINDER_4U numerator $st0 $st0               # stk[4] = stk[4] % numerator
  CONVERT_4U_2U $st0 $data[denominator]         # arr[denominator] = stk[4], sp = 4
  DIV_4U numerator $st0 $st0                    # stk[0] = stk[0] / numerator
  MUL_4U denominator $st0 carry                 # carry = denominator * stk[0], sp = 0
  DEC_2U numerator $st0                         # stk[0] = numerator - 1, sp = 2
  DEC_2U $st0 numerator                         # numerator = stk[0] - 1 (numerator -= 2), sp = 0
  DEC_2U denominator denominator                # denominator--
  EQUAL_ZERO_2U denominator $st0                # stk[0] = (denominator === 0), sp = 2
  BRANCH_FALSE $st0 update_loop                 # if (stk[0] === false) goto update_loop, sp = 0
### output digits
  MOVE_2U carry $st0                    # stk[0] = carry, sp = 2
  SAVE_2U $st0                          # stk[1] = stk[0], sp = 4
  GREATER_THAN_2U c9 $st0 $st0          # stk[1] = stk[1] > 9
  SAVE_2U digitFromCarry                # digitFromCarry = stk[1]
  BRANCH_FALSE $st0 nextDigit_computed  # if (stk[1] === 0) skip decrement, sp = 2
  SUB_2U c10 $st0 $st0                  # stk[0] = stk[0] - 10
nextDigit_computed:
  SAVE_2U nextDigit                     # nextDigit = stk[0]
  EQUAL_2U $st0 c9 $st0                 # stk[0] = stk[0] === 9
  BRANCH_FALSE $st0 print_digits
  INC_2U nineCount nineCount
  BRANCH main_loop
print_digits:
  ADD_2U previousDigit digitFromCarry $st0
  MOVE_TO_INTERCONNECT interconnectSegmentSelector interconnectRegUart $st0
  DEC_2U toPrint toPrint
  MOVE_2U nextDigit previousDigit
  EQUAL_ZERO_2U nineCount $st0
  BRANCH_TRUE $st0 check_done
print_nines_loop:
  # either output 0x0009, or 0x0000, based on digitFromCarry
  MOVE_TO_INTERCONNECT interconnectSegmentSelector interconnectRegUart c9[digitFromCarry]
  DEC_2U toPrint toPrint
  DEC_2U nineCount nineCount
  EQUAL_ZERO_2U nineCount $st0
  BRANCH_FALSE $st0 print_nines_loop
check_done:
  EQUAL_ZERO_2U toPrint $st0
  BRANCH_FALSE $st0 main_loop
### end of program
  MOVE_TO_INTERCONNECT interconnectSegmentSelector interconnectRegTiming c2
  RETURN_FROM_CONTEXT
```

Again, the listing shouldn't raise too many questions, especially with the manual stack tracing. But let me point out a few things.
- You can see clearly that the iAPX 432 has absolutely no control over how scalar values are used. Despite all its object-orientedness, the GDP lets you treat ints as shorts. Or even use them as arrays (have a look at `c9[digitFromCarry]`).
- The only way I found to decrease the stack pointer without an extra memory access is: `BRANCH_TRUE $st0 main_loop`.
- And one last thing - division and remainder are two separate operations, unlike in other architectures.

## Benchmarks and conclusions

<figure class="wp-block-image"><img alt="comparision table with other CPUs" class="aligncenter size-full wp-image-190" height="1502" src="../../../images/2026/05/11-scaled.png" width="2560"/></figure>

At this point, the iAPX 432 has taken first place in my performance chart. It even turned out to be faster than the Intel 8080, which was computing Pi using the considerably faster and more sophisticated Chudnovsky algorithm.
Though it's not entirely fair to compare processors from different generations. What about a contemporary - the 8086? Unfortunately my 8086 system is currently out of order, but I used a cycle-accurate emulator - 86Box. And it turns out the iAPX 432 is on average 2.5x faster!
How is that possible? A few factors played into it - I deliberately avoided many of the performance pitfalls induced by the Ada compiler. Also, while the ALU in the iAPX 432 is 16-bit, its performance on both 16-bit and 32-bit operations is still higher than the 8086's. Another possible factor is that I have no extra wait cycles for memory access since it's fast enough - though in the emulator I also tried to pick a system with similar characteristics, so that factor is probably negligible.
I can't say I enjoyed programming for the iAPX 432, but the journey to getting my own code running on this machine was genuinely satisfying.
All necessary stuff to replicate my work is hosted in my [github repo](https://github.com/quasiengineer/iapx432-sbc)
You also can find more information about iAPX 432 in my youtube videos (they are a bit more technical and dive deeper than this article):
<figure class="wp-block-embed is-type-video is-provider-youtube wp-block-embed-youtube wp-embed-aspect-16-9 wp-has-aspect-ratio">
<div class="wp-block-embed__wrapper"><iframe allowfullscreen="allowfullscreen" frameborder="0" height="563" src="https://www.youtube.com/embed/krHs7h7xZIY?feature=oembed" width="1000"></iframe></div>
</figure>

<figure class="wp-block-embed is-type-video is-provider-youtube wp-block-embed-youtube wp-embed-aspect-16-9 wp-has-aspect-ratio">
<div class="wp-block-embed__wrapper"><iframe allowfullscreen="allowfullscreen" frameborder="0" height="563" src="https://www.youtube.com/embed/38u6GqKjusc?feature=oembed" width="1000"></iframe></div>
</figure>

<figure class="wp-block-embed is-type-video is-provider-youtube wp-block-embed-youtube wp-embed-aspect-16-9 wp-has-aspect-ratio">
<div class="wp-block-embed__wrapper"><iframe allowfullscreen="allowfullscreen" frameborder="0" height="563" src="https://www.youtube.com/embed/nyYMQzDJiHg?feature=oembed" width="1000"></iframe></div>
</figure>
