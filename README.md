# Systolic Array Processor with Approximate Computing

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Verilog](https://img.shields.io/badge/Verilog-HDL-blue.svg)](https://en.wikipedia.org/wiki/Verilog)
[![Python](https://img.shields.io/badge/Python-3.8+-green.svg)](https://www.python.org/)
[![GitHub issues](https://img.shields.io/github/issues/yourusername/systolic-array)](https://github.com/yourusername/systolic-array/issues)
[![GitHub stars](https://img.shields.io/github/stars/yourusername/systolic-array)](https://github.com/yourusername/systolic-array/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/yourusername/systolic-array)](https://github.com/yourusername/systolic-array/network)

---

## Overview

**Systolic Array Processor with Approximate Computing** is a configurable hardware accelerator for high-throughput matrix operations, convolution, and tensor workloads. It combines a scalable systolic array, multiple dataflow modes, and a custom approximate multiplier to reduce power while keeping performance high.

The design is intended for:
- FPGA prototyping.
- ASIC research.
- AI/ML acceleration.
- Signal and image processing.

---

## Table of Contents

- [Overview](#overview)
- [Why Systolic Array](#why-systolic-array)
- [Key Features](#key-features)
- [Architecture Overview](#architecture-overview)
- [System Block Diagram](#system-block-diagram)
- [Processing Element](#processing-element)
- [Dataflow Techniques](#dataflow-techniques)
- [Custom Approximate Multiplier](#custom-approximate-multiplier)
- [Array Size Expansion](#array-size-expansion)
- [Global Control FSM](#global-control-fsm)
- [Supported Operations](#supported-operations)
- [Power-Efficient Model](#power-efficient-model)
- [Installation](#installation)
- [Usage](#usage)
- [Performance Metrics](#performance-metrics)
- [Testing and Verification](#testing-and-verification)
- [Future Work](#future-work)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)
- [Contact](#contact)

---

## Why Systolic Array

A systolic array is a regular mesh of processing elements that computes by passing data rhythmically between neighbors. This reduces memory traffic, increases reuse, and makes the architecture easier to scale.

Compared with a traditional CPU-style flow, this design is better suited for:
- Matrix multiplication.
- Convolution.
- Neural network inference.
- Streaming DSP workloads.

---

## Key Features

| Feature | Description |
|--------|-------------|
| Configurable size | 2^N × 2^N array, where N = 1 to 8 |
| Multiple dataflows | OS, WS, IS, and Axon Systolic |
| Approximate multiplier | Lower power, configurable accuracy |
| Global FSM | Centralized control for execution flow |
| Scalable design | Suitable for small to large arrays |
| Python tools | Analysis, profiling, and plotting |
| Verification support | Testbenches and waveform inspection |
| Synthesis ready | FPGA and ASIC compatible |

---

## Architecture Overview

The design is organized into a top-level controller, a 2D array of processing elements, interconnect logic, and supporting analysis scripts.

### Main modules
- `top.v`: Top-level integration.
- `pe.v`: Processing element.
- `approx_multiplier.v`: Approximate multiplier.
- `fsm.v`: Global control state machine.
- `array.v`: Systolic array interconnect.
- `tb/`: Testbench files.
- `tools/`: Python analysis scripts.

---

## System Block Diagram

```mermaid
flowchart TD
    A[Input Matrices A and B] --> B[Load Controller]
    B --> C[Global FSM]
    C --> D[Systolic Array Top Module]
    D --> E[Processing Element Grid]
    E --> F[Interconnect Network]
    E --> G[Approximate Multiplier]
    G --> H[Partial Sum Accumulation]
    H --> I[Output Matrix C]
    C --> J[Status Signals: start, done, ready]
```

This flow shows how operands enter the controller, get distributed into the array, and produce the final result.

---

## Processing Element

Each processing element performs multiply-accumulate operations and forwards values to adjacent PEs.

```verilog
module pe #(
    parameter DATA_WIDTH = 16
)(
    input  wire                   clk,
    input  wire                   rst,
    input  wire                   valid_in,
    input  wire [DATA_WIDTH-1:0]  a_in,
    input  wire [DATA_WIDTH-1:0]  b_in,
    input  wire [2*DATA_WIDTH-1:0] c_in,
    output reg  [DATA_WIDTH-1:0]  a_out,
    output reg  [DATA_WIDTH-1:0]  b_out,
    output reg  [2*DATA_WIDTH-1:0] c_out
);

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            a_out <= 0;
            b_out <= 0;
            c_out <= 0;
        end else if (valid_in) begin
            a_out <= a_in;
            b_out <= b_in;
            c_out <= c_in + (a_in * b_in);
        end
    end
endmodule
```

This PE structure is simple, pipeline-friendly, and easy to replicate across the array.

---

## Dataflow Techniques

### Output Stationary (OS)

In OS mode, partial sums stay inside the PE while inputs move across the array. This reduces output movement and is efficient for standard matrix multiplication.

```verilog
always @(posedge clk) begin
    if (dataflow_mode == OS) begin
        a_out <= a_in;
        b_out <= b_in;
        c_reg <= c_reg + (a_in * b_in);
        c_out <= c_reg;
    end
end
```

### Weight Stationary (WS)

In WS mode, weights remain fixed inside the PEs while inputs stream through. This is useful for inference workloads.

```verilog
always @(posedge clk) begin
    if (dataflow_mode == WS) begin
        a_out <= a_in;
        c_reg <= c_reg + (a_in * weight_reg);
        c_out <= c_reg;
    end
end
```

### Input Stationary (IS)

In IS mode, inputs stay fixed while weights flow through the array. This is useful for streaming and filter-based applications.

```verilog
always @(posedge clk) begin
    if (dataflow_mode == IS) begin
        b_out <= b_in;
        c_reg <= c_reg + (input_reg * b_in);
        c_out <= c_reg;
    end
end
```

### Axon Systolic

In Axon mode, values propagate diagonally for high-throughput matrix multiplication.

```verilog
always @(posedge clk) begin
    if (dataflow_mode == AXON) begin
        a_out <= (i == 0) ? a_in : pe_a_in[i-1][j];
        b_out <= (j == 0) ? b_in : pe_b_in[i][j-1];
        c_reg <= c_reg + (a_out * b_out);
        c_out <= c_reg;
    end
end
```

### Dataflow Comparison


### Dataflow Comparison

| Mode | Latency | Throughput | Power | Best Use |
|------|---------|------------|-------|----------|
| OS | Medium | High | Medium | Matrix multiplication |
| WS | High | Medium | Low | DNN inference |
| IS | Medium | High | Medium | DSP / streaming |
| AXON | Low | Very High | Medium | Real-time processing |

---

## PE Array and Dataflow Propagation

### Generic PE Array Layout

```mermaid
flowchart LR
    A[A Matrix Inputs] --> R0[Row Input Network]
    B[B Matrix Inputs] --> C0[Column Input Network]

    subgraph SA[Systolic Array PE Grid]
        direction LR

        subgraph ROW0[Row 0]
            PE00[PE00] --> PE01[PE01] --> PE02[PE02] --> PE03[PE03]
        end

        subgraph ROW1[Row 1]
            PE10[PE10] --> PE11[PE11] --> PE12[PE12] --> PE13[PE13]
        end

        subgraph ROW2[Row 2]
            PE20[PE20] --> PE21[PE21] --> PE22[PE22] --> PE23[PE23]
        end

        subgraph ROW3[Row 3]
            PE30[PE30] --> PE31[PE31] --> PE32[PE32] --> PE33[PE33]
        end
    end

    R0 --> PE00
    R0 --> PE10
    R0 --> PE20
    R0 --> PE30

    C0 --> PE00
    C0 --> PE01
    C0 --> PE02
    C0 --> PE03
```

### OS Data Propagation

```mermaid
flowchart TD
    A[A values move right] --> PE00[PE00]
    B[B values move down] --> PE00

    PE00 --> PE01[PE01]
    PE01 --> PE02[PE02]
    PE02 --> PE03[PE03]

    PE00 --> PE10[PE10]
    PE10 --> PE20[PE20]
    PE20 --> PE30[PE30]

    PE00 --> C00[C00 accumulates in PE]
    PE01 --> C01[C01 accumulates in PE]
    PE02 --> C02[C02 accumulates in PE]
    PE03 --> C03[C03 accumulates in PE]

    PE10 --> C10[C10 accumulates in PE]
    PE20 --> C20[C20 accumulates in PE]
    PE30 --> C30[C30 accumulates in PE]
```

### WS Data Propagation

```mermaid
flowchart TD
    A[Input activations A] --> P0[PEs]
    A --> P1[PEs]
    A --> P2[PEs]

    W[Weights B stay stationary in PEs] --> P0
    W --> P1
    W --> P2

    P0 --> P0C[C accumulation]
    P1 --> P1C[C accumulation]
    P2 --> P2C[C accumulation]

    P0 --> A1[Next input]
    P1 --> A2[Next input]
    P2 --> A3[Next input]
```

### IS Data Propagation

```mermaid
flowchart TD
    A[Input A stays in PE] --> PE00[PE00]
    A --> PE10[PE10]
    A --> PE20[PE20]
    A --> PE30[PE30]

    B[Weights B move through array] --> PE00
    PE00 --> PE01[PE01]
    PE01 --> PE02[PE02]
    PE02 --> PE03[PE03]

    B --> PE10
    PE10 --> PE11[PE11]
    PE11 --> PE12[PE12]
    PE12 --> PE13[PE13]

    PE00 --> C00[C accumulation]
    PE10 --> C10[C accumulation]
    PE20 --> C20[C accumulation]
    PE30 --> C30[C accumulation]
```

### AXON Diagonal Feeding

```mermaid
flowchart TD
    A0[A00] --> PE00[PE00]
    A1[A10] --> PE10[PE10]
    A2[A20] --> PE20[PE20]
    A3[A30] --> PE30[PE30]

    B0[B00] --> PE00
    B1[B10] --> PE01[PE01]
    B2[B20] --> PE02[PE02]
    B3[B30] --> PE03[PE03]

    PE00 --> PE01
    PE01 --> PE02
    PE02 --> PE03

    PE10 --> PE11[PE11]
    PE11 --> PE12[PE12]
    PE12 --> PE13[PE13]

    PE20 --> PE21[PE21]
    PE21 --> PE22[PE22]
    PE22 --> PE23[PE23]

    PE30 --> PE31[PE31]
    PE31 --> PE32[PE32]
    PE32 --> PE33[PE33]

    PE00 --> D1[Diagonal propagation]
    PE11 --> D2[Diagonal propagation]
    PE22 --> D3[Diagonal propagation]
    PE33 --> D4[Diagonal propagation]
```

### Dataflow Summary

- **OS (Output Stationary):** partial sums remain in the PE.
- **WS (Weight Stationary):** weights remain in the PE.
- **IS (Input Stationary):** inputs remain in the PE.
- **AXON:** diagonal feeding pattern for fast systolic propagation.

These modes allow the same hardware to support different workloads with different performance and power trade-offs.

---

## Custom Approximate Multiplier

The approximate multiplier reduces power and area by using approximation in lower-order bits.

### Key ideas
- LSB truncation.
- Approximate compressor logic.
- Booth-style partial products.
- Optional correction logic.

```verilog
module approx_multiplier #(
    parameter WIDTH = 16,
    parameter APPROX_BITS = 4
)(
    input  wire [WIDTH-1:0] a,
    input  wire [WIDTH-1:0] b,
    output wire [2*WIDTH-1:0] p
);

    wire [2*WIDTH-1:0] exact_product;
    assign exact_product = a * b;

    assign p = exact_product & ~((1 << APPROX_BITS) - 1);
endmodule
```

This block can be replaced by a more advanced approximate implementation if you want stronger power savings.

---

## Array Size Expansion

The array is parameterized to support power-of-two sizes.

### Supported configurations
- 2×2
- 4×4
- 8×8
- 16×16
- 32×32
- 64×64
- 128×128
- 256×256

### Expansion mechanism

```mermaid
flowchart LR
    A[2x2 Tile] --> B[4x4 Array]
    B --> C[8x8 Array]
    C --> D[16x16 Array]
    D --> E[32x32 Array]
    E --> F[64x64 Array]
```

The array grows by composing smaller reusable tiles, which keeps the design modular and easier to verify.

---

## Global Control FSM

The FSM manages array operation from reset to completion.

```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> LOAD : start
    LOAD --> COMPUTE : data_loaded
    COMPUTE --> DRAIN : done_compute
    DRAIN --> DONE : outputs_ready
    DONE --> IDLE : ack
    COMPUTE --> ERROR : fault
    LOAD --> ERROR : fault
    ERROR --> IDLE : reset
```

### Example FSM skeleton

```verilog
module control_fsm (
    input  wire clk,
    input  wire rst,
    input  wire start,
    input  wire compute_done,
    input  wire outputs_done,
    output reg  load_en,
    output reg  compute_en,
    output reg  drain_en,
    output reg  done
);

    typedef enum logic [2:0] {IDLE, LOAD, COMPUTE, DRAIN, DONE} state_t;
    state_t state, next_state;

    always @(posedge clk or posedge rst) begin
        if (rst) state <= IDLE;
        else state <= next_state;
    end
endmodule
```

---

## Supported Operations

The architecture supports multiple matrix and DSP workloads.

### Operations
- Matrix multiplication.
- Convolution.
- FIR filtering.
- Tensor-style multiply-accumulate operations.
- Streaming linear algebra kernels.

### Example matrix multiply flow

```verilog
// C = A x B
for (i = 0; i < M; i = i + 1) begin
    for (j = 0; j < N; j = j + 1) begin
        c[i][j] = 0;
        for (k = 0; k < K; k = k + 1) begin
            c[i][j] = c[i][j] + a[i][k] * b[k][j];
        end
    end
end
```

---

## Power-Efficient Model

The design reduces power through data reuse and approximate arithmetic.

### Techniques used
- Reduced memory movement.
- Lower switching activity.
- Approximate multiplication.
- Optional clock gating.
- Parameterized precision.

### Efficiency goals
- Lower MAC energy.
- Better TOPS/W.
- Reduced interconnect cost.
- Good accuracy for error-tolerant workloads.

---

## Installation

### Prerequisites
- Verilog/SystemVerilog simulator.
- Python 3.8+.
- GTKWave.
- Optional synthesis toolchain.

### Clone the repository

```bash
git clone https://github.com/yourusername/systolic-array.git
cd systolic-array
```

### Run simulation

```bash
make sim
```

### Run analysis tools

```bash
python3 tools/analysis/run_analysis.py
```

---

## Usage

### Configure the design
Set parameters in your config file or top module:
- `ARRAY_SIZE`
- `DATA_WIDTH`
- `DATAFLOW_MODE`
- `APPROX_BITS`

### Simulate a test case
1. Load matrix data.
2. Select a dataflow.
3. Run the simulation.
4. Compare output with the reference model.

### View waveforms

```bash
gtkwave dump.vcd
```

---

## Performance Metrics

| Metric | Description |
|--------|-------------|
| Throughput | Number of MACs per cycle or per second |
| Latency | Time from input load to valid output |
| Power | Dynamic and static power estimate |
| Accuracy | Match against exact multiplication |
| Area | RTL or synthesized resource usage |

---

## Testing and Verification

### Testbenches
- PE unit tests.
- Array integration tests.
- End-to-end matrix tests.
- Approximate multiplier checks.

### Verification flow
- Run RTL simulation.
- Compare outputs against a Python golden model.
- Inspect waveforms in GTKWave.
- Add assertions for control and timing correctness.

```bash
make test
make wave
```

---

## Future Work

Planned improvements include:
- RISC-V coprocessor integration.
- Sparse matrix support.
- Mixed-precision support.
- Fault tolerance.
- Memory hierarchy optimization.
- MLIR/TVM integration.
- ASIC tape-out exploration.

---

## Contributing

Contributions are welcome.

### Workflow
1. Fork the repository.
2. Create a feature branch.
3. Implement your changes.
4. Add tests.
5. Submit a pull request.

---

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.

---

## Acknowledgments

This project is inspired by the foundational and modern work in systolic architectures, VLSI design, and accelerator-based computing.

- **H.T. Kung** for introducing and popularizing systolic architectures as a general methodology for mapping computations into regular hardware structures.
- **Charles E. Leiserson** and **H.T. Kung** for early work on systolic and VLSI-oriented computation models.
- **Google Research / Google TPU team** for demonstrating the practical impact of systolic arrays in large-scale machine learning acceleration.
- **Researchers in approximate computing** for showing how accuracy-aware arithmetic can reduce power and area in digital systems.
- **Researchers in low-power VLSI architecture** for techniques such as clock gating, data reuse, and energy-aware datapath design.
- **Researchers in dataflow-based accelerators** for Output Stationary, Weight Stationary, and Input Stationary processing models.
- **Academic and industrial work on matrix multiplication accelerators, CNN accelerators, and tensor processing engines** that helped shape modern systolic-array design practices.
- **Carnegie Mellon University and related systolic architecture research groups** for advancing the original architectural concepts.
- **The open-source hardware and EDA community** for enabling RTL design, simulation, verification, and synthesis workflows.

### Selected References
- H.T. Kung, *Why Systolic Architectures?*
- H.T. Kung and Charles E. Leiserson, early systolic/VLSI computation research.
- Jouppi et al., *In-Datacenter Performance Analysis of a Tensor Processing Unit.*
- Eyeriss-related energy-efficient accelerator research.
- Approximate computing literature on low-power arithmetic and error-tolerant design.

---

## Contact

| Role | Name | Email | GitHub |
|------|------|-------|--------|
| Project Associate | GAURAV DHAK | [GAURAVDHAK@NITGOA.AC.IN](mailto:GAURAVDHAK@NITGOA.AC.IN) | @gauravdhak |

This project was developed by me as part of my own learning, curiosity, and knowledge-building journey.
---

<div align="center">
  <sub>Built with ❤️ by the Systolic Array Team</sub>
</div>
