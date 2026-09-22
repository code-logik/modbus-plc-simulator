# Modbus PLC Simulator | Engineering Case Study

A reusable, appliance-style Modbus/TCP simulator for industrial HMI integration, protocol testing, controlled fault injection, and OT/ICS test-bench workflows without requiring continuous access to a physical PLC.

> **Public portfolio edition.** This repository documents architecture and engineering practices, not a distributable simulator. The underlying implementation belongs to a private professional project. Proprietary code, production-derived register maps, organizational identifiers, internal network details, deployment artifacts, and credentials are not published. All examples here are synthetic.

**Case-study implementation reference:** `2.0.0-preview.4` (reviewed September 2026). This identifies the private codebase snapshot used to reconcile the case study; it is not a public software release or proof of a particular site deployment.

## Engineering problem

Industrial software and network-security testing needs a consistent Modbus/TCP endpoint. Physical PLC availability, changing equipment state, and specialized register layouts can make integration and troubleshooting difficult to repeat. A useful simulator must exercise **real network-side protocol requests** while allowing an engineer to change process-like values and intentionally simulate communication loss from a separate, local control plane.

The engineering objectives were to:

- Represent a typed, mapped PLC address space, including multiword register values and coils.
- Support external read/write workflows and return protocol exceptions for unmapped addresses instead of fabricating register values.
- Reproduce repeatable conditions using saved scenarios and direct operator-set values.
- Observe network traffic without mistaking the simulator's internal UI access for external Modbus activity.
- Keep the Modbus service independent of the UI and limit diagnostic memory and disk growth.
- Preserve a predictable Linux-appliance startup and a controlled configuration/deployment workflow.

## Implemented approach

The private implementation uses a **Python/PyModbus server** and a **separate Avalonia/.NET 8 fullscreen operator interface**. They communicate over a local Unix-domain JSON control socket. The Python process owns the live datastore, simulation engine, Modbus/TCP listener, and diagnostics. The UI is not in the Modbus request/response path; its failure should not, by itself, terminate the protocol server.

The earlier Tkinter/X11 interface is retained as a startup fallback. The current primary interface is **not** Tkinter. A Linux service supervises the backend, while the local kiosk/session launches the frontend; the two are not a single supervised process.

See [Architecture](docs/ARCHITECTURE.md) for the component boundaries, flows, and failure behavior.

## Capabilities reflected in the reviewed source

| Capability | Engineering behavior |
| --- | --- |
| Typed register model | Mapped holding registers and coils; typed values and configurable multiword ordering; distinct human-readable reference and zero-based PDU addressing. |
| Network protocol | Modbus/TCP read/write operations, mapped address validation, and exception responses for unsupported/unmapped operations. |
| Scenario execution | Saved view-associated scenarios can run manual, static, increment/decrement, wrap/counter, toggle, ramp, sawtooth, random, sine, and clock behaviors. The UI presents a narrower common-mode selection than the engine's full supported set. |
| Local intervention | An operator can apply/stop a scenario, inspect state, and set a live tag value without sending a Modbus request. A live manual value does not rewrite the saved scenario file. |
| External write ownership | A **confirmed, accepted** client write to a configured writable tag relinquishes that tag from periodic scenario updates. A rejected or failed request does not. A new scenario or explicit reconfiguration can reassign control. |
| Communication-loss test | **Block PLC Comms** stops the Modbus TCP listener while leaving backend control/diagnostics and the UI available; **Restore PLC Comms** restarts the listener against the existing datastore rather than resetting registers or write ownership. |
| External JSON definitions | HMI views and scenarios are stored as separate JSON definitions and can be reloaded without modifying application code. Saved traffic views filter observed transactions, not PLC values. |
| Network observability | Live decoded transactions and a separate passive packet view support correlation of real Modbus/TCP activity. The UI displays up to **18** matching live transaction rows and **30** newest packet rows, with larger bounded RAM buffers behind them. |
| Retained evidence | Event-oriented history includes accepted/rejected write context, exceptions/errors, selected manual changes, lifecycle/diagnostic events, and periodic summaries. Routine reads and raw packets remain in RAM rather than continuously growing on-disk history. |
| Data export | Current in-memory traffic can be exported as CSV, JSON, or PCAP. Exporting a live buffer is distinct from querying retained, rotated event history. |

**Important simulation distinction:** A scenario entry with `"mode": "manual"` disables its periodic scenario writes; its JSON `value` is **not** automatically applied to the datastore on scenario start. An initial value must come from the loaded catalog at datastore creation or from an explicit manual/client write. The examples below do not claim to seed live register values.

