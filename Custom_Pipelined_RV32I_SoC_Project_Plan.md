# Custom 5-Stage Pipelined RV32I SoC --- RTL Design and UVM Verification

## 1. Project definition

**Project title:** Design and UVM-Based Functional Verification of a
Custom 5-Stage Pipelined RV32I SoC with AXI4-Lite, APB Peripherals and
Memory Subsystem

  Parameter          Decision
  ------------------ -------------------------------------------
  Team               5 CDAC students
  Schedule           8 weeks, with contingency
  RTL level          Intermediate
  CPU                Designed by the team
  CPU architecture   5-stage, in-order pipeline
  ISA scope          Defined RV32I subset, expandable
  Bus                AXI4-Lite, single master
  Peripheral bus     APB3
  Verification       SystemVerilog, UVM, SVA
  Simulator          QuestaSim / ModelSim
  Design policy      All mandatory DUT RTL written by the team

## 2. Project philosophy

The main learning objective is to design and verify the essential
components rather than integrate a ready-made CPU or copy complete IP
implementations.

The team will write its own: - 5-stage pipelined CPU datapath and
control. - Pipeline registers and hazard-management logic. - ALU,
register file, instruction decoder and immediate generator. -
Instruction and data memory models/controllers. - CPU data-bus interface
and AXI4-Lite master. - AXI4-Lite address decoder/router. -
AXI4-Lite-to-APB bridge and APB decoder. - GPIO, timer and UART
transmitter. - UVM agents, sequences, scoreboards, assertions and
coverage model.

Use public specifications, textbooks and open-source projects as
learning references. Do not import ready-made CPU, bus, bridge or
peripheral RTL into the mandatory design. UVM libraries and simulator
infrastructure are tools, not DUT IP.

**Realism:** A production-quality pipelined RISC-V SoC is beyond a
two-month student project. The target is a small, synthesizable,
educational 5-stage processor and a deliberately limited SoC. The team
must document unsupported instructions and architectural limitations
rather than claim full RV32I compliance.

------------------------------------------------------------------------

## 3. Proposed SoC architecture

### System-level architecture

Use a Harvard-style memory organization: - Instruction fetch accesses a
dedicated instruction memory through a simple request/response
interface. - Data loads/stores use a separate CPU data interface
connected to a custom AXI4-Lite master. - The AXI4-Lite router selects
either data SRAM or the AXI-to-APB bridge.

This avoids forcing instruction fetch and data access to compete for one
AXI master, while keeping the AXI bus subsystem small enough to
implement.

``` text
                 +-------------------------------------+
                 |          CUSTOM RV32I CPU           |
                 |                                     |
                 |  IF -> ID -> EX -> MEM -> WB         |
                 |   |    |    |     |     |           |
                 |  IF/ID ID/EX EX/MEM MEM/WB           |
                 |                                     |
                 |  Hazard Detection + Forwarding      |
                 |  Stall / Flush / Branch Control     |
                 +-------------+-----------------------+
                               |
                 +-------------+------------------+
                 |                                |
          Instruction Interface             Data Interface
                 |                                |
          +------v-------+                +-------v--------+
          | Instruction  |                | AXI4-Lite      |
          | SRAM         |                | Master         |
          +--------------+                +-------+--------+
                                                  |
                                         +--------v---------+
                                         | AXI4-Lite Router |
                                         | Address Decoder  |
                                         +------+-----------+
                                                |
                                  +-------------+-------------+
                                  |                           |
                           +------v------+             +------v------+
                           | Data SRAM   |             | AXI-to-APB  |
                           | Controller  |             | Bridge      |
                           +-------------+             +------+------+
                                                             |
                                                       +-----v------+
                                                       | APB Decoder|
                                                       +--+----+----+
                                                          |    |
                                                +---------+    +---------+
                                                |          |              |
                                              GPIO       Timer          UART TX
```

### Why this organization?

