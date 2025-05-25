---
layout: post
title: Unveiling PolkaVM's Secrets:How Programs Really Work ⚙️
subtitle: A Deep Dive into the ProgramBlob:Code, Data, and Everything In Between 📦
categories: Technics
tags: PolkaVM ProgramRepresentation ProgramBlob
---

This article describes how executable programs are represented in PolkaVM, covering the structure, encoding, and transformation of programs. We focuse on the internal representation of compiled code, not the process of compilation itself here. For information about the compilation pipeline, see [Compilation Pipeline]().

## Overview

PolkaVM represents programs using a custom binary format called `ProgramBlob`. This format encapsulates all necessary components for program execution: code, data, jump tables, and symbols. Programs originate from either ELF binaries (compiled from languages like Rust) or from assembly source.

```mermaid!

flowchart TD
    subgraph Program Sources 
    A[ELF Binary]
    B[Assembly Source]
    end
    subgraph Transformation
    C[PolkaVM Linker]
    D[Assembler]
    end
    subgraph Program Representation
    E[ProgramBlob]
    F[Code Section]
    G[Read-only Data]
    H[Read-write Data]
    I[Jump Table]
    J[Instruction Bitmask]
    K[Import/Export Symbols]
    end

    A-->C
    B-->D
    C-->E
    D-->E
    E-->F
    E-->G
    E-->H
    E-->I
    E-->J
    E-->K

```



Sources:

* [crates/polkavm-linker/src/program\_from\_elf.rs](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-linker/src/program_from_elf.rs)
* [crates/polkavm-common/src/program.rs](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/program.rs)
* [crates/polkavm-common/src/writer.rs](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/writer.rs)

## ProgramBlob Format

The `ProgramBlob` is the core representation of a program in PolkaVM. It's a binary format that contains all the information needed to execute a program.

### Key Components

| Component             | Description                                                 |
| --------------------- | ----------------------------------------------------------- |
| Magic+Version Header  | Identifies the blob as a PolkaVM program and its version    |
| Memory Config         | Defines memory layout parameters (page size, section sizes) |
| Code Section          | Contains the executable instructions                        |
| Read-only Data        | Constant data that doesn't change during execution          |
| Read-write Data       | Data that can be modified during program execution          |
| Jump Table            | Maps basic block indices to instruction offsets             |
| Bitmask               | Bitfield that marks instruction boundaries                  |
| Import/Export Symbols | Symbols for interfacing with external code                  |

### Program Header Structure

The blob starts with a header that contains:

* Magic identifier "PVM\0" (4 bytes)
* Size of the entire blob (variable-length encoding)
* Memory configuration data

### Building ProgramBlobs

The `ProgramBlobBuilder` class handles the construction of program blobs, managing the serialization of all components into the final binary format.

```mermaid!

classDiagram
    class ProgramBlobBuilder{
      -is_64: bool
      -ro_data_size: u32
      -rw_data_size: u32
      -stack_size: u32-ro_data: Vec~u8~
      -rw_data: Vec~u8~
      -imports: Vec~ProgramSymbol~
      -code: Vec~Instruction~
      -jump_table: Vec~u32~
      +new()
      +new_64bit()
      +set_ro_data_size(size: u32)
      +set_rw_data_size(size: u32)
      +set_stack_size(size: u32)
      +set_code(code: &[Instruction], jump_table: &[u32])
      +build()-> ProgramBlob
    }
    class ProgramBlob{
      +code()-> &[u8]
      +ro_data()-> &[u8]
      +rw_data()-> &[u8]
      +bitmask()-> &[u8]
      +jump_table()-> &[u8]
      +jump_table_entry_count()-> u32
      +memory_map()-> &MemoryMap
    }
    
    ProgramBlobBuilder ..>ProgramBlob:builds

```



Sources:

