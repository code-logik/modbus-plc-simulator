# Architecture Notes

## Design Goals

The simulator was designed around five goals:

1. Provide a deterministic Modbus/TCP endpoint for repeatable testing.
2. Separate register definitions from simulation behavior.
3. Keep local operator activity separate from external protocol observability.
4. Support reusable scenario definitions without code changes.
5. Operate as a self-contained Linux test appliance.

## Logical Components

```text
Synthetic Definitions
        |
        v
Register Model <---- Scenario Definitions
        |                    |
        +------> Simulation Engine
                       |
                       v
                  Register Store
                   /          \
                  /            \
       Local Control         Modbus/TCP
          Channel              Server
             |                   |
             v                   v
       Operator UI         External Client
```

### Register Model

Represents simulated coils and holding registers using generic/synthetic definitions in the public case study.

### Simulation Engine

Applies configured behaviors such as static values, counters, ramps, toggles, randomized values, and periodic functions.

### Local Control Channel

Allows the local interface to inspect and modify simulator state without requiring an additional externally reachable management service.

### Modbus/TCP Server

Exposes the simulated datastore to external test clients using standard Modbus/TCP behavior.

### Transaction Observability

External Modbus transactions are logged separately from internal application access so test evidence reflects actual network activity.

## Security and Publication Boundary

The original system was developed for a private professional environment. This public architecture intentionally omits exact interface names, addresses, routes, production register definitions, device models, organization names, deployment artifacts, and proprietary implementation details.