-   The five-stage pipeline gives meaningful CPU microarchitecture
    experience.
-   Separate instruction and data paths simplify the memory interface
    and avoid a multi-master crossbar.
-   The data-side AXI4-Lite bus exposes the team to practical bus
    handshakes and backpressure.
-   APB provides a manageable peripheral subsystem.
-   The team owns the complete RTL path from instruction execution to
    peripheral access.

### Baseline scope

**CPU** - Five-stage, single-issue, in-order pipeline. - 32-bit datapath
and 32-bit registers. - 32 architectural integer registers; x0 hardwired
to zero. - No branch prediction. - Static not-taken branch policy is
sufficient for the baseline. - Pipeline forwarding, load-use stall and
branch flush.

**SoC** - Instruction SRAM. - Data SRAM. - One AXI4-Lite master and two
targets. - AXI4-Lite-to-APB bridge. - GPIO and timer. - UART
transmitter.

**Verification** - Unit-level directed tests. - UVM AXI4-Lite and APB
agents. - Scoreboards and reference models. - SVA protocol and pipeline
assertions. - Firmware/program-driven CPU and SoC tests. - Functional
coverage and regression.

------------------------------------------------------------------------

## 4. CPU architecture: five pipeline stages

### Pipeline overview

  -----------------------------------------------------------------------
  Stage                   Main operations         Main components
  ----------------------- ----------------------- -----------------------
  IF --- Instruction      Fetch instruction and   PC, instruction memory
  Fetch                   update PC               interface

  ID --- Instruction      Decode instruction,     Decoder, register file,
  Decode                  read registers,         immediate generator
                          generate immediate      

  EX --- Execute          ALU operation, address  ALU, branch comparator,
                          calculation, branch     forwarding muxes
                          comparison              

  MEM --- Memory Access   Load/store transaction  Load/store unit,
                                                  data-bus interface

  WB --- Write Back       Select result and       Write-back mux,
                          update register file    register write control
  -----------------------------------------------------------------------

### Pipeline registers

Implement and verify these pipeline registers:

-   `if_id_reg`
-   `id_ex_reg`
-   `ex_mem_reg`
-   `mem_wb_reg`

Each register must carry the control and data fields needed by later
stages. Define the exact fields in a pipeline-register specification
before coding.

### Required hazard handling

**1. Data forwarding**

Implement forwarding from: - EX/MEM to EX operands. - MEM/WB to EX
operands.

This avoids unnecessary stalls for common ALU dependencies.

**2. Load-use hazard**

When an instruction in EX is a load and the immediately following
instruction requires its result, insert a bubble and hold the relevant
earlier pipeline state until the value is available.

**3. Control hazard**

For the baseline, resolve branches in EX and use a static not-taken
policy: - If the branch is not taken, continue sequential fetch. - If
taken, redirect the PC and flush younger wrong-path instructions. -
Ensure a flushed instruction cannot update architectural state or issue
a data transaction.

**4. Pipeline stalls**

The data-memory interface may take multiple cycles. While a load/store
is waiting: - Hold the appropriate pipeline state. - Prevent duplicate
memory requests. - Preserve transaction address, data and control. -
Resume only when the response is accepted.

### CPU instruction scope

Implement in this order:

**Phase 1 --- Arithmetic and logic** - ADD, SUB - AND, OR, XOR - ADDI -
ANDI, ORI, XORI - SLT, SLTI

**Phase 2 --- Memory** - LW - SW

**Phase 3 --- Control flow** - BEQ - BNE - JAL

**Phase 4 --- Stretch, only after baseline passes** - SLTU, SLTIU -
Other branch conditions - LUI and AUIPC - LB, LBU, LH, LHU, SB and SH -
JALR

This is a defined RV32I subset, not full RV32I compliance. Full
compliance requires all mandatory instructions and specified
architectural behavior, including alignment and exception handling.