* [crates/polkavm-common/src/writer.rs84-168](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/writer.rs#L84-L168)
* [crates/polkavm-common/src/writer.rs157-340](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/writer.rs#L157-L340)

## Code Representation

### Instruction Encoding

PolkaVM uses a variable-length instruction encoding scheme. Each instruction starts with an opcode byte, followed by variable-length operands. This compact representation balances code density with execution efficiency.

Key characteristics:

* Variable-length instructions (1 to 17 bytes)
* First byte is always the opcode
* Operands use variable-length encoding for integers
* Register operands are encoded efficiently
* Relative jumps use offset-based encoding

### Instruction Bitmask

To efficiently identify instruction boundaries without full decoding, PolkaVM uses a bitmask:

* A bit is set (1) for the first byte of each instruction
* Continuation bytes are marked with 0
* Allows quick jumping to instruction boundaries

```
Code:    [Instr1][Instr2  ][Instr3][Instr4    ]
Bitmask: 1      1         1      1            
         ^      ^         ^      ^
         |      |         |      |
         Start  Start     Start  Start

```

### Jump Table

The jump table maps basic block indices to their positions in the code section:

* Enables efficient control flow transfers (jumps, branches)
* Abstracts away the actual instruction offsets
* Contains entries of variable size depending on offset magnitude

```mermaid!

flowchart LR
    A[Jump Table]--> B[Entry 0: Offset to Block 0]
    B ..-> C[Block 0 Instructions]
    A --> D[Entry 1: Offset to Block 1]
    D ..-> E[Block 1 Instructions]
    A --> F[Entry 2: Offset to Block 2]
    F ..-> G[Block 2 Instructions]
    H[Code Section] --> C
    H[Code Section] --> E
    H[Code Section] --> G

```



Sources:

* [crates/polkavm-common/src/program.rs187-349](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/program.rs#L187-L349)
* [crates/polkavm-common/src/program.rs351-445](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/program.rs#L351-L445)
* [crates/polkavm-common/src/writer.rs293-320](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/writer.rs#L293-L320)

## Register Representation

PolkaVM uses a fixed set of registers that map to RISC-V registers. These are encoded efficiently in instructions.

### Register Set

| PolkaVM Register | RISC-V ABI Name | Description               |
| ---------------- | --------------- | ------------------------- |
| `RA`             | ra              | Return address            |
| `SP`             | sp              | Stack pointer             |
| `T0`-`T2`        | t0-t2           | Temporary registers       |
| `S0`-`S1`        | s0-s1           | Saved registers           |
| `A0`-`A5`        | a0-a5           | Argument/return registers |

```mermaid!

classDiagram
    class RawReg{
      -value: u32
      +get() -> Reg
      +raw_unparsed() -> u32
    }
    class Reg{
        <<enumeration>>
      RA
      SP
      T0
      T1
      T2
      S0
      S1
      A0
      A1
      A2
      A3
      A4
      A5
      +to_usize()->usize
      +to_u32()->u32
      +name()->str
    }
    RawReg --> Reg:represents

```



Sources:

* [crates/polkavm-common/src/program.rs8-169](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/program.rs#L8-L169)
* [crates/polkavm-linker/src/program\_from\_elf.rs22-152](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-linker/src/program_from_elf.rs#L22-L152)

## Memory Layout

The memory layout for PolkaVM programs follows a specific structure to ensure proper isolation and efficient access.

```mermaid!

flowchart TD

    A["VM_ADDRESS_SPACE_BOTTOM (0x10000)"] --> B[Read-only Data Section]
    B --> C[Read-write Data Section]
    C --> D["Heap (grows upward)"]
    
    E["Auxiliary Data"]
    E -->F["Stack (grows downward)"]
    F-->G[VM_ADDRESS_SPACE_TOP]

```



### Memory Sections

| Section         | Description                      | Access                 |
| --------------- | -------------------------------- | ---------------------- |
| Read-only Data  | Constants, string literals, etc. | Read-only              |
| Read-write Data | Global variables, mutable data   | Read-write             |
| Stack           | Local variables, call frames     | Read-write             |
| Heap            | Dynamically allocated memory     | Read-write             |
| Auxiliary Data  | Implementation-specific data     | Implementation-defined |

### Memory Map Builder

The `MemoryMapBuilder` class handles the calculation of memory layout based on program requirements:

* Ensures proper alignment of sections
* Handles page size requirements
* Calculates maximum heap size based on available address space

```mermaid!

classDiagram
    class MemoryMapBuilder{
      -page_size: u32
      -ro_data_size: u32
      -rw_data_size: u32
      -stack_size: u32
      -aux_data_size: u32
      +new(page_size: u32)
      +ro_data_size(value: u32)
      +rw_data_size(value: u32)
      +stack_size(value: u32)
      +aux_data_size(value: u32)
      +build()-> MemoryMap
    }
    class MemoryMap{
      -page_size: u32
      -ro_data_size: u32
      -rw_data_address: u32
      -rw_data_size: u32
      -stack_address_high: u32
      -stack_size: u32
      -aux_data_address: u32
      -aux_data_size: u32
      -heap_base: u32
      -max_heap_size: u32
      +ro_data_address()-> u32
      +ro_data_range()-> Range
      +rw_data_address()-> u32
      +rw_data_range()-> Range
      +stack_address_low()-> u32
      +stack_range()-> Range
      +heap_base()-> u32
    }
   MemoryMapBuilder ..> MemoryMap:builds

```



Sources:

* [crates/polkavm-common/src/abi.rs1-333](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/abi.rs#L1-L333)
* [crates/polkavm-common/src/abi.rs41-172](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/abi.rs#L41-L172)
* [crates/polkavm-common/src/abi.rs176-283](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/abi.rs#L176-L283)

## Program Transformation Process

The process of transforming source programs (ELF binaries or assembly) into the PolkaVM representation involves several steps.

### ELF to ProgramBlob Conversion

```mermaid!

flowchart TD
    A[ELF Binary] -->B[ELF Parser]
    B --> C[Section Extractor]
    C --> D[Code Sections]
    C --> E[Data Sections]
    C --> F[Debug Information]
    D --> G[RISC-V Instruction Decoder]
    G --> H[Intermediate Representation]
    H --> I[PolkaVM Code Generator]
    E --> J[Data Processor]
    J --> K[ProgramBlobBuilder]
    F --> L[Symbol Processor]
    I --> K
    L --> K
    K --> M[ProgramBlob]

```



1. Parse the ELF file to extract sections and symbols
2. Process code sections by converting RISC-V instructions to PolkaVM instructions
3. Handle relocations and symbol references
4. Place data sections in appropriate locations
5. Generate the jump table and bitmask
6. Construct the final ProgramBlob

### Assembly to ProgramBlob Conversion

Assembly is parsed directly into PolkaVM instructions, which are then assembled into a ProgramBlob. This provides a more direct route for creating PolkaVM programs.

Sources:

* [crates/polkavm-linker/src/program\_from\_elf.rs](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-linker/src/program_from_elf.rs)
* [crates/polkavm-linker/src/elf.rs](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-linker/src/elf.rs)
* [crates/polkavm-common/src/writer.rs](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/writer.rs)

## Instruction Visitor Pattern

PolkaVM uses the Visitor pattern to process instructions. The `OpcodeVisitor` trait defines an interface for traversing and operating on instructions:

```mermaid!

classDiagram
    class OpcodeVisitor{
        <<trait>>
      +State: Type
      +ReturnTy: Type
      +InstructionSet: InstructionSet
      +instruction_set()-> InstructionSet
      +dispatch(state: &mut State, opcode: usize, chunk: u128, offset: u32, skip: u32)-> ReturnTy
    }
    class InstructionSet{
        <<trait>>
      +opcode_from_u8(byte: u8)-> Option<Opcode>
    }
    class Opcode{
        <<enumeration>>
      +from_u8_any(byte: u8)-> Option<Opcode>
      +can_fallthrough()-> bool
    }
    OpcodeVisitor --> InstructionSet:uses
    InstructionSet --> Opcode:creates

```

This pattern enables:

* Efficient instruction traversal
* Different operations (execution, disassembly, analysis) sharing the same traversal mechanism
* Separation of instruction decoding from operation

Sources:

* [crates/polkavm-common/src/program.rs742-840](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/program.rs#L742-L840)
* [crates/polkavm-common/src/program.rs326-349](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/program.rs#L326-L349)

## Symbol Handling

PolkaVM programs can import and export symbols for interaction with host functions and to expose functionality.

### Import Symbols

Imports represent external functions that the program can call:

* Each import has a unique name
* Host environments provide implementations for these functions
* Calls to imports are handled via special `ecalli` instructions

### Export Symbols

Exports are functions within the program that can be called from outside:

* Each export has a name and a target (basic block index)
* The main entry point of a program is typically an export
* External code can invoke exports by name

```mermaid!

flowchart LR
    subgraph PolkaVM Program
        A[Import Table]
        B[Export Table]
        C[Code]
    
    end

    subgraph Host Environment
        D[host_function_1]
        E[host_function_2]
    
    end
    A-->D
    A-->E
    E ..->|calls|B
    B-->C

```



Sources:

* [crates/polkavm-common/src/writer.rs138-149](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/writer.rs#L138-L149)
* [crates/polkavm-linker/src/program\_from\_elf.rs](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-linker/src/program_from_elf.rs)

## Variable-Length Integer Encoding

PolkaVM uses variable-length encoding for integer values to save space in instructions. Two main encoding schemes are used:

### Varint Encoding

Used for general-purpose variable-length integers:

* Small values use fewer bytes
* First byte contains size information and part of the value
* Can encode 32-bit values in 1-5 bytes

### Simple Varint Encoding

Used for instruction operands:

* Optimized for the common case of small values
* Number of bytes determined by the instruction's "skip" value
* Efficiently handles sign extension

```mermaid!

flowchart TD
    A[32-bit value] -->B[Varint Encoder]
    B --> C["Variable-length encoded value (1-5 bytes)"]
    C -->D[Varint Decoder]
    D -->E[Original 32-bit value]

```



Sources:

* [crates/polkavm-common/src/varint.rs1-215](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/varint.rs#L1-L215)
* [crates/polkavm-common/src/program.rs446-521](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm-common/src/program.rs#L446-L521)

## Conclusion

The program representation in PolkaVM is designed to efficiently store and execute programs. The `ProgramBlob` format encapsulates all necessary components (code, data, jump table, etc.) in a compact binary representation. The variable-length instruction encoding and efficient memory layout ensure that programs can be executed with minimal overhead.

This representation is a core part of the PolkaVM system, bridging the gap between source programs (ELF binaries or assembly) and the execution engine. Understanding this representation is essential for working with the PolkaVM ecosystem.
