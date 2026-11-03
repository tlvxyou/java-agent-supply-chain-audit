# Security & Reverse Engineering Audit: Untrusted Java Agent (loader.jar)

[![PDF Report](https://img.shields.io/badge/View_Full_Report-PDF-red?style=for-the-badge&logo=adobe-acrobat-reader)](./Report.pdf)

> **Auditor Signature:** `APHP`
> **Target Subject:** `loader.jar` (Java Bytecode Agent & License Patcher)
> **Source Repository:** `xiv3r/Burpsuite-Professional`
> **SHA-256 Digest:** `dcdf28acf360554a5a98d78f403c96ccea500be24b27d02b020e142820637c0a`
> **Analysis Environment:** Kali Linux (Isolated Hypervisor Sandbox — Layer-3 Network Isolated, No WAN Gateway)

---

## 📄 Full Analysis Report

The comprehensive 6-page reverse-engineering report containing all static bytecode audits, dynamic kernel tracing (`strace`), packet captures (`tcpdump`), and execution screenshots is available in the repository:

👉 **[Click here to view / download the complete `Report.pdf`](./Report.pdf)**

---

## Key Findings Summary

* **Verdict:** **Benign / Non-Malicious** (No Trojans, RATs, Infostealers, or persistence mechanisms detected).
* **Classification:** HackTool / License Key Generator / Java Bytecode Agent Patcher.
* **Primary Finding Method:** Static JADX bytecode decompilation of all 4 core classes (`Loader.class`, `Filter.class`, `KeygenForm.class`, and supporting utilities) — confirmed absence of trojan droppers, hidden shell execution, and malicious C2 logic.
* **Dynamic Supplementation:** `strace` process/filesystem syscall auditing inside an air-gapped VM; no outbound `connect()` syscalls to public IPs observed during standalone GUI execution.
* **Filesystem Activity:** Only wrote local UI preference states (`.config.ini`).

---

## Methodology

This audit combined three complementary analysis layers:

1. **Static Bytecode Decompilation (Primary)** — JADX was used to decompile and inspect all 4 core class files (`Loader.class`, `Filter.class`, `KeygenForm.class`, and supporting utilities), audit every ASM injection point, and verify license validation hooks contain no dropper, shell-execution, or C2 logic.
2. **Dynamic Kernel Tracing (Supplementary)** — `strace` monitored process/filesystem syscalls during a supervised standalone execution run (`java -jar loader.jar`) inside an air-gapped VM (no WAN gateway), confirming no unexpected child-process spawning or sensitive file writes.
3. **Network Packet Capture (Environmental Baseline)** — `tcpdump` established a network baseline in the isolated environment. Note: the air-gapped VM had no WAN gateway, so this confirms no outbound connection *attempts* reached the network stack — the primary C2/exfiltration determination derives from the JADX static audit.

---

## 🔬 Methodology & Sandbox Constraints (Why the VM is Air-Gapped)

Two design decisions in the analysis environment are documented here to pre-empt technical questions:

### 1. Intentional Layer-3 Network Isolation (`ICMP Host Unreachable`)

Dynamic execution (`strace` + `tcpdump`) was performed inside a **network-isolated VM with no live WAN gateway** — a deliberate precaution to prevent any accidental C2 check-in during the execution of untrusted bytecode. Network safety was verified by two independent signals:

- **`strace` verification:** Zero `connect()` socket syscalls to public IP addresses were observed in the full syscall trace. All socket-related calls were limited to loopback (`127.0.0.1`) for JVM internal IPC.
- **`tcpdump` verification:** Zero DNS `A` or `AAAA` lookup queries were captured on `UDP/53` prior to any routing failure. The absence of DNS resolution attempts — not merely failed connections — confirms there were no outbound hostname lookups attempted by the process.

The `ICMP Host Unreachable` responses visible in the packet capture are artifacts of the isolated routing table, not of the subject binary attempting connections.

### 2. Standalone GUI Execution vs. `-javaagent` Instrumentation

The dynamic runtime tracing (`strace`, `tcpdump`) targeted the **standalone `KeygenForm` invocation** (`java -jar loader.jar`) — the mode in which the loader presents its GUI and generates license keys — rather than attaching the untrusted agent to a live Burp Suite process via `-javaagent:loader.jar`.

The `-javaagent` instrumentation pathway — specifically the `Loader.class premain()` entry point and the `Filter.class` HTTP traffic mocking hooks — was audited **statically via full JADX bytecode decompilation**. This approach:

- Avoids running untrusted ASM hooks inside a live JVM process
- Provides complete visibility into every bytecode instruction without sandbox escape risk
- Produces reproducible, screenshot-backed static evidence rather than runtime artifacts

Both execution pathways are documented in the full PDF report.

---

### Verification Matrix

| Vector | Observed Behavior | Malware Pattern | Status |
| :--- | :--- | :--- | :--- |
| **Network Traffic** | No outbound `connect()` syscalls observed in strace; air-gapped VM had no WAN gateway (C2 absence confirmed primarily via JADX static audit) | C2 beaconing, DNS tunneling | **PASS (Clean)** |
| **Process Execution** | Java version verification only | Spawning hidden interactive shells | **PASS (Clean)** |
| **File Operations** | Local `.config.ini` write only | Dropping stagers to `/tmp` | **PASS (Clean)** |
| **Bytecode Safety** | ASM hooks targeted at licensing logic in `Loader.class` / `Filter.class` | Encrypted classloaders, memory injection | **PASS (Clean)** |