### Suggested CPU RTL modules

  Module                    Responsibility
  ------------------------- ----------------------------------------
  `cpu_top`                 CPU top-level integration
  `program_counter`         PC update and redirect
  `instruction_decoder`     Opcode/funct decoding
  `register_file`           32 × 32-bit integer registers
  `immediate_generator`     I/S/B/U/J immediate generation
  `alu`                     Arithmetic and logic
  `branch_unit`             Branch comparison and target decision
  `forwarding_unit`         Operand forwarding selection
  `hazard_detection_unit`   Load-use hazard detection
  `pipeline_control`        Stall, bubble and flush control
  `if_id_reg`               IF/ID pipeline state
  `id_ex_reg`               ID/EX pipeline state
  `ex_mem_reg`              EX/MEM pipeline state
  `mem_wb_reg`              MEM/WB pipeline state
  `load_store_unit`         Load/store formatting and byte enables
  `instruction_memory`      Instruction storage/interface
  `cpu_data_if`             CPU data request/response interface

------------------------------------------------------------------------

## 5. Memory and bus architecture

### Instruction memory

Use a simple synchronous instruction SRAM with a documented
request/response interface. Define how fetch latency is handled by the
front end. The IF stage must not assume a zero-cycle response unless
that behavior is explicitly implemented and verified.

### Data memory path

The CPU load/store unit connects to a custom CPU data request/response
interface. The AXI4-Lite master adapter converts accepted CPU data
requests into AXI transactions.

Example internal signals:

  Signal              Direction relative to adapter   Meaning
  ------------------- ------------------------------- ---------------------------------
  `req_valid`         Input                           CPU requests a data transaction
  `req_ready`         Output                          Adapter can accept request
  `req_addr[31:0]`    Input                           Byte address
  `req_write`         Input                           1 = write, 0 = read
  `req_wdata[31:0]`   Input                           Write data
  `req_wstrb[3:0]`    Input                           Byte write enables
  `rsp_valid`         Output                          Response is valid
  `rsp_rdata[31:0]`   Output                          Read data
  `rsp_error`         Output                          Transaction error

Define precisely when request fields are sampled, how responses are
acknowledged, and how the pipeline holds state while waiting. Prevent
duplicate requests during stalls.

### AXI4-Lite implementation

Implement all five channels: - AW --- write address. - W --- write
data. - B --- write response. - AR --- read address. - R --- read data.

Required behavior: - Independent AW and W handshakes. - Correct write
response generation. - Read address and read data handshakes. -
VALID/READY backpressure. - Byte strobes. - OKAY and error responses. -
Reset behavior. - One outstanding transaction at a time is acceptable
for the baseline.

Do not implement full AXI4 bursts, IDs or multiple outstanding
transactions in the mandatory scope.

### Router and memory targets

Implement one AXI4-Lite master and two targets: 1. Data SRAM controller.
2. AXI4-Lite-to-APB bridge.

This is a single-master address router/decoder, not a multi-master
crossbar. Do not claim multi-master arbitration.

### Proposed memory map

  Address range               Component             Size
  --------------------------- ------------------ -------
  `0x0000_0000–0x0000_FFFF`   Instruction SRAM     64 KB
  `0x1000_0000–0x1000_FFFF`   Data SRAM            64 KB
  `0x4000_0000–0x4000_0FFF`   GPIO                  4 KB
  `0x4000_1000–0x4000_1FFF`   Timer                 4 KB
  `0x4000_2000–0x4000_2FFF`   UART TX               4 KB

The instruction memory is accessed through its dedicated instruction
interface, not through the data-side AXI router.

Document register offsets, access permissions, reset values, byte-strobe
behavior, invalid address behavior and error responses.

### Memory subsystem RTL

-   `instruction_memory`
-   `data_sram`
-   `sram_controller`
-   `axi_lite_router`
-   `axi_lite_master`
-   `cpu_data_if`