## Observability and reliability decisions

The private design uses two complementary network evidence paths. Transaction records are created by handling external Modbus operations; passive packet capture is an independent observer of packets on the test interface. Neither the UI's local control-socket polling nor an internal scenario update should be labeled as an external network request. Internal value changes can instead be identified by their change origin.

Live transaction and packet buffers are bounded. Persistent diagnostics are queued outside the response path and emphasize meaningful events, not every repeated read or raw packet. The reviewed source configures rotating history files and free-space protection; it also reports capture and persistence issues rather than treating every event as guaranteed durable evidence. Packet capture depends on available operating-system permissions and network visibility. See [Validation and limitations](docs/VALIDATION.md).

The communication-block feature simulates **loss of the Modbus service**, not power loss of the appliance or guaranteed behavior of any specific HMI. Observed HMI behavior must be measured separately by the tester.

## Reusable definitions with synthetic examples

A view groups named tags. A scenario references a view and associates individual tags with simulation behaviors. A traffic view names tags to include when filtering *observed* transactions; it does not generate reads or execute a scenario. Definitions are file-based; supported updates can be picked up by the running application. The current UI additionally checks saved traffic-view files for changes while Live Traffic is open.

The following examples match the **shape** of the reviewed view/scenario definitions and traffic-view model. Their tag identifiers intentionally do not correspond to the private tag catalog. They are illustrations, not import-ready definitions for the private appliance.

- [Synthetic HMI view](docs/examples/demo-process.hmi-view.json)
- [Synthetic scenario](docs/examples/demo-process.scenario.json)
- [Synthetic traffic filter](docs/examples/demo-process.traffic-view.json)

```json
{
  "schema_version": 1,
  "kind": "hmi_view",
  "name": "Demo Process View",
  "description": "Illustrative tags only; not a real PLC map.",
  "tags": ["DEMO_PRESSURE", "DEMO_TEMPERATURE", "DEMO_RUN_STATE"]
}
```

The previous portfolio example used a `points` array. The reviewed HMI-view format uses `schema_version`, `kind`, and **`tags`** instead.

## Deployment engineering

The project is designed as a Linux test appliance with a managed Python environment, configured network binding, persistent JSON definitions, backend service supervision, and automatic local kiosk startup. Configuration edits and application deployments are handled as controlled changes with validation and rollback considerations. The simulator is a test endpoint; it is **not** a production PLC, a safety controller, or a security enforcement product.

The portfolio deliberately excludes real interface names, IPs, routing, privilege configuration, installation scripts, executable artifacts, tag exports, and production configurations. An unrelated reader should not be able to reconstruct the private environment from this repository.

## Verification and evidence

The codebase contains a Modbus functional self-test, model/engine checks, build and deployment validation guidance, and explicit traffic and recovery behaviors. This case-study revision performed **static source-to-documentation reconciliation**; it did **not** run a physical-device, live Modbus-network, HMI, firewall, reboot, or packet-capture acceptance test. The documented verification matrix separates what can be checked in source from what needs a controlled test bench: [Validation and limitations](docs/VALIDATION.md).

## Technologies and engineering practices

**Industrial / OT:** Modbus/TCP, mapped coils and holding registers, multiword data, HMI integration, controlled service-interruption testing, industrial network segmentation in a test bench.

**Software:** Python, PyModbus, asynchronous state management, Avalonia/.NET 8, C#, JSON definitions, schema-aware configuration, separation of protocol/data/control/observation concerns.

**Linux and networking:** Linux service supervision, local X11/kiosk session, Unix-domain sockets, TCP/IP, passive packet capture, bounded in-memory telemetry, asynchronous event persistence.

**Engineering methods:** Source-to-documentation traceability, validation design, reproducible scenarios, failure isolation, controlled changes, rollback planning, and evidence qualification.

## Engineering role and AI-assisted work

I defined the requirements, integrated the simulator, reviewed the technical decisions, and directed validation of the private professional project. AI tools assisted with portions of code drafting, troubleshooting, test planning, and documentation; their output was reviewed rather than treated as independent validation. This public case study does not transfer ownership or licensing rights in the underlying private implementation.

## Publication and use boundary

Do not add the private source tree, production-derived register exports, proprietary manuals, company/customer references, device IDs, credentials, internal logs/packet captures, internal network settings, or real deployment assets to this public repository. Synthetic examples and generalized design explanations are the intended scope. The descriptions here are tied to the reviewed snapshot, not a guarantee that every deployment has the same configuration or operational result.
