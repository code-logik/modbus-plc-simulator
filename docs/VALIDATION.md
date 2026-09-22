# Validation and Evidence Boundaries

**Purpose:** Define what an engineer should verify before asserting that a particular appliance or connected client behaves as documented. This public case-study revision reviewed source and documentation only. The checks below are a **test plan**, not a record of tests executed during this revision.

## Source-review findings versus bench evidence

| Claim or workflow | Supported by reviewed source | Separate bench evidence needed |
| --- | --- | --- |
| UI/backend separation | Local Unix-domain control client; independently supervised Python server and kiosk UI | Terminate/relaunch the UI while a test client continues reading mapped data. |
| Mapped Modbus reads/writes and exceptions | Server/register model and a repository functional self-test | Run on an isolated simulator against synthetic/approved test addresses; compare responses and exception codes. |
| Scenario versus accepted client-write ownership | Scenario engine, write-commit observer, ownership assignment | Start a periodic scenario, write a test tag externally, verify it stays client-controlled; separately test rejected writes and scenario reapplication. |
| Communication Block/Restore | Listener stop/start reuses the existing datastore context | Confirm loss/recovery of external connections while local control remains available; compare register values before and after. |
| Live traffic and packet observation | Bounded decoded transaction rings and passive packet capture implementation | Generate known requests and independently capture test-bench traffic; check timestamps, client identity, drops, and capture permissions. |
| History and exports | Non-blocking event queue, rotated searchable event history, and live-buffer export paths | Generate writes, errors, and read-heavy traffic; inspect retained record types, export limits, rotation, free-space behavior, and dropped-event counters. |
| HMI client behavior | Simulator cannot observe remote application state from traffic alone | Observe the specific HMI UI, retry policy, alarms, reconnection behavior, and command handling during scripted network tests. |
| Restart persistence | Configuration files are stored separately from volatile live runtime state | Test definition persistence, service restart and appliance reboot; do not assume in-memory register values survive backend process restart. |

## Proposed controlled test sequence

1. Establish an **isolated** bench with approved synthetic tags and a trusted Modbus test client. Record the starting software build and configuration without placing private values into a public report.
2. Exercise mapped coil and holding-register reads/writes, multiword ordering, and undefined-address exceptions. Compare the client response with a passive trace where capture is enabled.
3. Apply a scenario, verify periodic value changes, set a manual value, and confirm that the saved scenario file remains unchanged.
4. With a periodic scenario active, perform a **successful external write** to a writable test tag and confirm takeover. Verify that a rejected/failed write does not take ownership. Start a new scenario and record the behavior after reconfiguration.
5. Record a known value, **Block PLC Comms**, verify the external listener is unavailable while the local UI/control path works, **Restore PLC Comms**, and verify the existing value and ownership state remain in effect.
6. Produce a small known mixture of reads, accepted writes, failed writes, and address exceptions. Compare Live Traffic, Packets, filtered History, and CSV/JSON/PCAP *live-buffer* export, accounting for capture/queue drops and bounded-retention limits.
7. Close or relaunch the GUI while keeping the backend running. Separately test service restart and a full appliance reboot; document the expected differences between persistent JSON definitions and volatile runtime datastore state.
8. For a real HMI test, observe its application behavior directly. Never infer its screen, alarm, or workflow state solely from Modbus packets.

## Acceptance record template

For each bench run, capture: test date, approved non-sensitive build identifier, preconditions, expected result, actual observation, outcome, and a link to **privately retained** evidence. Mark steps **Not run** rather than claiming a pass from reading source or old documentation. Before any public publication, review screenshots, logs, packet captures, addresses, tag names, endpoints, hostnames, and identifiable equipment details.

## Scope and known limitations

- This case study is a snapshot, not a warranty of all deployments or a substitute for a site acceptance test.
- A stopped TCP listener represents one failure mode; it does not simulate loss of device power or every network fault.
- Packet capture may be unavailable or lossy. No packet view guarantees perfect capture or complete application-state knowledge.
- Live transaction/packet buffers are bounded, and the visible rows are smaller than those buffers; event history intentionally omits routine reads and raw packets.
- Retained event history is subject to queue capacity, file rotation, available storage, and permission/error conditions. Read/export counters and retention claims should be verified in the actual lab configuration.
- Sample tag names in this repository are **synthetic placeholders** and are not guaranteed to exist in the private deployment's catalog.