Use synthesizable memory arrays where practical. A simulation memory
model is acceptable for early development, but clearly distinguish it
from the final synthesizable implementation.

------------------------------------------------------------------------

## 6. APB bridge and peripherals

### AXI4-Lite-to-APB bridge

Write the bridge FSM yourselves. It must: - Accept AXI4-Lite read or
write transactions. - Capture address, write data and strobes. -
Generate APB SETUP and ACCESS phases. - Hold APB controls stable while
waiting. - Support APB wait states using `PREADY`. - Return read data
and response to AXI. - Handle invalid peripheral addresses and reset.

### Mandatory peripherals

**GPIO** - Direction register. - Output register. - Input register. -
Read/write behavior and reset values.

**Timer** - 32-bit counter. - Enable control. - Compare/terminal-count
register. - Status flag. - Software clear behavior. - Optional interrupt
output.

**UART transmitter** - TX data register. - Status register. -
Configurable baud divider. - Start bit, 8 data bits and stop bit. - Busy
and completion status.

UART RX is a stretch feature, not mandatory. SPI, DMA and cache are
removed from the baseline.

------------------------------------------------------------------------

## 7. UVM verification strategy

Verification must be developed alongside RTL. Do not wait until the CPU
and SoC are complete.

### Verification levels

**Level 1 --- Unit verification** - ALU operations and boundary
values. - Register file read/write and x0 behavior. - Immediate
generator formats. - Instruction decoder. - Forwarding selection. -
Hazard detection. - Pipeline register stall/flush behavior. -
Instruction/data memory. - AXI adapter and router. - APB bridge. - GPIO,
timer and UART TX.

**Level 2 --- Pipeline verification** - Independent instruction
sequences. - Back-to-back dependent ALU instructions. - EX/MEM and
MEM/WB forwarding cases. - Load-use hazard. - Taken and not-taken
branches. - Branch immediately after a dependent instruction. -
Wrong-path instruction flush. - Stalls during data-memory access. - No
duplicate load/store request during a stall.

**Level 3 --- Bus and peripheral verification** - AXI reads and
writes. - Independent AW/W timing. - VALID/READY backpressure. - Byte
strobes. - Error responses. - APB wait states. - Peripheral register
behavior. - Reset during idle and transaction activity, according to the
defined reset contract.

**Level 4 --- SoC/program verification** - Arithmetic program. - Memory
load/store program. - Branch and jump program. - Program that writes and
reads GPIO. - Timer configuration and status checking. - UART serial
output checking. - Randomized memory response delays. - End-to-end
comparison against a software reference or expected architectural
results.

### UVM environment

``` text
                    soc_tb
                       |
                    soc_test
                       |
                    soc_env
          ______________|______________
         |              |              |
      AXI Agent      APB Agent      CPU Checker
         |              |              |
     Sequencer       Sequencer     Program Tests
         |              |
       Driver         Driver
         |              |
       Monitor        Monitor
          \             /
           \           /
            +---------+
                |
            Scoreboard
                |
          Reference Model
                |
          Functional Coverage
```

Use active AXI/APB agents for standalone IP testing. Use passive
monitors during CPU-driven SoC tests; do not have an active agent drive
a bus already controlled by the CPU.

### UVM components

  -----------------------------------------------------------------------
  Component                           Purpose
  ----------------------------------- -----------------------------------
  `axi_lite_agent`                    AXI transaction generation and
                                      monitoring

  `apb_agent`                         APB transaction generation and
                                      monitoring

  `axi_lite_sequencer`                AXI sequences

  `axi_lite_driver`                   AXI signal driving

  `axi_lite_monitor`                  AXI transaction observation

  `apb_driver`                        APB signal driving

  `apb_monitor`                       APB transaction observation

  `soc_scoreboard`                    Expected-versus-actual checking

  `reference_model`                   Expected memory/register behavior

  `pipeline_checker`                  Check architectural instruction
                                      results and pipeline invariants

  `soc_coverage`                      Functional coverage

  `soc_virtual_sequencer`             Coordinate agents

  `soc_virtual_sequence`              Coordinate system scenarios
  -----------------------------------------------------------------------

