---
layout: post
title: CoreVM Engine
subtitle: CoreVM Engine
categories: Technics
tags: CoreVM Engine JAM PolkaVM
---

# CoreVM Engine

The Core VM Engine is the central execution system of PolkaVM, responsible for loading, compiling, and running guest programs in a secure and efficient manner. It provides the foundation for program execution, memory management, sandboxing, and interaction between host and guest code. For information about program representation, see [Program Representation](https://deepwiki.com/paritytech/polkavm/4-program-representation), and for the sandboxing system, see [Sandboxing](https://deepwiki.com/paritytech/polkavm/3-sandboxing).

## Architecture Overview

The Core VM Engine consists of several key components that work together to enable secure and efficient execution of programs.

Core VM Engine ArchitectureEngineModuleConfigurationRaw InstanceBackendInterpreter BackendCompiler BackendArchitecture-specific code generationSandbox implementationLinux SandboxGeneric Sandbox



Sources: [crates/polkavm/src/api.rs96-234](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L96-L234) [crates/polkavm/src/api.rs962-2044](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L962-L2044) [crates/polkavm/src/lib.rs1-172](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/lib.rs#L1-L172)

## Key Components

### Engine

The `Engine` is the main entry point for using PolkaVM. It holds global state and configuration, and is responsible for creating new modules and managing sandbox workers.

Engine+selected\_backend: BackendKind+selected\_sandbox: Option\<SandboxKind>+interpreter\_enabled: bool+crosscheck: bool+state: Arc\<EngineState>+allow\_dynamic\_paging: bool+allow\_experimental: bool+new(config: \&Config) : -> Result\<Engine, Error>+backend() : -> BackendKind+idle\_worker\_pids() : -> Vec\<u32>«enumeration»BackendKindCompilerInterpreter«enumeration»SandboxKindLinuxGenericEngineState+sandboxing\_enabled: bool+sandbox\_global: Option\<GlobalStateKind>+sandbox\_cache: Option\<WorkerCacheKind>+compiler\_cache: CompilerCache+module\_cache: ModuleCache



Sources: [crates/polkavm/src/api.rs96-234](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L96-L234) [crates/polkavm/src/config.rs7-49](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L7-L49) [crates/polkavm/src/config.rs51-85](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L51-L85)

### Module

A `Module` represents a compiled program that can be instantiated for execution. It contains the program code, memory layout, and metadata about exports and imports.

Module+blob() : : \&ProgramBlob+memory\_map() : : \&MemoryMap+default\_sp() : : RegValue+exports() : : Iterator\<ProgramExport>+imports() : : Imports+is\_64\_bit() : : bool+new(engine: \&Engine, config: \&ModuleConfig, bytes: ArcBytes) : -> Result\<Module, Error>+from\_blob(engine: \&Engine, config: \&ModuleConfig, blob: ProgramBlob) : -> Result\<Module, Error>+instantiate() : -> Result\<RawInstance, Error>ModulePrivate+engine\_state: Option\<Arc\<EngineState>>+crosscheck: bool+blob: ProgramBlob+compiled\_module: CompiledModuleKind+interpreted\_module: Option\<InterpretedModule>+memory\_map: MemoryMap+gas\_metering: Option\<GasMeteringKind>+is\_strict: bool+step\_tracing: bool+dynamic\_paging: bool+page\_size\_mask: u32+page\_shift: u32+instruction\_set: RuntimeInstructionSet+cost\_model: CostModelRef



Sources: [crates/polkavm/src/api.rs258-780](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L258-L780) [crates/polkavm/src/api.rs386-655](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L386-L655)

### RawInstance

A `RawInstance` is an instantiated module that can be executed. It holds the execution state including registers, memory, and a pointer to the current instruction.

RawInstance+module: Module+backend: InstanceBackend+crosscheck\_instance: Option\<Box\<InterpretedInstance>>+module() : -> \&Module+is\_64\_bit() : -> bool+run() : -> Result\<InterruptKind, Error>+reg(reg: Reg) : -> RegValue+set\_reg(reg: Reg, value: RegValue)+read\_u8(address: u32) : -> Result\<u8, MemoryAccessError>+read\_u16(address: u32) : -> Result\<u16, MemoryAccessError>+read\_u32(address: u32) : -> Result\<u32, MemoryAccessError>+read\_u64(address: u32) : -> Result\<u64, MemoryAccessError>+write\_u8(address: u32, value: u8) -> ResultUnsupported markdown: del+write\_u16(address: u32, value: u16) -> ResultUnsupported markdown: del+write\_u32(address: u32, value: u32) -> ResultUnsupported markdown: del+write\_u64(address: u32, value: u64) -> ResultUnsupported markdown: del«enumeration»InstanceBackendCompiledLinux(SandboxInstance\<SandboxLinux>)CompiledGeneric(SandboxInstance\<SandboxGeneric>)Interpreted(InterpretedInstance)



Sources: [crates/polkavm/src/api.rs962-2044](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L962-L2044) [crates/polkavm/src/api.rs859-873](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L859-L873)

## Execution Flow

The execution flow of the Core VM Engine encompasses program loading, instantiation, and the execution cycle.

"Execution Backend""RawInstance""Module""Engine""Host Program""Execution Backend""RawInstance""Module""Engine""Host Program"alt\[Normal Execution]\[Host Call]\[Error Condition]create(config)new module(blob)from\_blob(engine, config, blob)ModuleModuleinstantiate()new(module)initializeRawInstanceset\_reg(...)run()run()InterruptKind::FinishedInterruptKind::FinishedInterruptKind::Ecalli(n)InterruptKind::Ecalli(n)set\_reg(...)run()InterruptKind::Trap/NotEnoughGas/SegfaultInterruptKind::Trap/NotEnoughGas/Segfault



Sources: [crates/polkavm/src/api.rs386-655](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L386-L655) [crates/polkavm/src/api.rs962-2044](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L962-L2044) [crates/polkavm/src/utils.rs103-139](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/utils.rs#L103-L139)

### Instantiation Process

When a module is instantiated:

1. The engine selects the appropriate backend (Compiler or Interpreter)
2. If the Compiler backend is selected, the VM instructions are compiled to native code
3. If sandboxing is enabled, a sandbox environment is prepared
4. A RawInstance is created, linked to the appropriate backend
5. Memory is allocated according to the module's memory map

### Execution Cycle

The execution cycle involves:

1. Setting up the initial state (registers, program counter)
2. Running the code via the selected backend
3. Handling interrupts that occur during execution
4. Returning control to the host when necessary (for host calls or on completion)

### Interrupt Types

The VM uses interrupts to handle various execution events:

| Interrupt Type | Description             | Trigger                                                  |
| -------------- | ----------------------- | -------------------------------------------------------- |
| Finished       | Normal termination      | Program jumps to `VM_ADDR_RETURN_TO_HOST`                |
| Trap           | Abnormal termination    | Invalid instruction, jump, or explicit trap              |
| Ecalli         | Host function call      | `ecalli` instruction executed                            |
| NotEnoughGas   | Gas limit exceeded      | Gas counter reaches zero                                 |
| Segfault       | Memory access violation | Accessing unmapped memory or writing to read-only memory |
| Step           | Single step completed   | Step tracing is enabled                                  |

Sources: [crates/polkavm/src/utils.rs103-139](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/utils.rs#L103-L139) [crates/polkavm/src/tests.rs365-375](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/tests.rs#L365-L375)

## Backends

The Core VM Engine supports two execution backends:

### Interpreter Backend

The Interpreter backend executes instructions one by one, interpreting each instruction as it runs. It's implemented in `InterpretedInstance`:

InterpretedInstance+module: Module+basic\_memory: BasicMemory+dynamic\_memory: DynamicMemory+program\_counter: ProgramCounter+program\_counter\_valid: bool+next\_program\_counter: Option\<ProgramCounter>+next\_program\_counter\_changed: bool+cycle\_counter: u64+gas: i64+regs: \[u64; Reg::ALL.len() : ]+new\_from\_module(module: Module, force\_step\_tracing: bool) : -> Self+run() : -> Result\<InterruptKind, Error>+reg(reg: Reg) : -> RegValue+set\_reg(reg: Reg, value: RegValue)



Sources: [crates/polkavm/src/interpreter.rs397-752](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/interpreter.rs#L397-L752) [crates/polkavm/src/interpreter.rs1-20](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/interpreter.rs#L1-L20)

### Compiler Backend

The Compiler backend translates the VM instructions to native machine code, which is executed directly by the CPU. This provides better performance but is more complex and requires platform-specific optimizations:

Compiler Backend FlowProgramBlobCompiler VisitorGas VisitorArchitecture VisitorGas Metering StubsNative Machine CodeCompiled ModuleSandbox Execution



Sources: [crates/polkavm/src/api.rs237-256](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L237-L256) [crates/polkavm/src/api.rs496-575](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L496-L575)

## Memory Management

The VM has a well-defined memory layout to ensure security and proper isolation:

Memory LayoutVM Memory SpaceGuard PagesCode SectionRead-Only Data SectionRead-Write Data SectionStack RegionAuxiliary Data RegionHeap (via sbrk)



The memory layout is configured by the `MemoryMap` which defines the address ranges for each section:

| Section         | Purpose                            | Access                 |
| --------------- | ---------------------------------- | ---------------------- |
| Code Section    | Contains executable instructions   | Read-only              |
| Read-Only Data  | Constants and immutable data       | Read-only              |
| Read-Write Data | Global variables and mutable data  | Read-write             |
| Stack           | Function calls and local variables | Read-write             |
| Auxiliary Data  | Host-configurable data region      | Read-write (from host) |
| Heap            | Dynamically allocated memory       | Read-write             |

Sources: [crates/polkavm/src/api.rs594-623](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L594-L623) [crates/polkavm/src/interpreter.rs79-133](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/interpreter.rs#L79-L133)

### Dynamic Paging

When enabled, dynamic paging allows memory to be mapped on-demand:

1. Pages are only allocated when accessed
2. Attempting to access an unmapped page triggers a `Segfault` interrupt
3. The host can map new pages or protect pages as needed
4. Useful for memory-efficient execution of large programs

Sources: [crates/polkavm/src/interpreter.rs250-282](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/interpreter.rs#L250-L282) [crates/polkavm/src/config.rs458-471](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L458-L471)

## Configuration

### Engine Configuration

The `Config` struct controls engine-level settings:

| Setting                | Description                              | Default                          |
| ---------------------- | ---------------------------------------- | -------------------------------- |
| `backend`              | Execution backend (Compiler/Interpreter) | Best available                   |
| `sandbox`              | Sandbox implementation (Linux/Generic)   | Best available                   |
| `crosscheck`           | Enable execution cross-checking          | false                            |
| `allow_experimental`   | Enable experimental features             | false                            |
| `allow_dynamic_paging` | Enable dynamic paging                    | false                            |
| `worker_count`         | Number of worker sandboxes               | 2                                |
| `cache_enabled`        | Enable module caching                    | true (with module-cache feature) |
| `sandboxing_enabled`   | Enable security sandboxing               | true                             |

Sources: [crates/polkavm/src/config.rs100-217](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L100-L217) [crates/polkavm/src/config.rs152-366](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L152-L366)

### Module Configuration

The `ModuleConfig` struct controls module-level settings:

| Setting          | Description                               | Default          |
| ---------------- | ----------------------------------------- | ---------------- |
| `page_size`      | Memory page size                          | 4096 (4K)        |
| `gas_metering`   | Gas metering type (None/Sync/Async)       | None             |
| `is_strict`      | Enable strict mode for validation         | false            |
| `step_tracing`   | Enable instruction-by-instruction tracing | false            |
| `dynamic_paging` | Enable dynamic paging for this module     | false            |
| `aux_data_size`  | Size of auxiliary data region             | 0                |
| `allow_sbrk`     | Allow dynamic heap allocation             | true             |
| `cost_model`     | Gas cost model                            | Naive cost model |

Sources: [crates/polkavm/src/config.rs392-583](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L392-L583) [crates/polkavm/src/config.rs413-427](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L413-L427)

## Gas Metering

Gas metering limits the computational resources a program can use:

Gas MeteringModule ConfigurationGas Metering KindSynchronousAsynchronousCheck after every instructionPeriodic checkingInterrupt: NotEnoughGas



### Gas Metering Types

* **Synchronous**: Checks gas after every instruction. Precise but higher overhead.
* **Asynchronous**: Checks gas periodically. Lower overhead but less precise timing of interruption.

Sources: [crates/polkavm/src/config.rs369-385](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L369-L385) [crates/polkavm/src/api.rs361-365](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L361-L365)

## Security Considerations

### Sandboxing

The Core VM Engine provides security through sandboxing:

* **Linux Sandbox**: Uses Linux-specific security features (seccomp, namespaces)
* **Generic Sandbox**: Uses signal handlers for basic protection (less secure, considered experimental)

### Memory Protection

Memory protection ensures that:

* Code and read-only data cannot be modified
* Out-of-bounds memory access is prevented
* Stack overflows are caught
* Guard pages isolate different memory regions

Sources: [crates/polkavm/src/api.rs153-188](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L153-L188) [crates/polkavm/src/config.rs357-366](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/config.rs#L357-L366)

## Error Handling

The VM uses a comprehensive error handling system:

1. The `Error` type encapsulates various error kinds
2. The `InterruptKind` enum represents different types of execution interrupts
3. The `MemoryAccessError` handles memory access failures

Sources: [crates/polkavm/src/error.rs19-108](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/error.rs#L19-L108) [crates/polkavm/src/utils.rs103-139](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/utils.rs#L103-L139) [crates/polkavm/src/api.rs876-909](https://github.com/paritytech/polkavm/blob/910adbda/crates/polkavm/src/api.rs#L876-L909)
