











![](https://cloudfront.codeproject.com/emulation/795037/c64-screenshot.png)

Background
----------

This repository implementations Commodore 64 emulator in C#.

The performances are not that great, because of several reasons, like cycle-based and real drive emulation as well as the way some things are implemented in source code. I re-implemented the emulator in C++ for other purposes and the implementation yielded a great increase of performance. In the wake of these results I abandoned the C# implementation. Since it would be a waste time and effort not to use the code available in some way, I decide to write an article about the subject.

Unfortunately there are stuffs that will remain incomplete, missing or implemented correctly, but it is still a good base to explain the basic concept of emulation.

This document will not try to explain how the actual hardware works. There is a huge amount of great resources that cover these subjects. Instead it will be focused on the emulation - what should be done and how it was done in this implementation.

Before you try to build the solution or run the emulator you should read [section about ROM files](#rom_files).

Introduction
------------

Parts of Commodore 64 that are visible and can be used by the programmer are:

*   6510 chip - the CPU (same as 6502 chip but with one additional IO port)
*   VIC-II chip - responsible for graphic
*   SID chip -responsible from audio
*   2xCIA chips - responsible for timers and IO (like serial bus, keyboard, joystick...)
*   Color RAM - 4-bit memory used by VIC-II chip to generate colors of displayed graphic
*   RAM chips - 64 kilobytes available to program
*   3xROM chips - storing Basic interpreter, OS (KERNAL) and character generator used by VIC-II chip

The implementation covers the most of the hardware needed to run games. The one major thing that is missing is SID chip emulation. REU and tape emulation is also not available. With appropriate tools it should be possible to convert games that are available in T64 format to D64 (disk) format, so those games should be playable too.

Emulator is cycle-based, which allows perfect emulation of chip timings and synchronization among different chips in the system. It also implements real-drive emulation of 1541-II drive. All these techniques provide better compatibility with different software, but they come with a performance costs.

<img width="591" height="409" alt="image" src="https://github.com/user-attachments/assets/105908a6-afc1-46ad-8b4f-48d952de88ce" />

_**Commodore 64 Block Diagram**_

### Project Structure

Emulation code is separated in several assemblies so some things, like CPU, memory and IO port emulation, can be reused for other projects like VIC, PET or NES emulators.

*   `c64_common` - clock, memory, IO port and interrupt line emulation
*   `c64_cpu` - 6502 and 6510 emulation
*   `c64_av` - VIC-II and SID chips emulation
*   `c64_io` - CIA chip and IEC (serial) bus emulation
*   `c64_system` - C64 board that connects all emulated hardware as well as keyboard/joystick emulation
*   `c64_1541` - 1541-II drive emulation
*   `c64_environment` - definition of interfaces used by emulator to interact with external world (like outputting graphic and audio, reading files...)
*   `c64_win_gdi` - implements GUI and connects everything together (C64 board, 1541-II drive and environment)

Clock
-----

Everything in the system is synchronized by the CPU clock. Clock cycles are separated into phases. These phases define the order of chip emulation in each cycle and each chip takes a single phase of the cycle. In each phase emulator execute single clock operation scheduled for the chip. There can be any number of phases in a cycle depending on the number of chips that are emulated.

<img width="529" height="431" alt="image" src="https://github.com/user-attachments/assets/5d9b4af7-7bb6-40c8-b65b-a1f15100a988" />

_**System Clock Block Diagram**_

`ClockOp` defines the interface for clock operation where `Execute` should contain the logic that should be executed in the clock cycle.

`Clock` class represent clock of the system, it keeps queue of clock operation that should be executed for each phase. `Clock` does not work with `ClockOp` directly but through `ClockEntry` and `ClockEntryRep` classes. These classes provide additional context to `ClockOp` like what is the next operation that should be executed, and whether that instruction should be executed in same cycle. `ClockEntryRep` class is used for operations that take multiple cycles to complete and contains information about its length - number of cycle it takes to execute.

`QueueOpsStart` and `QueueOps` methods are used for queuing operations for specified phase of the clock. `QueueOpsStart` method is used when the first operations are queued after the emulation is started, for every subsequent request `QueueOps` should be used. For most chips calling `QueueOpsStart` once at the beginning, with all clock operations specified in cyclic list is usually enough. Only CPU is using `QueueOps` method to queue operations that should be executed after it decodes the instruction.

`Stall` methods can be used to stall execution of queued operations in a certain phase (i.e. holding CPU while VIC-II chip is accessing RAM) or `Prolong` if additional cycles are required to complete operation (i.e. executing branch instruction that causes cross-page jump).

Once everything is ready to start/resume the emulation, `Run` method of the `Clock` should be called. `Halt` method can be used for halting execution of the operations queued in the clock. This is used for holding emulation after the frame is completed and before new one should be rendered, or for stopping drive emulation when drive is not active to save resources.

Once the last phase of the current cycle is completed, `OnPhaseEnd` event is raised. This event can be used to chain additional clock of other boards like 1541-II drive.

<img width="580" height="940" alt="image" src="https://github.com/user-attachments/assets/5dfd6df8-f686-45fe-b094-5a2c24fe681a" />

_**Clock emulation class diagram**_

Memory Model
------------

This chapter will describe how the emulation of memory bus, ROM and RAM is implemented. It will also explain what memory mapped devices are and how memory map is working in this implementation.

Memory mapped devices
---------------------

Each device that can be accessed through the memory bus is memory mapped device (including RAM and ROM). They are mapped to a certain address range which is specified by start address and the size (number of memory location).

`MemoryMappedDevice` class is base for all memory mapped devices. These devices should implement and support `Read` and `Write` methods which are called when CPU or some other chip uses memory bus to read or write content of address to which the device is mapped.

Memory map
----------

Memory map represents a memory configuration. This configuration defines which devices are mapped and where they are mapped in the address space of the system. Important thing to notice is that write operations can trigger different chip from read operations for the same memory location even when the same memory map is used. Due to this requirement, this implementation has two separate maps, one for writes, and one for reads.

<img width="492" height="182" alt="image" src="https://github.com/user-attachments/assets/8d715ed9-583d-41cd-9131-2efb2f8727e5" />

_**Memory Map Block Diagram**_

Commodore 64 can have different memory maps that can be selected by the programmer. To accomplish this, emulator will create multiple memory maps are pre-configured for each possible configuration.

Each location is represented by `MemoryMapEntry` class and it has two entries - device that should be called for writes and device that should be called for reads.

`MemoryMap` class handles memory mappings and it's just an array of memory map entries. It has `Map` and `Unmap` methods that can map a device in specified address space and remove it from there. Another important pair of methods is `Read` and `Write`. These methods will find which device is mapped at the specified location for the requested memory operation and invoke appropriate methods on the device.

<img width="584" height="801" alt="image" src="https://github.com/user-attachments/assets/5d90d7e4-0679-4f60-a2a8-a3ee06efb322" />

_**Memory emulation class diagram**_

RAM
---

RAM emulation is as simple as it gets. Memory is just an array of bytes which can be read or written to by the CPU or some other chips like VIC-II.

`RAM` class implements emulation of RAM. Since the RAM is obviously mapped into address space, this class is based on `MemoryMappedDevice` class. `Read` and `Write` methods are straightforward and do as they say - read or write memory location.

ROM
---

ROM emulation is even simpler, implementation just provide reading of bytes that were loaded from predefined file.

Due to hardware implementation of memory bus in Commodore 64, stores to locations which are currently mapped to ROM will result in values being written to RAM at these location. This feature is realized by memory map of the emulator and not by the ROM emulation, which just ignore writes.

ROM emulation is implemented by `ROM` class which is similar to `RAM` class. It also provides additional method `Patch` which allows content of ROM to be changed which can be useful for skipping various checks performed by firmware.

Color RAM
---------

Commodore 64 has additional RAM dedicated for storing colors of all characters that are currently being displayed and it is used by VIC-II chip to generate video output. Only lower 4 bits of each byte is used by VIC-II.

Emulation of Color RAM is implemented in `ColorRAM` class and it based `RAM` class.

Shadowing
---------

In some cases, single device can be mapped to multiple locations, but this is not supported by the current implementation. Also chips are usually mapped to much larger space then they have registers. For instance VIC-II address size is 1024 bytes, but it only has 64 registers. In these cases it is up to the chip to determine what it should do with reads/writes to locations beyond available register, but most often only the first few bits of the address are significant and others are assumed to be 0. This makes memory map looks like those 64 registers are mapped multiple times in the allotted address space to fill whole 1024 bytes of address space.

Unmapped address space
----------------------

Certain memory configurations allow address ranges that do not have any devices mapped. In real hardware, writes to these locations would be ignored and reads would return predefined values. In this implementation an exception would occur, which can cause problems with some of the existing software.

IO ports
--------

IO ports are available in several chips: VIA, CIA and 6510 CPU. Pins on a port can be either input or output. Direction of the pin can be controlled programmatically. These ports are used for connecting hardware that is not connected to memory bus directly - they are not memory mapped. These are devices like serial bus, keyboard and joystick in Commodore 64 or drive head controller in 1541-II disk drive...

Logic for controlling IO ports is provided by `IOPort` class, since it is the same for all chips.

Properties `Input`, `Output` and `Direction` provides access to pins of IO port and allows control of their direction as well as their state. These properties will set state of all pins to specified values which is not always desirable. `SetSingleInputFast` method can be used when there is a need to change the state of single input pin.

In addition to methods that can read or set state of pins, or change their direction, this class also exposes event when the state of output pins are changed - `OnPortOut`.

How these ports are controlled and exposed to the rest of the system depends on the chip logic of which they are part.

<img width="281" height="342" alt="image" src="https://github.com/user-attachments/assets/32e1f681-8eea-423b-b996-d2cce8a98e54" />

_**IOPort class**_

CPU
---

Commodore 64 computer is powered by MOS6510 CPU. This version of CPU is exactly the same as MOS6502 chip, but with an extra general purpose IO port that is mapped into address space. This IO port is used for controlling tape drive and memory configuration of the system.

Only documented opcodes are implemented in the emulator. Execution of illegal opcodes (not documented by MOS) will cause emulator crash.

`MOS6502` class connects all the individual components of CPU emulation. `MOS6510` class extends `MOS6502` and it only implements emulation of additional IO port available on that chip.

CPU Registers
-------------

6502 has set of 6 registers that are visible to the programmer. This set is represented by `CPUState` class.

Each register implements `IRegister` generic interface, where the underlying type of the interface depends on the size of the register.

Accumulator (A) and index (X and Y) registers are implemented by `GpRegister` class. Stack register and program counter are implemented by `StackRegister` and `ProgramCounter` classes, respectively.

Processor status register is implemented by `StatusRegister` class where each status flag is exposed as property.

<img width="585" height="875" alt="image" src="https://github.com/user-attachments/assets/362640e1-68bc-4fb9-bcbb-da81a83cb6e9" />

_**CPU registers class diagram**_

Decoding
--------

Fetching and decoding opcode is the first step of instruction execution. In case of 6502 CPU all opcodes are one byte long and it is enough to decode whole instruction - so we know which instruction it is and what addressing mode it uses.

Decoding process uses pre-created lookup table which maps opcodes to structure that indicates which addressing mode should be invoked and which instruction should be executed as well as their timings - number of cycles it takes to address, execute and store result. This table is implemented by `DecodingTable` class.

Decoder will then create list clock operations that will be queued for the execution based on the information obtained from decoding table.

Decoder itself is implemented as clock operation (`DecodeOpcodeOp` class) and it is attached at the end of clock operation list for every decode instruction.

Addressing
----------

Next step, after the instruction has been decoded, is to determine operands. Operands tell the source and/or destination of the instruction.

6502/6510 CPU has several addressing modes, which are not going to be discussed here, since there are far better sources on the subject, including the official datasheet.

Each addressing mode is implemented as separate class which implements `AddressingMode` interface. The only method defined is `Decode` which is responsible for calculating target address depending on current state of the processor.

<img width="552" height="797" alt="image" src="https://github.com/user-attachments/assets/88920d82-6a7f-4d75-a0dc-deccf0294237" />

_**Class diagram of addressing modes**_

Address decoding process will create and set active target of CPU depending on the addressing. Target can be writable or only readable; it can be memory or register. Target is defined by `AddressedTarget` interface it should implement logic for retrieving and storing values of the targeted location and each addressing mode has its own kind.

`ReadableTarget` class is used by the addressing modes that only require reading memory location, while `WritableTarget` class is used by those modes that also requires storing values to the targeted location. `IndirectTarget` is used by addressing modes which does not reference target directly but through another, intermediate, memory location that stores the actual target location.

There are also targets specific for certain addressing modes, like accumulator addressing which targets accumulator register of the CPU and immediate addressing which specify that target is part of decoded instruction.

It is important to note that some addressing modes require fetching additional bytes from memory referenced by program counter to decode target location and each fetch causes moving program counter to the next address.

<img width="590" height="753" alt="image" src="https://github.com/user-attachments/assets/8fc36c28-85d6-46f6-b4b1-345ca41ee244" />

_**Class diagram of addressing targets**_

| Addressing mode                      | Addressing class name | Target class name                          |
|--------------------------------------|-----------------------|--------------------------------------------|
| Accomulator                          | `AccAddressing`       | specific target (`WritableTarget` as base) |
| Immediate                            | `ImmAddressing`       | specific target (`ReadableTarget` as base) |
| Absolute                             | `AbsAddressing`       | `WritableTarget`                           |
| Zero Page                            | `ZeroAddressing`      | `WritableTarget`                           |
| Indexed Zero Page (using X register) | `ZeroXAddressing`     | `WritableTarget`                           |
| Indexed Zero Page (using Y register) | `ZeroYAddressing`     | `WritableTarget`                           |
| Index Absolute (using X register)    | `AbsXAddressing`      | `WritableTarget`                           |
| Index Absolute (using Y register)    | `AbsYAddressing`      | `WritableTarget`                           |
| Implied                              | `ImpAddressing`       | none                                       |
| Relative                             | `RelAddressing`       | `WritableTarget`                           |
| Indexed Indirect                     | `IndAddressing`       | `IndirectTarget`                           |
| Absolute Indirect(using X register)  | `IndXAddressing`      | `IndirectTarget`                           |
| Absolute Indirect(using X register)  | `IndYAddressing`      | `IndirectTarget`                           |

Address decoding is wrapped in `DecodeAddressOp` clock operation class so it can be queued for the execution.

How each of these addressing modes works is described in 6502's datasheet.

Instructions
------------

Each class that represents an instruction needs to implement `Instruction` interface which exposes `Execute` method. This is the method which is responsible for executing the logic of the instruction.

Execution phase of the instruction is wrapped into `DecodeAddressOp` class which represent queueable clock operation. This wrapper provides all the necessary information to `Execute` method, such as CPU that executes the instruction, current cycle needed for instruction that take multiple cycles and so on...

Interrupts
----------

6502/6510 CPUs have two interrupt lines - standard one that is maskable (IRQ) and non-maskable (NMI). IRQ line is level triggered - meaning that CPU will detect interrupt every time the level of the line is raised, while the NMI line is edge triggered - meaning the CPU will recognize only when the level of the line changed from low to high.

Interrupt line emulation is realized by `IrqLine` and `NmiLine` classes. Unfortunately `NmiLine` does not handle multiple interrupt sources correctly.

CPU is the owner of these lines, but chips that are attached to them can get the references and use them to raise or lower the levels of the line.

Clock operation that is responsible for instruction decoding (`DecodeOpcodeOp`) checks interrupt lines before it fetches the next instruction. If any of the lines are active it will enqueue clock operation that is responsible for interrupt handling. In the case of maskable interrupt (IRQ) this operation will check process status register before it starts interrupt handling.

Interrupt handling clock operation is implemented by `InterruptOp` class.

<img width="590" height="844" alt="image" src="https://github.com/user-attachments/assets/94636577-9297-450c-8d4f-830bf4b0f7f0" />

_**Class diagram of CPU's different clock operations**_

6510 IO port
------------

The only addition to 6510 is IO port. The port is always mapped to addresses `$0000` (pin direction register) and `$0001` (pin state register). Exact function of each pin is provided in the references.

<img width="593" height="968" alt="image" src="https://github.com/user-attachments/assets/e774214b-58fc-4d90-9208-4109564d310a" />

_**MOS 6502/6510 emulation Class diagram**_

Commodore 64
------------

The following section discusses emulation of individual chips and hardware components that are present on Commodore 64's main board.

VIC-II chip
------------

This chip is responsible for generating graphic in Commodore 64. Although there is no official datasheet for VIC-II chip, there is an article with the detailed description of internal workings of the chip which is a product of reverse engineering. Link to the article is provided in the section with references. It would be good for a reader to get familiar with the subject before getting into details of the actual implementation.

VIC-II emulation is implemented by `VIC` class. Since the chips is mapped in address space and accessed through the memory bus, `VIC` class have to extend `MemoryMappedDevice`. `Read` and `Write` methods are responsible for retrieving state of chip's register and updating them.

`RasterLine` class is responsible for managing the well-defined sequence of operations performed by the chip in each cycle.

Each clock cycle is separated in two steps - memory access operation and graphic generation operation. Some cycles have both operations defined, others have only one or none. To cover all the cases there are several implementations clock operations for VIC-II chip:

*   `VicNop` - cycles where the whole chip is idle
*   `VicGraphOp` - cycles where only the graphic generator is active, but no memory access is needed by the chip
*   `VicReadOp` - cycles where graphic generator is idle, but the memory access is required by the chip
*   `VicGraphReadOp` - cycles that requires memory access and have graphic generator active
*   `VicReadOp62` - special case of `VicReadOp` which used only for the last cycle of raster line

Raster line has two pre-defined lists of clock operations. One list represents operations that should be performed in the raster line when graphic output is active and the other list contains operations that are performed by VIC-II chip when graphic output is idle - during rendering borders or deactivated programmatically by the programmer.

<img width="593" height="1383" alt="image" src="https://github.com/user-attachments/assets/1c01c9e1-77be-4ab9-8bb4-bd49382a7cd2" />

_**Raster line emulation class diagram**_

As it already mentioned, each cycle has strictly defined set of actions that it performs. Logic of each cycle for memory access and graphic generator operations are implemented as different methods of `RasterLine` class. These methods are responsible for updating state of the chip, performing memory reads and invoking active graphic mode to generate graphic.

Graphic modes are responsible for generating all graphic output except sprite graphic. Each graphic mode has its own class that implemented `GraphiceMode` interface. The only method of the interface is render, which would generate pixel for the current position according to current state of the chip.

<img width="497" height="672" alt="image" src="https://github.com/user-attachments/assets/f75afb66-81ae-4299-a87a-1337d3ea1560" />

_**Class diagram of VIC-II's graphic modes**_

`RasterLine` class is also responsible for generating graphic output for sprites.

`Output` method of `IVideoOutput` interface is called when the chip needs to output pixel on the screen.

VIC-II object also keeps collision matrix to implement collision detection feature of the chip. This matrix stores information for each pixel and it defines whether the pixel can cause collision with another sprite or not. This information is stored when pixel of background graphic is outputted and checked when sprite graphic is outputted.

<img width="565" height="1899" alt="image" src="https://github.com/user-attachments/assets/f64bd218-3ca1-47c9-9a9c-3051144bf98d" />

_**VIC-II chip emulation class diagram**_

CIA chip
--------

CIA chip has two IO ports (A and B), two timers (A and B), one clock (Time of Day or TOD) and serial pin which is unused in Commodore 64. These chips are responsible for connectivity with IO devices such as keyboard and joysticks that are connected directly and disk drives and printers that are through IEC bus.

The chips are attached to interrupt lines so they can raise interrupts when required, i.e. when the state of input pin is changed, timer reached 0, etc. CIA #1 is attached to IRQ line while CIA #2 is attached to NMI line, but in this implementation it is also attached to IRQ line incorrectly.

Both CIAs are mapped into address space of Commodore 64, so they can be accessed through memory bus.

For the details about each component of CIA chip you can find link to CIA datasheet in the references.

Emulation of the chip is implemented by `CIA` class. This class extends `MemmoryMappedDevice` since CIAs are mapped into memory space. `Read` and `Write` method reads and updates registers of the chip.

`CIA` class also implements `ClockOp` interface which is responsible for updating timers' counters. In some cases CIA chip will be idle for a few cycles when the timer is reloaded, but they are not covered by this implementation.

Timer emulation is implemented by `TimerState` class.

`IncrementTod` method which is responsible for updating Time of Day is called after end of each frame.

<img width="588" height="874" alt="image" src="https://github.com/user-attachments/assets/555b4a2f-dac5-4035-a352-80fd754b3e25" />

_**CIA chip emulation class diagram**_

SID chip
--------

SID chip emulation is not implemented, unfortunately, and it will not be discussed here. Link to SID datasheet is provided in the reference section of the article.

Memory Configurations
---------------------

Commodore 64 has several memory configurations and appropriate configuration can be selected by setting the first 3 bits of 6510's IO port. Details about each configuration can be found in referenced articles. Each configuration, of 8 possible, has its own instance of `MemoryMap` which is pre-configured at the startup. This improves performance since it is enough to set reference to appropriate instance of `MemoryMap` when the memory configuration is changed by writing the IO port.

Keyboard and Joystick
---------------------

In Commodore 64, keyboard and joysticks are connected to IO ports (A and B) of CIA #1 chip. Keyboard forms a matrix which can be decoded based on state of output port A and input port B. _Restore_ is attached directly to NMI line (this is not implemented). `Keyboard` class is responsible for receiving key presses from the system, converting them according to the matrix and the output port A and setting the state of input port B.

IEC bus
-------

IEC bus is used for communication with disk drives, such as 1541-II, and printers, among the other things. It has three lines: DATA, CLOCK and ATN. These lines are attached to IO port B of CIA #2. Even though CIA chip has hardware support for serial communication, this feature is not used, but the protocol is implemented in software. `SerialPort` class implements serial bus emulation, where each line is represented by `BusLine` class. State of the line can be controlled with `State` property of `BusLine` class. The class tracks how many devices keeps the line low and raises an event when the state of the line is changed. Devices are not attached to the line directly but through and object of `BusLineConnection` class which is responsible for tracking local state - state of the line set by that device.

<img width="292" height="758" alt="image" src="https://github.com/user-attachments/assets/3e0e6f14-475b-4584-8e50-265c57e018ca" />

_**IEC bus emulation class diagram**_

1541-II Drive
-------------

The emulator also implements real drive emulation of 1541-II disk drive. This means that emulation is not on IEC bus protocol level but that the real hardware of the drive is emulated. This type of emulation requires more resources, but provides much better compatibility.

<img width="537" height="371" alt="image" src="https://github.com/user-attachments/assets/21976645-6fa1-4534-ba39-04b5f4997522" />

_**1541-II drive block diagram**_

GCR Coding and D64 Format
-------------------------

GCR is format used by Commodore to store information on disks. It defines how the data are laid on disk: track sizes, data synchronization, how sector headers looks like and coding of sector data. Link that contains detailed explanation of the format is provided in sections with references.

The most widespread format for storing disk images is D64, so it is the only one implemented here. Disk images in D64 format contains sector data from actual disk. These are the sector data that Commodore 64 receives from the disk drive and they are not GCR encoded. In addition to sector data, there are versions of D64 format that provide additional data for emulation of bad sectors, which improves compatibility with some copyright protection mechanisms. Unfortunately these extensions are not implemented here. D64 format is also covered by the referenced material.

`GCRImage` class is responsible for loading disk images, from D64 format, and creating in-memory GCR encoded image that can be used by the rest of 1541-II emulator.

<img width="394" height="455" alt="image" src="https://github.com/user-attachments/assets/199f6775-8351-4391-907c-8aff8d1c35f0" />

_**GCRImage class**_

VIA chip
--------

VIA is an IO port controller like its successor - CIA chip. It provides two general purpose IO ports as well as two programmable timers. In addition to 8 IO pins each port has two additional control lines that can be used as interrupt inputs or as handshake outputs.

Drive head and motor as well as the IEC bus lines are attached to the ports of two VIA chips that are installed in the drive.

`VIA` class implements emulation of the chip. It extends `MemmoryMappedDevice` since VIAs are mapped into memory space of the drive's CPU. `Read` and `Write` method reads and updates registers of the chip.

This class also implements `ClockOp` interface which is responsible for updating timers' counters as well as ports' control lines.

`PortA` and `PortB` properties of the class expose two IO ports of VIA chip. `CA1`, `CA2`, `CB1` and `CB2` properties provide access to control line of the IO ports.

<img width="597" height="832" alt="image" src="https://github.com/user-attachments/assets/56314201-c3c0-43eb-8282-6af851b245f9" />

_**VIA and Drive head emulation class diagram**_

Drive head
----------

Drive head is electro-mechanical subsystem that controls drive's motors, position of the head and the head itself that reads/writes data from/to the disk. It is attached to IO ports VIA#2, where port B is responsible for head control and port A is responsible for transferring data.

Head is also attached to control lines of VIA chip as well as SO pin of CPU, which sets overflow flag, to notify it that the data are ready for reading or that they have been written to disk.

`DriveHead` class as its name suggests implements emulation of drive's head. This class implements `ClockOp` interface, so in each clock cycle logic for reading or writing data will be executed. If the head is active, which is controlled by the VIA chip, after certain number of cycles, data will become available for reading operation or stored to disk for write operation. To increase capacity of disks, tracks have different density, depending how far away they are from the center of the disk. So the number of cycles after which the pending read/write operation will be completed depends on the current disk track on which the head is positioned.

Persisting Emulator State
-------------------------

Each emulated device has certain state, these states should be persisted in a file if we want to be able to save or load emulation session which is very useful feature of an emulator and makes old games much easier.

Emulator will save state only at the end of a frame during which the save command is issued. The reason for this is simplification of code that is responsible for handling the state of VIC-II chip.

Saving and loading state of the emulator is done through `IDeviceState` interface. Class for each emulated component that needs to save its state should implement this interface. `ReadDeviceState` and `WriteDeviceState` methods of the interface are called when state needs be persisted or restored. Emulated component can read or write state file using reference to file object that is provided as a parameter of these two methods.

If it is composite and contains other components that implement `IDeviceState` interface, parent component should call appropriate methods on all of its subcomponents. For instance, C64 board will call store/load methods on components that represent memory, CPU and all other chips.

<img width="531" height="411" alt="image" src="https://github.com/user-attachments/assets/ff53be7f-3dfa-4f16-b96a-32a80c81b435" />

_**IDeviceState interface**_

<img width="537" height="257" alt="image" src="https://github.com/user-attachments/assets/26dd31bf-26af-4c64-9f78-8bf8fd882565" />

_**Structure of Emulator State**_

Environment
-----------

Environment project contains interfaces whose purpose is to isolate external system from the emulator. Currently there are two interfaces defined: `IVideoOutput` interface which abstracts video output and `IFile` which abstracts file operations.

ROM Files
---------

ROM files are not included in source code, due to legal reasons, but they should be available on the Internet:

Files that are required by the emulator are:

*   `kernal.rom` - content of ROM chip hosting OS
*   `basic.rom` - content of ROM chip hosting Basic interpreter
*   `chargen.rom` - Charachter ROM
*   `d1541.rom` - firmware of CBM 1541-II drive

They should be copied to `.\c64_roms` folder of the project. If you are just try to run the emulator binaries, copy ROM files into the same folder with the executables.

References
----------

This section provides links official documentation and datasheets for hardware. It also has links to very useful resources that are product of reverse engineering of Commodore 64's hardware and software, such as memory maps, illegal opcodes, ROM disassemblies...

### Hardware datasheets and documentation

*   MOS 6510 Datasheet - CPU \[[PDF](http://www.6502.org/documents/datasheets/mos/mos_6510_mpu.pdf)\]
*   MOS 6502 Illegal Opcodes - CPU \[[TXT](http://www.ffd2.com/fridge/docs/6502-NMOS.extra.opcodes
    )\]
*   MOS 6526 Datasheet - CIA \[[PDF](http://archive.6502.org/datasheets/mos_6526_cia.pdf)\]
*   MOS 6522 Datasheet - VIA \[[PDF](http://archive.6502.org/datasheets/mos_6522.pdf)\]
*   MOS 6581 Datasheet - SID \[[PDF](http://archive.6502.org/datasheets/mos_6581_sid.pdf
    )\]
*   MOS 6567 Description - (VIC-II) \[[TXT](http://www.cebix.net/VIC-Article.txt)\]

### Memory Maps

*   C64 Memory Map \[[HTML](http://sta.c64.org/cbm64mem.html)\]
*   CMB 1541-II Drive Memory Map \[[HTML](http://sta.c64.org/cbm1541mem.html)\]

### ROM Dissasemblies

*   BASIC and KERNAL ROMs Disassemblies \[[HTML](http://www.ffd2.com/fridge/docs/c64-diss.html)\]
*   CBM 1541-II Firmware Disassembly (DOS 2.6 ROM) \[[HTML](http://www.ffd2.com/fridge/docs/1541dis.html)\]

### Hardware Schematics

*   C64 Schematics \[[GIF](http://www.zimmers.net/anonftp/pub/cbm/schematics/computers/c64/250469-rev.A-left.gif
    ), [GIF](http://www.zimmers.net/anonftp/pub/cbm/schematics/computers/c64/250469-rev.A-right.gif)\]
*   CMB 1541-II Drive Schematics \[[PNG](http://www.zimmers.net/anonftp/pub/cbm/schematics/drives/new/1541/1541-II.340503.gif)\]

### Others

*   D64 file format \[[TXT](http://ist.uwaterloo.ca/~schepers/formats/D64.TXT)\]
*   IEC bus \[[PDF](http://c64emulator.111mb.de/c64/docu/IEC_1541_info.pdf)\]