A full UVM agent for every internal CPU pipeline stage is unnecessary.
Verify pipeline behavior using directed instruction programs,
architectural checking, monitors and assertions. Apply UVM where
transaction-level reuse provides value.

### Required test plan

  Test                      Objective
  ------------------------- -------------------------------------
  `reset_test`              Reset and initialization
  `alu_directed_test`       ALU correctness
  `register_file_test`      Register behavior and x0
  `decoder_test`            Instruction field decoding
  `forwarding_test`         EX/MEM and MEM/WB forwarding
  `load_use_hazard_test`    Required stall and bubble
  `branch_flush_test`       Taken branch flush
  `pipeline_stall_test`     Hold pipeline during memory wait
  `cpu_arithmetic_test`     Arithmetic instruction execution
  `cpu_load_store_test`     Memory instruction behavior
  `axi_read_write_test`     AXI read/write correctness
  `axi_backpressure_test`   VALID/READY stalls
  `axi_strobe_test`         Byte-enable behavior
  `axi_error_test`          Error response handling
  `apb_wait_state_test`     APB PREADY wait states
  `gpio_test`               GPIO register behavior
  `timer_test`              Counter and compare behavior
  `uart_tx_test`            UART serial frame correctness
  `invalid_address_test`    Unmapped address response
  `soc_program_test`        End-to-end CPU program execution
  `random_soc_test`         Randomized bus delays and scenarios

### Assertions

At minimum, implement assertions for: 1. AXI VALID remains asserted
until handshake. 2. AXI payload remains stable while VALID is high and
READY is low. 3. Accepted requests receive the defined response. 4. APB
ACCESS follows SETUP. 5. APB control/address signals remain stable
during wait states. 6. No two APB slaves are selected simultaneously. 7.
Register x0 always reads as zero. 8. A stalled pipeline register retains
its state. 9. A flushed instruction cannot commit a register write or
issue a memory transaction. 10. A load-use hazard causes the specified
stall/bubble. 11. A completed older instruction is not lost when a
younger instruction stalls. 12. Reset places control logic and valid
bits into a known state.

### Functional coverage

Cover: - Instruction class × execution outcome. - ALU operation ×
operand patterns. - Forwarding source × operand position. - Load-use
hazard occurrence. - Branch type × taken/not-taken. - Flush × pipeline
stage valid bits. - Memory operation × byte lane. - AXI read/write ×
response. - AXI channel stall scenarios. - APB transfer type ×
wait-state count. - Peripheral register × access type. - Timer and UART
configurations.

Aim for at least 90% of the planned functional coverage model, review
uncovered bins and report exclusions. Coverage percentage alone does not
prove correctness.

------------------------------------------------------------------------

## 8. Five-person responsibility division

Each person owns an RTL subsystem and contributes to verification. Every
RTL owner must write unit tests for their blocks.

### Member 1 --- Pipeline architecture and control

**RTL** - PC and fetch control. - Pipeline registers. - Pipeline control
and valid bits. - Stall, bubble and flush logic. - Top-level CPU
integration.

**Verification** - Pipeline stall/flush tests. - Branch flush tests. -
Pipeline state assertions. - CPU integration tests.

**Deliverable:** Correct five-stage pipeline control and integration.

### Member 2 --- CPU datapath and execution

**RTL** - ALU. - Register file. - Instruction decoder. - Immediate
generator. - Branch comparator. - Write-back selection.

**Verification** - ALU and register-file tests. - Decoder/immediate
tests. - Directed instruction tests. - Architectural result checking.

**Deliverable:** Verified CPU datapath and agreed instruction subset.

### Member 3 --- Hazards, memory and AXI

