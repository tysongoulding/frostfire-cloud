# Frostfire Cloud Agent Directives

Autonomous cloud control plane, microVM virtualization infrastructure, and swarm orchestrator.

This document serves as the constitutional operational specification and governance directive for all autonomous coding agents, LLM runtimes, and engineering contributors operating within the `frostfire-cloud` repository.

---

## 1. System Identity & Mission

`frostfire-cloud` provides the private cloud control plane, secure microVM virtualization infrastructure, edge ingress gateway, and multi-agent swarm orchestrator for the Frostfire platform.

The system is designed to instantiate, supervise, and coordinate isolated, ephemeral cloud sandboxes where autonomous AI agents (coding, browser automation, human-in-the-loop, and computer-use models) can execute commands, compile software, interact with graphical applications, and collaborate safely at scale.

---

## 2. Mandatory Verification Gates

Execute after every modification before declaring work complete or pushing code:

1. **Unit & Integration Suite**:
   ```bash
   cargo test --workspace
   ```
   *Requirement*: Must pass all tests across all workspace crates with zero failures and zero warnings.

2. **Linter & Static Analysis**:
   ```bash
   cargo clippy --workspace -- -D warnings
   ```
   *Requirement*: Must pass with zero warnings allowed (`-D warnings`).

3. **No Unchecked Warnings or Dead Code**:
   Ensure unused imports, unhandled `Result` variants, and unformatted code are resolved prior to staging changes.

---

## 3. Strict Architectural Invariants

All agents and contributors must strictly enforce the following security and networking invariants:

### A. Outbound-Only Ingress (`OpenTunnel`)
* **Directive**: Guest microVMs and edge daemons must never bind or expose public listening ports directly to the Internet.
* **Mechanism**: Ingress is mediated exclusively through outbound reverse-streaming gRPC channels (`AgentTunnelService.OpenTunnel`) over TLS 1.3 terminating at the Cloud Gateway (`cloud/gateway`).
* **Multiplexing**: Interactive PTY sessions, file diff synchronizations, browser events, and control messages are multiplexed inside encrypted bidirectional frames over the reverse tunnel.

### B. MicroVM Isolation & Network Boundaries
* **Directive**: MicroVM instances run on isolated bridge networks (`172.16.x.0/24` or `172.30.0.0/24`). Never bridge unauthenticated guest networks to the public internet or the host's private management network.
* **Firewalling**: The hypervisor host enforces strict iptables/nftables egress policies. DNS requests, package management, and outbound calls must route through monitored proxy interfaces or strictly controlled tap devices.
* **Hypervisor Sandboxing**: Root filesystems run in paravirtualized VirtIO containers with copy-on-write `overlayfs`, ensuring untrusted agent execution leaves zero persistent footprint on the host.

### C. Tenant Authorization & Constant-Time Verification
* **Directive**: All display routes, terminal sockets, and window-mux sessions must authenticate tenant identity.
* **Security Primitive**: Inbound requests must supply the `x-sand-window-owner` token. Comparisons against authorized session tokens must use constant-time comparison (e.g., `timingSafeEqual` in Node.js, `subtle::ConstantTimeEq` in Rust) to eliminate timing side-channel attacks against session hijacking.

### D. Zero Secrets in Git
* **Directive**: Never commit AWS credentials, GCP service account keys, private SSH keys, TLS certificates, or external LLM API tokens into version control.
* **Credential Brokering**: All credentials and tokens must be stored in encrypted local keystores or passed at runtime via environment variables managed by `crates/frostfire-security`.

---

## 4. MicroVM Hypervisor & Cgroup Constraints

When managing or modifying the microVM virtualization layer (`cloud/microvm`):

1. **Kernel Configuration**:
   * Use monolithic paravirtualized kernels (`Linux 6.12+`) with static VirtIO drivers (`virtio_net`, `virtio_blk`, `virtio_pci`, `overlayfs`) and `nomodule` enabled.
   * Serial console (`console=ttyS0 earlyprintk=ttyS0`) must be used for fast boot times (< 250ms).
   * Fast panic failover (`reboot=k panic=1`) must be configured so fatal crashes hand control back to the host supervisor immediately.

2. **Cgroup v2 Hierarchies**:
   * Schedule processes into two strict cgroup v2 slices:
     * `interactive` (`/sys/fs/cgroup/interactive`): High priority for window managers (`xfwm4`), Xvfb framebuffers, compositors (`picom`), and VNC daemons (`x11vnc`, `websockify`).
     * `agent` (`/sys/fs/cgroup/agent`): Lower priority execution slice for compilers, code execution, test suites, and sub-processes.
   * Swap is disabled (`SWAP_SIZE_MB=0`) to eliminate disk thrashing under heavy compiler or LLM memory pressure.

---

## 5. Swarm Orchestration & Multi-Agent Protocol

When orchestrating or interacting with multi-agent swarms (`services/swarm-orchestrator`, `crates/frostfire-core`):

1. **Context-Driven Dynamic Teams**:
   * Do not use static agent rosters. Spawn specialized agent pairs (Architect + Staff Engineer) dynamically matched to the problem domain (e.g., Context Mode Architect, QA Engineer, DX Engineer, Platform Architects).
   * Follow the orchestration loop: `CLASSIFY` $\to$ `RECRUIT` $\to$ `DISPATCH` $\to$ `MONITOR` $\to$ `PING-PONG` $\to$ `VALIDATE` $\to$ `SHIP`.

2. **Blackboard Coordination Pattern**:
   * Inter-agent communication and state synchronizations must be published to the append-only Blackboard event store (`frostfire-core::blackboard`).
   * Never couple agents through undocumented peer-to-peer side-channels; all DAG state transitions must be serialized and traceable.

3. **Orchestrator Role Boundaries**:
   * Lead orchestrators focus on recruiting, dispatching, and validating tasks across subagents. Subagents perform deep codebase analysis, code modifications, and test verifications.

---

## 6. Git & Pull Request Protocol

* **GitHub Tooling**: Use the official GitHub CLI (`gh`) for PR generation, issue management, and workflow triggers.
* **Commit Conventions**: Follow conventional commits (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`).
* **PR Verification**: Every PR must state what was tested, verify that `cargo test --workspace` passed, and ensure compliance with all 4 architectural invariants.
