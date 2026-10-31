<p align="center">
  <img src="./docs/assets/github-social-preview-1280x640-light.png" alt="Project RADP: RISC-V Agentic Design Platform" width="100%">
</p>

<p align="center">
  <b>AI agents design RISC-V cores. Open-source EDA tools verify them.</b>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-blue.svg" alt="License: Apache-2.0"></a>
  <img src="https://img.shields.io/badge/status-pre--alpha-orange.svg" alt="Status: pre-alpha">
  <a href="https://halid.org"><img src="https://img.shields.io/badge/blog-halid.org-slategray.svg" alt="Blog"></a>
</p>

---

> [!WARNING]
> Project RADP is in early development. Interfaces, file formats and the spec schema will change. Sections marked **(planned)** describe intended behavior, not working features.

## What is RADP?

Project RADP (RISC-V Agentic Design Platform) takes a RISC-V processor from one spec file to RTL, verification, synthesis and an FPGA bitstream, with shuttle tapeout as the next target.

AI agents do the work: writing RTL, testbenches, assertions and constraints. Deterministic open-source tools make every decision: simulators, ISA reference models, formal engines and timing analysis. A change is accepted only when the tools say it passes, and a human approves the merge.

**Agents propose, tools decide.**

## Principles

- **Spec-driven.** Every run starts from a versioned `chip.yaml` validated against a JSON Schema. Agents never change the spec without a reviewed change request.
- **Gated.** Each stage has a gate backed by a tool result, not by model judgment.
- **Inspectable.** Every agent step records inputs, tool calls, diff, gate result and cost.
- **Bounded.** Agents run with iteration and cost budgets. Repeated failure signatures escalate to a human instead of looping.
- **Reproducible.** Tools are pinned with Nix, run in rootless, network-isolated containers, and results are content-hash cached.
- **Open toolchain only.** No proprietary EDA anywhere in the pipeline.

## Pipeline

```
chip.yaml
   │
   ▼
 RTL ─► lint ─► unit sim ─► compliance ─► lockstep ─► formal ─► coverage ─► synth/timing ─► FPGA / GDS
   ▲                                                                                           │
   └───────────────────────── agents iterate on failures, within budget ──────────────────────┘
```

| Stage | Gate | Tools |
| --- | --- | --- |
| Lint | Zero errors, waivers reviewed | Verilator, Verible |
| Unit sim | All tests pass | cocotb, Verilator |
| Compliance | ISA test suite passes | riscv-arch-test, RISCOF, Sail |
| Lockstep | Commit trace matches reference model | Spike |
| Formal | Properties prove or bounded checks pass | riscv-formal, SymbiYosys |
| Coverage | Targets in spec are met | Verilator coverage, riscv-dv |
| Synth / timing | Meets frequency target | Yosys, nextpnr (ECP5), OpenROAD |
| FPGA / GDS | Bitstream boots; GDS passes DRC/LVS | Trellis, LibreLane, KLayout |

## Agents

| Role | Responsibility |
| --- | --- |
| Architect | Turns `chip.yaml` into a microarchitecture plan and module breakdown |
| RTL Designer | Writes and fixes SystemVerilog under `rtl/` |
| Testbench Writer | Writes cocotb tests under `tb/` (cannot edit RTL) |
| Formal | Writes assertions and formal configs |
| Coverage Analyst | Finds coverage holes and requests new tests |
| Debug | Triages failures, assigns an owner (RTL or test), proposes a fix |
| PPA Optimizer | Improves timing, area and power within the spec |
| Physical | Floorplan, constraints and implementation flow settings |
| Reviewer | Reviews diffs against the spec before a human sees them |

Write access is enforced per role by path, so a single agent cannot change both the design and the test that judges it.

## Spec example

```yaml
# chip.yaml (schema subject to change)
name: core0
isa:
  base: RV32I
  extensions: [M]
microarch:
  pipeline: 5-stage
  branch_predictor: static
memory:
  imem: { size: 16KiB }
  dmem: { size: 16KiB }
targets:
  - type: fpga
    device: ecp5-25k
    clock_mhz: 50
verification:
  gates: [lint, unit_sim, compliance, lockstep, formal, coverage, timing]
  coverage:
    line: 95
    toggle: 85
budget:
  max_cost_usd: 10
  max_iterations_per_stage: 3
```

## Architecture

- **Orchestrator:** a Go DAG runner with SQLite state for the MVP, Temporal later.
- **Tool contracts:** every EDA tool sits behind a typed MCP interface with structured results and failure signatures.
- **Executors:** local or remote Docker workers; Kubernetes Jobs later.
- **LLM gateway:** a provider abstraction with per-role model tiers and cost tracking. Secrets stay in the control plane, never in sandboxes.
- **Review:** agents push branches and open pull requests. Gates run automatically; merges are human.
- **Web UI (planned):** run view, change review, debug reports with waveforms, spec editor.

## Getting started (planned)

```bash
# Requirements: Nix, Docker (rootless), an LLM API key
git clone https://github.com/ORG/radp.git && cd radp
nix develop

radp init core0 --template rv32im-5stage
radp run --target ecp5
radp status
```

## Roadmap

- [ ] Spec schema v0 and validator
- [ ] Typed tool contracts for lint, simulation, compliance and lockstep
- [ ] Orchestrator with gates, budgets and escalation
- [ ] RTL Designer, Testbench Writer and Debug agents
- [ ] First demo: `chip.yaml` to CoreMark running on an ECP5 board
- [ ] Formal and coverage stages
- [ ] Web UI: run view, change review, debug report
- [ ] Agent eval harness (CVDP subset, golden designs)
- [ ] ASIC flow with LibreLane on Sky130 / GF180 shuttles

Progress notes are published on [halid.org](https://halid.org).

## Repository layout (planned)

```
radp/
├── cmd/            CLI and services
├── orchestrator/   DAG runner, gates, budgets
├── agents/         role prompts, policies, write-path rules
├── tools/          MCP tool contracts and adapters
├── schema/         chip.yaml JSON Schema
├── sandbox/        container images and Nix definitions
├── web/            web UI
├── evals/          benchmark tasks and harness
└── docs/
```

## Contributing

Feedback from RTL, verification and physical design engineers is the most valuable contribution right now. Open an issue or a discussion.

Code contributions are welcome under Apache-2.0 and require signing the Contributor License Agreement. See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

Apache-2.0. See [`LICENSE`](LICENSE).

GPL-licensed tools (for example Verilator and Yosys plugins) are invoked as separate processes and are not linked into RADP. Designs produced by the platform belong to the user.

## Acknowledgements

RADP builds on the work of the open-source hardware community: RISC-V International, YosysHQ, the OpenROAD project, FOSSi Foundation (LibreLane), Verilator, cocotb, the Spike and Sail maintainers, and the SkyWater, GlobalFoundries and IHP open PDK teams.

<sub>RISC-V is a trademark of RISC-V International. Project RADP is an independent project and is not affiliated with RISC-V International.</sub>