**RTL** - Forwarding unit. - Hazard detection unit. - Load/store unit. -
CPU data interface. - AXI4-Lite master adapter. - Data SRAM controller.

**Verification** - Forwarding and load-use tests. - AXI UVM agent and
sequences. - Memory scoreboard. - Backpressure and byte-strobe tests.

**Deliverable:** Correct hazard handling and CPU data transactions.

### Member 4 --- AXI/APB subsystem and peripherals

**RTL** - AXI4-Lite router/address decoder. - AXI4-Lite-to-APB bridge. -
APB decoder. - GPIO, timer and UART TX.

**Verification** - APB UVM agent. - Bridge and peripheral sequences. -
APB assertions. - Peripheral register and behavior tests.

**Deliverable:** Working memory-mapped peripherals.

### Member 5 --- Verification integration and SoC integration support

**RTL** - Instruction memory. - SoC top-level integration support. -
Reset/clock integration. - Integration fixes, jointly reviewed.

**Verification** - UVM environment integration. - Virtual sequences. -
Pipeline/SoC scoreboard coordination. - Functional coverage. -
Regression scripts and reports.

**Deliverable:** Reproducible system-level regression and consolidated
verification evidence.

**Team rule:** Member 5 coordinates verification; they are not the only
verification contributor. Every member must review another member's RTL
and contribute tests or assertions.

------------------------------------------------------------------------

## 9. Eight-week execution roadmap

  -----------------------------------------------------------------------
  Week                    Main work               Required milestone
  ----------------------- ----------------------- -----------------------
  1                       Pipeline design,        Architecture and
                          instruction subset,     interfaces frozen
                          interfaces, memory map, 
                          tool validation         

  2                       ALU, register file,     Unit-level blocks
                          decoder, immediate      simulate
                          generator, pipeline     
                          registers, memories     

  3                       Pipeline integration,   Arithmetic/dependency
                          forwarding, hazard      programs pass
                          detection, branch flush 

  4                       Load/store path,        CPU memory programs
                          AXI4-Lite master,       pass
                          router and SRAM         
                          integration             

  5                       APB bridge, GPIO,       CPU can access
                          timer, UART TX; UVM     peripherals
                          agents                  

  6                       Pipeline corner cases,  Pipeline and protocol
                          SVA, constrained-random regression
                          bus tests               

  7                       Program-driven SoC      End-to-end scenarios
                          tests, randomized       pass
                          delays, coverage        

  8                       Regression, debug,      Final reproducible demo
                          coverage review,        
                          documentation and       
                          demonstration           
  -----------------------------------------------------------------------

### Weekly deliverables

**Week 1** - Freeze pipeline stage responsibilities and
pipeline-register fields. - Freeze instruction subset. - Define branch
policy and hazard behavior. - Define instruction/data memory
interfaces. - Freeze CPU data handshake, AXI/APB interfaces and memory
map. - Validate Questa SystemVerilog/UVM setup.

**Week 2** - Complete ALU, register file, decoder, immediate generator
and pipeline registers. - Implement instruction and data memory
models. - Run unit-level directed tests.

**Week 3** - Integrate IF/ID/EX/MEM/WB. - Implement forwarding and
load-use hazard detection. - Implement branch flush. - Run
dependent-instruction and branch programs.

**Week 4** - Implement load/store unit and CPU data interface. -
Implement AXI4-Lite master and router. - Demonstrate CPU access to data
SRAM with variable response delays.

**Week 5** - Implement AXI-to-APB bridge and peripherals. - Build UVM
AXI/APB agents and scoreboards. - Demonstrate CPU-controlled
GPIO/timer/UART TX.

**Week 6** - Complete pipeline assertions and corner-case tests. - Add
constrained-random bus delays and protocol tests. - Run regression.

**Week 7** - Run program-driven SoC tests. - Check UART serial frames
and timer behavior. - Review functional coverage and debug failures.

**Week 8** - Run final regression with multiple seeds. - Review coverage
gaps and known limitations. - Prepare architecture, verification and
demonstration reports.

Keep 5--10 January as contingency if starting on 10 November 2026.

------------------------------------------------------------------------

## 10. Research references

Use references to understand architecture and specifications. Do not
copy complete DUT implementations.

### RISC-V and pipeline design

-   [RISC-V specifications](https://riscv.org/technical/specifications/)
-   Study RV32I instruction formats: R, I, S, B, U and J.
-   Study forwarding, load-use hazards, branch flushing and pipeline
    control.
-   Use a textbook pipeline datapath as a conceptual reference.
-   Write an instruction support matrix before coding.

### AXI4-Lite and APB

-   [Arm AMBA
    specifications](https://www.arm.com/architecture/system-architectures/amba/amba-specifications)
-   Study independent AW/W handshakes.
-   Study VALID/READY backpressure.
-   Study B/R response behavior.
-   Study APB SETUP, ACCESS, PREADY and PSLVERR.

### UVM and verification

-   Study UVM test, environment, agent, sequencer, driver, monitor and
    scoreboard.
-   Study sequence items, constrained randomization and objections.
-   Study SVA and covergroups.
-   [OpenTitan DV
    methodology](https://opentitan.org/book/doc/contributing/dv/) is a
    methodology reference, not a source of DUT RTL to import.

### Toolchain

-   Validate QuestaSim SystemVerilog and UVM support.
-   Install/validate a RISC-V compiler toolchain for the implemented
    instruction subset.
-   Create a program-to-hex/memory-image flow.
-   Confirm testbench memory initialization and reset behavior.

------------------------------------------------------------------------

## 11. Suggested repository structure

``` text
Custom-Pipelined-RV32I-SoC/
│
├── rtl/
│   ├── cpu/
│   │   ├── cpu_top.sv
│   │   ├── program_counter.sv
│   │   ├── instruction_decoder.sv
│   │   ├── register_file.sv
│   │   ├── immediate_generator.sv
│   │   ├── alu.sv
│   │   ├── branch_unit.sv
│   │   ├── forwarding_unit.sv
│   │   ├── hazard_detection_unit.sv
│   │   ├── pipeline_control.sv
│   │   ├── if_id_reg.sv
│   │   ├── id_ex_reg.sv
│   │   ├── ex_mem_reg.sv
│   │   └── mem_wb_reg.sv
│   │
│   ├── bus/
│   │   ├── cpu_data_if.sv
│   │   ├── axi_lite_master.sv
│   │   └── axi_lite_router.sv
│   │
│   ├── memory/
│   │   ├── instruction_memory.sv
│   │   ├── data_sram.sv
│   │   └── sram_controller.sv
│   │
│   ├── bridge/
│   │   └── axi_lite_to_apb.sv
│   │
│   ├── peripherals/
│   │   ├── apb_decoder.sv
│   │   ├── gpio.sv
│   │   ├── timer.sv
│   │   └── uart_tx.sv
│   │
│   └── top/
│       └── soc_top.sv
│
├── tb/
│   ├── interfaces/
│   ├── agents/
│   ├── sequences/
│   ├── scoreboards/
│   ├── coverage/
│   ├── tests/
│   └── soc_env/
│
├── programs/
│   ├── arithmetic/
│   ├── dependencies/
│   ├── branches/
│   ├── memory/
│   └── peripherals/
│
├── sim/
│   ├── compile.do
│   ├── run.do
│   └── regression.do
│
├── docs/
│   ├── architecture.md
│   ├── pipeline_specification.md
│   ├── instruction_support.md
│   ├── bus_specification.md
│   ├── register_map.md
│   ├── verification_plan.md
│   └── coverage_plan.md
│
└── README.md
```

------------------------------------------------------------------------

## 12. Acceptance criteria

### CPU and RTL

-   [ ] Team-designed five-stage pipeline.
-   [ ] ALU, register file, decoder and immediate generator pass unit
    tests.
-   [ ] Agreed instruction subset executes correctly.
-   [ ] EX/MEM and MEM/WB forwarding work.
-   [ ] Load-use hazard inserts the required stall/bubble.
-   [ ] Taken branch flushes younger wrong-path instructions.
-   [ ] Pipeline handles data-memory wait states without duplicate
    requests or lost instructions.
-   [ ] CPU accesses instruction and data memories correctly.
-   [ ] Custom AXI4-Lite master and router work.
-   [ ] Custom APB bridge handles wait states and errors.
-   [ ] GPIO, timer and UART TX work through memory-mapped registers.

### Verification

-   [ ] UVM AXI4-Lite agent.
-   [ ] UVM APB agent.
-   [ ] Monitors and scoreboards.
-   [ ] Directed and constrained-random tests.
-   [ ] SVA protocol and pipeline assertions.
-   [ ] Functional coverage report.
-   [ ] Regression with multiple random seeds.
-   [ ] All planned baseline tests pass.
-   [ ] Coverage gaps and unsupported behavior documented.

### Demonstration

-   [ ] Execute an arithmetic program.
-   [ ] Demonstrate dependent instructions requiring forwarding.
-   [ ] Demonstrate a load-use stall.
-   [ ] Demonstrate a taken branch and pipeline flush.
-   [ ] Execute a load/store program.
-   [ ] Write/read GPIO registers.
-   [ ] Configure and observe timer behavior.
-   [ ] Transmit and verify a UART frame.
-   [ ] Show AXI/APB waveforms.
-   [ ] Show UVM logs, scoreboard results and coverage report.

------------------------------------------------------------------------

## 13. Risk controls and scope cuts

A five-stage pipeline adds real complexity. The team must control scope.

1.  **No branch prediction:** Resolve branches in EX and flush on taken
    branches.
2.  **No cache:** Use simple instruction and data SRAM.
3.  **No interrupts in the first baseline:** Timer status/compare
    behavior is sufficient. Add an interrupt path only if the pipeline
    and bus are stable.
4.  **No DMA or SPI:** These do not improve the core learning objectives
    enough to justify their schedule cost.
5.  **No full AXI4:** Keep one master, two targets and one outstanding
    transaction.
6.  **Limited ISA first:** Arithmetic, logic, LW/SW, BEQ/BNE and JAL
    first. Expand only after regression passes.
7.  **No late feature additions:** Freeze mandatory features after Week
    5.
8.  **Verification starts early:** Unit tests begin in Week 2; pipeline
    and bus corner cases are tested as soon as their blocks exist.

If the team falls behind: - First remove UART TX and demonstrate
GPIO/timer only. - Next reduce the instruction subset to the agreed
minimum. - Keep the five-stage pipeline, forwarding, load-use handling,
branch flush, AXI/APB path and verification environment. These are the
central learning objectives.

------------------------------------------------------------------------

## Final project decision

Build a **custom five-stage, in-order RV32I-subset pipelined CPU SoC**.
The team designs the CPU datapath, pipeline registers, forwarding and
hazard logic, memories, AXI4-Lite data path, APB bridge and selected
peripherals. Verify the design using directed instruction programs,
SystemVerilog assertions, UVM bus agents, scoreboards, functional
coverage and repeatable regressions.

This is more balanced than a multicycle CPU project: it provides
substantial CPU microarchitecture work, meaningful SoC bus integration
and a serious verification workload. It is still ambitious for eight
weeks, so the reduced ISA, single AXI master, limited peripherals and
absence of cache/DMA/branch prediction are deliberate.

**First action:** Freeze the pipeline-register fields, instruction
subset, branch policy, hazard rules, CPU data handshake, AXI/APB
contracts and memory map before parallel RTL development begins.
