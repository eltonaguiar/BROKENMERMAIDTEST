<!-- saved: C:/Users/zerou/SERVICENOW_DISCOVERY_TECHNICAL_REFERENCE.md | all cited URLs verified HTTP 200 on 2026-11-13 -->

# ServiceNow Discovery — Technical Reference

ServiceNow Discovery is the agentless (and optionally agent-assisted) data-collection subsystem of IT Operations Management (ITOM) that finds devices and applications on a network, classifies them, extracts their attributes and relationships, and writes them into the Configuration Management Database (CMDB) through the Identification and Reconciliation Engine (IRE).

- Platform scope: `com.snc.discovery`, ITOM suite
- Runtime location: orchestration on the instance; execution on the MID Server
- Primary write target: CMDB (`cmdb_ci` and descendants)
- Primary write path: IRE (never a direct table upsert)
- Source documentation hub: [Exploring Discovery](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_GetStartedWithDiscovery.html)

---

## 1. What Discovery is, and the problem it solves

### 1.1 The problem

A CMDB is only useful if it is accurate, complete and current. Manual CI entry fails at scale for three reasons: coverage (thousands of hosts, network gear, cloud resources), drift (servers are patched, renamed, re-IPed, retired daily) and trust (an audit trail that says "a human typed this in 2019" is not defensible).

Discovery solves this by making the CMDB a *continuously reconciled derivative* of observed infrastructure state rather than a hand-maintained spreadsheet. It produces:

1. **Inventory** — CIs of the right class, in the right table (`cmdb_ci_server`, `cmdb_ci_linux_server`, `cmdb_ci_win_server`, `cmdb_ci_netgear`, `cmdb_ci_app_server`, database, load balancer, hypervisor, VM, and so on).
2. **Attributes** — hostname, FQDN, IP, MAC, OS version, CPU, RAM, disks, serial number, installed software, running processes.
3. **Relationships** — `Runs on::Runs` (app → host), `Hosted on::Hosts` (VM → hypervisor), `Connects to::`, `Depends on::`, containment.
4. **Drift detection** — attribute changes surface as updates with source attribution, which feeds Change, Incident, Event Management and audit.

### 1.2 Two discovery styles (this distinction drives everything downstream)

| Style | Purpose | Produces | Depends on |
|---|---|---|---|
| **Horizontal discovery** (Discovery) | Sweep a network range, find and classify everything, populate CMDB | CIs + direct relationships (`Runs on::Runs`) | MID Server, credentials |
| **Top-down discovery** (Service Mapping) | Start from a known entry point and trace application dependencies to build a service map | Application service instances + deep connection paths | Horizontal discovery must already work |

Horizontal discovery is the subject of this document; Service Mapping is covered in §7 as a consumer.

> Source: [Exploring Discovery](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_GetStartedWithDiscovery.html), [Exploring Service Mapping](https://www.servicenow.com/docs/r/it-operations-management/service-mapping/service-mapping-get-started.html)

### 1.3 What Discovery is *not*

- It is **not** a monitoring system — it produces no thresholds or alerts (that is Event Management, which consumes Discovery's CIs).
- It is **not** a CMDB integrity layer — it feeds IRE; IRE is what decides insert vs update. Discovery can create duplicates if identification rules are wrong (§5, §8).
- It is **not** strictly agentless-forever — Cloud Discovery, Agent Client Collector and Service Graph Connectors are deliberate alternatives for environments where network scanning cannot reach.

---

## 2. Components and how they interact

### 2.1 MID Server (Management, Instrumentation and Discovery)

The MID Server is a Java process installed inside the customer network. It is the *only* component that touches target devices. It:

- Initiates **outbound** connections to the instance over HTTPS/WSS (no inbound firewall hole needed).
- Polls the ECC queue for work assigned to it or to its cluster.
- Executes probes, patterns, PowerShell/WMI/SSH/SNMP calls and API calls.
- Returns raw payloads to the ECC queue.
- Caches credentials and an IP exclusion list locally.

Operational facts that matter in production:

- A MID Server must be **Up** and **Validated** against the instance. Validation proves the MID's user can authenticate and that the MID can read/write the ECC queue.
- MID Servers are grouped into **clusters**; a schedule or credential can target a specific MID or "All MID Servers", and an *application* assignment (Discovery, Service Mapping, Orchestration, Cloud) controls which work types a MID accepts.
- Auto-assignment is desirable for capacity; **pinning** a schedule to a MID is required when the MID is the only one with network reachability (DMZ, segmented VLAN, cloud VPC).
- Restarting or upgrading a MID clears its credential/exclusion caches — behaviour changes subtly right after a restart.

> Source: [Resolving MID Server issues](https://www.servicenow.com/docs/r/servicenow-platform/mid-server/r_MIDServerTroubleshooting.html)

### 2.2 ECC Queue (External Communication Channel)

The ECC queue (table `ecc_queue`) is the asynchronous message bus between instance and MID Server. Every unit of discovery work is a row, and its lifecycle is the diagnostic spine of the whole product:

- **Output** records: instance → MID ("run this probe against 10.0.0.5").
- **Input** records: MID → instance ("here is the payload / here is the error").
- States progress `ready` → `processing` → `processed` / `error`.

Reading ECC records in order is the single most reliable way to locate where a discovery run broke: a stuck **output** record points at the MID (down, unassigned, unreachable); a stuck **input** record points at a sensor (payload parse or script error).

> Source: [Horizontal discovery process flow with probes and sensors](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoProcessFlows.html), [Advanced Discovery configuration](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryExtendedCapabilities.html)

### 2.3 Discovery Schedule, Ranges and Targets

A schedule (`discovery_schedule`) is the entry point. It binds together:

- **What to scan**: IP ranges, individual IPs, named CI lists, or serverless/cloud targets (each row is a range item, `discovery_range_item`).
- **Which MID**: specific MID Server or cluster/all.
- **When**: run type (once, on demand, periodically, daily/weekly/monthly), start time, time zone, repeat interval.
- **Behaviour switches**: use SNMP, use Nmap fallback, MID selection, and the discovery *behaviour* that governs which probes Shazzam fires.

A schedule run creates a **Discovery Status** record (`discovery_status`) that aggregates logs, ECC correlation and per-device outcomes for that run.

> Source: [Create a discovery schedule](https://www.servicenow.com/docs/r/it-operations-management/discovery/t_CreateADiscoverySchedule.html), [Running discoveries](https://www.servicenow.com/docs/r/it-operations-management/discovery/running-discoveries.html)

### 2.4 Shazzam and port probes (scanning layer)

**Shazzam** is the scan probe. It sweeps the target range, tests a configured set of TCP/UDP ports, and records which ones respond. Historically it wraps `nmap`; discovery **behaviours** decide which follow-on port probes Shazzam launches and from which MID Servers.

Port response is a *class hypothesis*, not a fact: 135 → Windows, 22 → UNIX/Linux, 161 → SNMP-capable device, 443 → web/app tier. Shazzam's output is what makes the classification step possible.

> Source: [Exploring Discovery](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_GetStartedWithDiscovery.html), [Advanced Discovery configuration](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryExtendedCapabilities.html)

### 2.5 Probes

A probe (`discovery_probe`) is a named unit of collection executed by the MID Server. Examples: `Shazzam`, `WMIRunner`, `PowerShell`, `SSHCommand`, `SNMP`, `HorizontalPatternProbe`, VMware/vCenter probes.

Probes are referenced by name from classifiers via a `trigger_probes` list. A classifier pointing at a probe that does not exist, or a probe whose handler is unavailable on the MID, produces the "probe not found" class of error.

> Source: [Probes and sensors](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryProbesAndSensors.html)

### 2.6 Sensors

A sensor (`discovery_sensor`) is the server-side script that consumes a probe's returned payload and turns it into CMDB operations. Sensors are where:

- Payload XML is parsed (malformed payloads fail here, not on the MID).
- Attribute extraction and normalisation happens.
- The call into IRE is made.

Sensor failures are the classic "input record stuck in processing/error" symptom.

> Source: [Probes and sensors](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryProbesAndSensors.html), [Discovery error messages](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryErrorMessages.html)

### 2.7 Classifiers

A classifier (historically the table `discovery_classyfier` — the platform's long-standing misspelling) maps *observed signals* to a *CI class* and a *trigger probe list*. It is the pivot of the run:

- It holds the conditions (port response, OS string, process, IP range) that determine the class.
- It names the identification trigger probe and the exploration trigger probe(s).
- It can instead name `HorizontalPatternProbe`, which launches a pattern rather than a chain of probes.

Three classifier kinds exist: **device** classification, **process** classification (runs during exploration, creates child app CIs with `Runs on::Runs`), and **IP address** classification.

> Source: [Discovery classifiers](https://www.servicenow.com/docs/r/it-operations-management/discovery/discovery-classifiers.html)

### 2.8 Patterns (modern collection)

Patterns are declarative, versioned descriptions of *how* to collect a given CI type: which commands/queries to run, which steps depend on which, how to parse, and which CI class and attributes to emit. They replace long chains of imperative probes for most modern targets (Linux/Windows/average app tiers).

Pattern structure that matters operationally:

- **Identification sections** — determine whether the target matches this CI type, and produce identifying attributes.
- **Connectivity sections** — trace outbound connections for Service Mapping.
- Patterns run on the MID (`HorizontalPatternProbe` / pattern engine), emit an IRE payload, and return via ECC.
- **Pattern Designer / Pattern Designer (legacy)** versions patterns; instance and MID versions must be compatible.

Patterns are preferred over legacy probes: they are portable across protocols (SSH, WMI, REST, SNMP), versioned, and testable. An environment can run both simultaneously; the classifier decides which path a given device takes.

> Source: [Create CI types for Service Mapping and Discovery](https://www.servicenow.com/docs/r/it-operations-management/discovery-and-service-mapping-patterns/t_CreateCITypeForSM.html), [Pattern-based discovery in Service Mapping](https://www.servicenow.com/docs/r/zurich/it-operations-management/service-mapping/pattern-based-discovery.html)

### 2.9 Credentials

Credential records live in `discovery_credentials` (plus SSH key stores, SNMP communities/v3, VMware, cloud API credentials). The resolution machinery is more important than the records:

- **Credential type** must match the target protocol. A Windows credential on a Linux host is not an error — it just is not tried.
- **"Applies to"** determines whether a MID can even see the credential. A credential scoped to a different MID is invisible, which reads as "no credential found" rather than "wrong password".
- **Order** governs which credential is tried first; on failure the next is tried, so a long list of wrong credentials can lock an account out.
- **Affinity** (IP/range/CI binding) restricts a credential to a subnet or device class — essential in multi-tenant or multi-domain environments, and the usual fix for a domain credential being sent to the wrong forest.
- **Exclusion list**: after *every* credential fails for an IP, the MID caches that IP as excluded for about five minutes (cleared on MID restart or credential change). Re-running discovery immediately after fixing a credential can therefore still fail for up to five minutes.
- **Credential debug**: probes that support it emit a `<credentials_debug>` block into the ECC input payload listing credential types, tags, affinities, and each credential tried in order.

> Source: [Credentials troubleshooting](https://www.servicenow.com/docs/r/platform-security/connections-and-credentials/r_CredentialTroubleshooting.html)

### 2.10 Identification and Reconciliation Engine (IRE)

IRE is the *only* sanctioned write path into the CMDB. Discovery does not insert rows; it submits a payload through the Identification Engine API, and IRE decides what happens.

IRE is governed by **four rule types**, not one:

| Rule type | Decides | Insert or update |
|---|---|---|
| **Identification rules** (`cmdb_identifier`) | Whether a CI already exists — one rule per class, ordered identifier entries | Insert vs update |
| **Reconciliation rules** (static `cmdb_reconciliation_definition`, dynamic `cmdb_dynamic_reconciliation_definition`) | Which source may write which attribute of an existing CI | Update only |
| **IRE data source rules** (`cmdb_ire_data_source_rule`) | Whether a source may create CIs of a class at all | Insert gate |
| **Data refresh rules** (`cmdb_datasource_staleness`) | Whether a CI is stale enough for a *lower*-priority source to take over | Update only, static mode only |

Two task types are generated rather than applied: **de-duplication tasks** and **reclassification tasks**. Full treatment is in §5.

IRE is shared by Discovery, Service Mapping, Service Graph Connectors, import sets, manual forms and integrations. All data sources call the Identification Engine API rather than writing to CMDB tables directly. Understanding IRE is non-optional for anyone operating Discovery, because **most "Discovery created duplicates" complaints are identification-rule defects, not Discovery defects.**

> Source: [Identification and Reconciliation Engine (IRE)](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/ire.html), [Components and process of identification and reconciliation](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/c_CompsandProcessIDandReconcil.html)

### 2.11 CMDB and class hierarchy

CIs land in `cmdb_ci` and its descendants. Discovery's target class is determined by classification, and the class hierarchy (`cmdb_ci_computer` → `cmdb_ci_server` → `cmdb_ci_linux_server` / `cmdb_ci_win_server`, etc.) matters because **children inherit identifiers and many attributes from parents**. Configure an identifier on the parent and every child class inherits it — usually helpful, occasionally wrong (e.g. a class whose identity should be `serial_number` inheriting a `name`-based identifier from its parent).

> Source: [Create CI types for Service Mapping and Discovery](https://www.servicenow.com/docs/r/it-operations-management/discovery-and-service-mapping-patterns/t_CreateCITypeForSM.html)

### 2.12 Supporting subsystems

| Component | Role |
|---|---|
| Discovery Status / Discovery Log | Per-run aggregation; the first place to look |
| Discovery Behaviours | Which probes Shazzam launches, and from which MIDs |
| Application Dependency Mapping (ADM) | Detects TCP connections to build `Connects to` relationships |
| Nmap fallback | Collects basic CI data when authentication fails |
| Process filtering | Suppresses noise processes before they reach the instance |
| Event Framework throttling | `discovery.use.event.processing` — queue-based prioritisation instead of synchronous job processing |
| Agent Client Collector (ACC) | Agent-based collection where scanning cannot reach (DMZ, endpoints, domain controllers) |
| Service Graph Connectors | Third-party/cloud data into CMDB via IRE |

> Source: [Advanced Discovery configuration](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryExtendedCapabilities.html), [Discovery properties](https://www.servicenow.com/docs/r/it-operations-management/discovery/r_DiscoveryProperties.html)

### 2.13 Interaction map (who calls whom)

```
Scheduler
   └─> discovery_schedule (range + MID + run type)
         └─> discovery_status created
               └─> Shazzam output record ──> ecc_queue ──> MID Server
                                                              └─> nmap scan of range
               <── ecc_queue input (open ports)
         └─> Behaviour + port probe → device hypothesis
               └─> Classification probe (output → MID → input)
                     └─> CI class determined
                           └─> Classifier selected (by class + conditions)
                                 └─> IDENTIFICATION trigger probe or HorizontalPatternProbe
                                       └─> payload → Sensor (or pattern engine)
                                             └─> IRE payload
                                                   ├─ identifier match? → UPDATE
                                                   └─ no match → INSERT
                                 └─> EXPLORATION trigger probe(s)
                                       └─> attributes, software, processes, relationships
                                             └─> IRE (attributes + relations)
         └─> discovery_status → processed; logs written
```

---

## 3. Complete data flow: first contact → documented CI

This is the order of operations. Each phase depends on the previous one completing; a failure in phase *n* prevents *n+1* entirely (which is why "no CIs found" almost never has its root cause at the CMDB layer).

### Phase 0 — Trigger and fan-out

1. The scheduler fires the `discovery_schedule` at its configured time (or an operator runs *Discover now* / *Quick Discovery*, or a CI-scoped discovery is launched from a CI record).
2. A `discovery_status` record is created for the run.
3. The schedule's range items are expanded into individual IP targets.

**Dependency:** the schedule must be `Active`; the MID must be Up and Validated; the range must not be exhausted/empty.

### Phase 1 — Scanning (Shazzam)

4. Discovery writes a Shazzam **output** record into the ECC queue for each target (one scan job per range, batched internally).
5. A MID Server in the schedule's cluster polls the ECC queue, claims the job, and runs the scan.
6. The MID writes an **input** record containing the open/closed port map.
7. A port **sensor** records the responsive ports against the IP.

**Dependency:** MID→target network reachability. No reachability means Shazzam returns nothing and the device simply never appears — the single most common "Discovery is not working" cause that is *not* a Discovery defect.

### Phase 2 — Classification

8. Discovery consults **discovery behaviours** and the port probes to hypothesise a device type from the port fingerprint (135 → Windows, 22 → UNIX/Linux, 161 → SNMP device, and so on).
9. Discovery writes a **classification probe** output record (e.g. `WMIRunner` for Windows, `SSHCommand` for Linux, SNMP walk for network gear).
10. The MID runs it and returns OS/device details.
11. **Exactly one classification probe runs per device.** Its result fixes the CI class.

**Dependency:** a working credential for the protocol. Without one, classification fails and the device is either skipped or picked up only by Nmap fallback (§8).

### Phase 3 — Classifier selection

12. Discovery matches the resulting class and signals against **CI classifier** criteria.
13. The winning classifier supplies the **identification trigger probes** and **exploration trigger probes** — or `HorizontalPatternProbe` to run a pattern instead.

**Dependency:** classifier order and conditions. A too-broad classifier earlier in the order will shadow a correct one later.

### Phase 4 — Identification

14. Discovery writes the identification trigger probe (or pattern identification section) as an output record.
15. The MID executes it and returns identification data: serial number, hostname, FQDN, MAC, BIOS UUID, instance identifiers.
16. **Sensors** (or the pattern engine) parse the payload.
17. An **IRE identification payload** is submitted.

**Dependency:** IRE configuration for that class.

### Phase 5 — IRE insert/update decision

18. IRE evaluates the class's **identification rules** (identifier entries, in precedence order) against existing CIs of that class (and, depending on rule configuration, related classes).
19. **Match found** → the CI is *updated*: reconciled attributes are written, changed attributes get change history, and the discovery source is recorded.
20. **No match** → a *new CI* is inserted with `discovery_source` set to the discovering source, and a `correlation_id` allocated.

**This is the insert-vs-update boundary.** Nothing before this step decides it.

### Phase 6 — Exploration

21. Discovery writes the classifier's exploration trigger probes as output records.
22. The MID collects deeper data: disks, NICs, drivers, memory, installed software, running processes, patch level.
23. Sensors/patterns submit further IRE payloads (attributes and relationships).

**Dependency:** phase 4/5 completed — exploration is scoped to an identified CI.

### Phase 7 — Process and application classification

24. An active-processes probe runs.
25. Results are compared against **process classification conditions**. A match may trigger a **process handler script** that normalises parameters so the same application is not re-created under a slightly different name.
26. The application CI is created or updated, and a `Runs on::Runs` relationship is created to the host.
27. Application dependency mapping may determine `Connects to` relationships.

### Phase 8 — Relationships and hierarchy

28. Relationships are submitted through IRE alongside attributes. Containment/hosting relationships link VM → hypervisor, server → cluster, app → server.
29. Because relationships are CI-class-sensitive, an incorrectly classified parent can silently prevent a relationship from being created (the relation type is rejected for that class pair).

### Phase 9 — Completion

30. The final ECC inputs are processed; `discovery_status` moves to processed/complete.
31. Logs are written per device; failures are attached to the status record.

### Phase 10 — Downstream propagation

32. Change Management starts referencing the new/updated CIs; Event Management can bind alerts to them; Service Mapping can use them as entry points; Dependency Views renders them; CMDB health/CSDM compliance reports pick them up.

---

## 4. Configuration locations, triggers, schedules, dependencies

### 4.1 Where configuration lives

| Concern | Location |
|---|---|
| Schedules, ranges, targets | Discovery → Discovery Schedules |
| MID Server definitions, clusters, app assignment | MID Server → Servers / Clusters |
| Credentials and credential affinities | Discovery → Credentials (and Platform Security → Connections & Credentials) |
| Classifiers | Discovery → Classifiers |
| Probes | Discovery → Probes |
| Sensors | Discovery → Sensors |
| Behaviours | Discovery → Behaviours |
| Patterns | Pattern Designer / Discovery & Service Mapping Patterns |
| Identification rules | CMDB → Identification rules (per CI class) |
| Reconciliation rules | CMDB → Reconciliation rules |
| Discovery properties | System Properties, `mid.*` and `discovery.*` namespaces |
| Discovery behaviours / ADM / Nmap | Advanced Discovery configuration |
| Status and logs | Discovery → Discovery Status → Discovery Log, ECC Queue |
| CMDB data | `cmdb_ci*` tables, CMDB Workspace |

### 4.2 Triggers

- **Scheduled** — the normal path; run type and repeat interval on the schedule.
- **On demand** — *Discover now* on a schedule; useful after a credential fix.
- **Quick Discovery** — ad-hoc, single-target, bypasses the schedule.
- **CI-scoped** — launched from a CI record to refresh one host.
- **Event-driven (cloud)** — lifecycle events from AWS/Azure/GCP refresh resources near-real-time; complements, and does not replace, scheduled authoritative collection.
- **Agent-based** — Agent Client Collector pushes data on its own cadence.

### 4.3 Schedule design constraints

- Ranges should not overlap between schedules. Overlapping ranges mean two runs race to write the same CI; IRE will serialise the writes but you get churn, duplicate-insert races on first discovery, and confusing change history.
- Schedule windows should not overlap the MID's other workload (Service Mapping, Orchestration, cloud connectors). A saturated MID looks exactly like a broken MID.
- Stagger large ranges. A 65,000-address /16 through one MID will not finish inside a maintenance window.
- Time zone is stored on the schedule; a schedule authored in one zone and reviewed in another is a recurring source of "it didn't run".

### 4.4 Dependency chain (must hold, in order)

```
1. MID Server installed → Up → Validated → assigned the Discovery application
2. Network path MID → target (TCP as required per protocol)
3. Credential exists → correct type → Applies-to includes this MID → ordered sensibly
4. Behaviour configured for the protocol so Shazzam launches the right port probes
5. Classifier exists for the class, with a valid trigger_probes / pattern reference
6. Pattern version compatible with MID version (pattern-based only)
7. Identification rule exists for the class (and, if desired, on the parent)
8. Reconciliation rule exists so multi-source attributes resolve predictably
9. Role: discovery_admin (or equivalent) on the user running/creating discovery
```

Break any link and the symptom appears at the *end* of the chain, not at the broken link.

> Source: [Discovery configuration](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryConfiguration.html), [Create a discovery schedule](https://www.servicenow.com/docs/r/it-operations-management/discovery/t_CreateADiscoverySchedule.html)

---

## 5. Insert vs update behaviour

### 5.1 The mechanism

Discovery never writes to `cmdb_ci*` directly. It submits an IRE payload; IRE resolves identity and applies the write. The decision is made by **identification rules** for the CI class:

- An **identification rule** belongs to a CI class and contains **identifier entries** — a set of attributes that together constitute identity (for example `serial_number`, or `name` + `os_domain`, or `object_id` for cloud resources).
- Entries are evaluated in order. The first matching set wins.
- **Match** → update the existing CI. **No match** → insert a new CI.

Consequences that surprise people:

- Two discovery sources (Discovery and a Service Graph Connector) that use different identifier sets for the same physical host **will produce two CIs**. Discovery is not "wrong"; the identification configuration is inconsistent.
- Adding an attribute to an identifier is a data-migration event, not a config tweak: existing CIs that no longer match will be duplicated on the next run.
- Identifiers are **inherited** down the class hierarchy. A parent-class identifier applies to all children unless overridden.

### 5.2 When Discovery inserts

- First discovery of a device: no CI matches → insert.
- A device that changed identity in a way that breaks the identifier (re-imaged with a new serial, replaced motherboard, renamed host where `name` is the identifier).
- A CI was manually deleted but the device still exists.
- Classification changed to a *different class* where no identifier match exists → insert into the new class (and the old CI may become orphaned rather than automatically merged).

### 5.3 When Discovery updates

- A CI matches on identifier → attributes reconciled.
- The device's class is unchanged.
- Only attributes the discovering probe/pattern actually collects are considered; attributes it does not collect are left alone (this is why hand-entered warranty or support-group data survives discovery).

### 5.4 The four rule types that govern the write

IRE does not have one "reconciliation" mechanism. It has **four distinct rule types**, each answering a different question. Treating them as one is the single most common conceptual error in Discovery documentation, including in earlier drafts of this document.

| # | Rule type | Question it answers | Table |
|---|---|---|---|
| 1 | **Identification rules** | Does this CI already exist? | `cmdb_identifier` + `cmdb_identifier_entry` |
| 2 | **Reconciliation rules** (static) | Which source may write this *field* on this table? | `cmdb_reconciliation_definition` |
| 2b | **Reconciliation rules** (dynamic) | Which source wins for this *attribute* on this class? | `cmdb_dynamic_reconciliation_definition` |
| 3 | **IRE data source rules** | May this source *insert* CIs of this class at all? | `cmdb_ire_data_source_rule` |
| 4 | **Data refresh rules** | May a *lower*-priority source take over a stale CI? | `cmdb_datasource_staleness` (+ `cmdb_datasource_last_update`) |

Plus two task types that IRE generates rather than applies: **de-duplication tasks** (`reconcile_duplicate_task`, `duplicate_audit_result`) and **reclassification tasks** (`reclassification_task`).

> Source: [Components installed with Identification and Reconciliation](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/r_IWIdentAndReconcil.html), [Components and process of identification and reconciliation](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/c_CompsandProcessIDandReconcil.html)

### 5.5 Identification rules — the insert/update decision

- A CI class is associated with **exactly one identification rule**. You do not stack rules on a class; you edit one rule and give it multiple **identifier entries**.
- `cmdb_identifier_entry` rows each carry a **priority**. IRE applies entries in priority order and the first entry that matches an existing CI wins.
- **Inclusion rules** narrow which CIs are eligible for identification at all — a CI excluded by inclusion criteria never participates in the match and therefore is always treated as new.
- **Dependent relationship rules** let IRE identify dependent CIs by walking the dependency structure of a service definition, which is how child CIs get attached to an already-identified parent rather than being re-created.

Because it is one rule per class with inherited entries, the practical consequences are:

- Adding an attribute to an identifier is a **data migration**, not a config tweak — existing CIs that no longer match will be re-inserted as duplicates on the next run.
- Children **inherit** identifier entries from parents. A parent's `name`-based entry can silently apply to a class that should have been matching on `serial_number`.
- Inconsistent identifiers between Discovery and a Service Graph Connector guarantee duplicate CIs, and merging them does not fix it — the next run re-duplicates.

### 5.6 Reconciliation rules — static vs dynamic

Reconciliation applies **only to updates**. If IRE inserts a new CI, reconciliation is not applied at all. This is explicit in the product documentation and it matters: a "reconciliation problem" can only exist for CIs that already matched.

**Static reconciliation** (`cmdb_reconciliation_definition`) is defined **at the table and field level**. For a given class and a given field, it declares which discovery sources are authorized to write. It is the traditional mechanism and it is the only mode that data refresh rules work with.

**Dynamic reconciliation** (`cmdb_dynamic_reconciliation_definition`) is defined **per attribute and per class**, and resolves the winner dynamically rather than from a static field map.

Critical interaction: **data refresh rules have no effect while dynamic reconciliation rules are in place.** If someone enables dynamic reconciliation to "clean up" precedence, they silently disable the staleness/failover behaviour that keeps lower-priority sources able to take over abandoned CIs. Symptom: a source that dies keeps owning its attributes forever.

### 5.7 IRE data source rules — the insert gate

`cmdb_ire_data_source_rule` answers a question neither identification nor reconciliation can: **should this source be allowed to create CIs of this class at all?**

Use case: you trust a source to *update* CIs that already exist, but you do not want it inventing new ones. Classic examples are a monitoring tool that fabricates placeholder CIs, or a partially-configured connector. The data source rule blocks the insert while leaving updates permitted.

This is not reconciliation. Reconciliation decides *whose value wins on an existing record*; a data source rule decides *whether a record may be created by that source*. Conflating them produces the "I restricted the source but it still created CIs" bug — because the restriction was placed in the wrong table.

### 5.8 Data refresh rules — staleness and failover

`cmdb_datasource_staleness` holds a **Data Source Staleness Definition** per data source: an **Effective Duration**. `cmdb_datasource_last_update` tracks which source last updated each attribute.

The semantics:

- If the fields named in the class's **static** reconciliation rule have not been updated by the owning source within the Effective Duration, the CI (or that attribute) is judged **stale for that source**.
- A stale judgement authorises a **lower-priority** source to update it. This is the failover path that stops a dead integration from freezing CMDB data forever.
- **Effective Duration = 0 (default)** means the CI is stale *immediately* after an update, so any lower-priority authorised source may update it. Setting 0 is therefore not "disabled" — it is maximum permissiveness, and a common accidental cause of attribute flapping.
- A malformed value like `15 x` uses the valid prefix (`15`); a wholly invalid value falls back to `0`.
- Rule precedence for the class is unchanged by the presence of data refresh rules — refresh adds a staleness gate, it does not reorder priority.

Data refresh rules apply to supported non-CMDB tables too (IRE supports non-CMDB tables), where they are configured via `cmdb_datasource_staleness.list` rather than through the CI Class Manager.

> Source: [Create a data refresh rule](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/create-datasource-staleness-rule.html), [Create a data refresh rule for a non-CMDB table](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/create-non-cmdb-data-refresh-rule.html), [IRE support for non-CMDB tables](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/ire-support-non-cmdb-tables.html)

### 5.9 How the four compose

```
Incoming CI payload
        |
        v
[1] IDENTIFICATION RULES (cmdb_identifier, one rule/class, ordered entries)
        |
        +-- no match --> INSERT
        |                  |
        |                  +-- [3] IRE DATA SOURCE RULE: is this source
        |                  |        permitted to INSERT this class?
        |                  |        no  -> rejected
        |                  |        yes -> CI created, reconciliation skipped
        |
        +-- match   --> UPDATE
                           |
                           +-- [2] RECONCILIATION (static or dynamic):
                           |        is this source authorised for this
                           |        class/attribute? (field-level for static)
                           |
                           +-- [4] DATA REFRESH RULE (static mode only):
                                    is the CI stale for the owning source?
                                    yes -> lower-priority source may write
                                    no  -> higher-priority source retained
```

### 5.10 The audit trail

Every CI carries `discovery_source`, and `cmdb_datasource_last_update` records the last source per attribute. Together they answer "who wrote this field, and when" — which is the only defensible way to debug attribute churn. Discovery simulation (`cmdb_ie_context`, `cmdb_ie_run`, `cmdb_ie_log`) replays a payload through IRE without committing, and is the correct tool before changing an identifier on a populated instance.

> Source: [Identification and Reconciliation Engine (IRE)](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/ire.html), [Configuring CMDB Identification and Reconciliation](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/configuring-ire.html)

---

## 6. Class-level vs attribute-level scope

Two independent scopes govern what Discovery is allowed to touch. Confusing them is the root of most "why did/didn't Discovery change this" questions.

### 6.1 Class-level scope — *which CI type, which table*

Controls:

- **Which CIs are eligible.** Discovery can only write to the class the classifier/pattern names. A custom device type with no classifier and no pattern will never be discovered, regardless of credentials.
- **Identifier inheritance.** Rules set on a parent apply to children.
- **Relationship legality.** A relationship is only created if the relation type permits that class pair. Wrong class → no relationship, silently.
- **Reclassification.** If classification changes (a host re-imaged from Linux to Windows), the CI class changes; identity may or may not survive depending on whether the identifier attributes still match.
- **Pattern applicability.** Patterns are bound to CI types; a CI type with no pattern falls back to probes, and a CI type with neither is undiscoverable.
- **Where the data lands.** This decides whether your servers appear in `cmdb_ci_linux_server` (good) or in `cmdb_ci` base (a defect that breaks every downstream report and CSDM alignment).

### 6.2 Attribute-level scope — *which fields, which values*

Controls:

- **Collection.** A probe or pattern collects a defined set of attributes. Attributes not collected are simply not in the payload — Discovery cannot update what it did not observe.
- **Reconciliation (static or dynamic).** Even when collected, an attribute may be blocked from update because the source is not authorised for that field or attribute.
- **IRE data source rules.** Separate from reconciliation: this gates whether the source may *insert* CIs of the class, not which attributes it may write. A source can be blocked from inserting and still permitted to update.
- **Data refresh rules.** Only relevant in static reconciliation mode; determines whether a lower-priority source may take over a stale attribute.
- **Foundational vs non-foundational.** Determines whether an attribute is considered Discovery-owned.
- **Normalisation.** Sensors and process handler scripts normalise values (e.g. collapsing `OpenJDK 17.0.9` and `openjdk-17.0.9` into one software CI). Weak normalisation creates duplicate software CIs even when the host CI is correct.
- **Per-attribute write permissions.** Some attributes are write-protected or derived from relationships.

### 6.3 Where the distinction bites

| Situation | Scope that matters | Symptom if mishandled |
|---|---|---|
| Server lands in `cmdb_ci` instead of `cmdb_ci_linux_server` | Class | CSDM/reporting broken, relationships missing |
| Nightly overwrite of assignment group | Attribute | Business data churn; trust erosion |
| Duplicate host CIs after adding a domain suffix | Class+attribute | Two CIs, split relationships, double-counted licences |
| New device type never discovered | Class | Silent invisibility; no error anywhere |
| Software titles duplicated 5× | Attribute (normalisation) | Licence counts wrong |
| Relationship missing between VM and hypervisor | Class pair | Dependency maps incomplete |
| Same host discovered by connector and by Discovery | Identification (class-level) | Duplicate CIs |

### 6.4 What actually controls each

- Class scope: classifiers, patterns/CI types, discovery behaviours, schedule targets, MID reachability.
- Attribute scope: probe/pattern definitions, sensors, reconciliation rules, CI class attribute definitions, foundational-data classification.

---

## 7. Related workflows and integrations

### 7.1 Upstream dependencies (Discovery cannot work without these)

| Dependency | Why |
|---|---|
| MID Server | Sole execution engine; must be Up/Validated/assigned |
| Credentials store | Authentication to targets |
| ECC queue | Transport |
| IRE + identification/reconciliation rules | CMDB write path |
| CMDB class model | Target tables, identifiers, relationship legality |
| Patterns (for modern targets) | Collection definitions |
| Network/DNS | Reachability and FQDN correctness |
| Roles (`discovery_admin` etc.) | Authoring vs running |

### 7.2 Downstream consumers (Discovery feeds these)

| Consumer | Consumes | Notes |
|---|---|---|
| **CMDB / CMDB Workspace** | CIs, attributes, relationships | The primary output |
| **Service Mapping** | Horizontal-discovery results as entry points | Requires Discovery active and configured; builds top-down maps, `cmdb_tcp` / `sa_flow_connection` enrichment |
| **Event Management** | CIs and service context | Alert binding and service health; Discovery does not generate alerts |
| **Dependency Views** | Relationships | Visual impact analysis |
| **Change Management** | CIs, relationships | Impact and risk calculation |
| **Incident Management** | CIs | Affected-CI context |
| **Application Portfolio Management (APM)** | Application CIs | Business application makeup |
| **CSDM alignment** | Class + relationship correctness | Discovery data quality is a CSDM prerequisite |
| **Asset Management / HAM / SAM** | Hardware and software inventory | Licence reconciliation, warranty |
| **Certificate Inventory and Management** | Certificate discovery patterns | Expiry tracking |
| **AIOps / Health Log Analytics** | Topology | Anomaly context |
| **CMDB 360 / CMDB health** | Completeness/consistency metrics | Surfaces missing identifiers, orphaned CIs |

### 7.3 Adjacent collection methods that write to the same CMDB

These are alternatives or complements; all should route through IRE:

- **Service Graph Connectors** — vendor/cloud/third-party data (SCCM, Intune, AWS, Azure, GCP, monitoring tools).
- **Agent Client Collector (ACC)** — agent-based, for endpoints, domain controllers, DMZ.
- **Cloud Discovery** — API/metadata-based; event-driven refresh possible.
- **Import sets / integrations** — bulk loads.
- **Manual entry** — the baseline that discovery augments.

If these use different identifiers than Discovery, you get duplicates. Identifier alignment across all sources is a design requirement, not an optimisation.

> Source: [Exploring Service Mapping](https://www.servicenow.com/docs/r/it-operations-management/service-mapping/service-mapping-get-started.html), [Advanced Discovery configuration](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryExtendedCapabilities.html)

---

## 8. Edge cases, failure modes and interactions with adjacent systems

### 8.1 Failure localisation method (use this before changing anything)

Walk the chain and find the **first** layer that diverges:

```
schedule → ECC output → MID pickup → network/auth → probe execution
        → ECC input → sensor/pattern parse → IRE → CI/relationship
```

A missing CI at the end is a *symptom*. The first upstream divergence is the *cause*. Concretely:

- ECC **output** stuck `ready` → MID not picking up (down, unvalidated, wrong app assignment, wrong cluster/queue).
- ECC **output** in `processing` forever → MID claimed it and died mid-execution.
- ECC **input** stuck `processing`/`error` → sensor script exception or payload problem.
- Input processed, no CI → IRE identification rejected or matched something unexpected.

> Source: [Resolving MID Server issues](https://www.servicenow.com/docs/r/servicenow-platform/mid-server/r_MIDServerTroubleshooting.html)

### 8.2 Failure catalogue

**Infrastructure / transport**

| Failure | Signature | Cause | Fix |
|---|---|---|---|
| MID down | MID Server Down alert; ECC output piling up | Service stopped, host reboot, heap exhaustion | Restart service; check `agent0.log.0`, `wrapper.txt`; raise max heap |
| MID unvalidated | MID shows Down despite process running | Instance→MID auth broken, post-clone, password change | Re-validate; check MID user credentials |
| Wrong app assignment | Work never claimed | MID not assigned Discovery application | Assign application to MID |
| MID unreachable from segment | Some ranges never scan | Firewall/routing/ACL | Prove MID→target from the MID host itself |

**Network / protocol**

| Failure | Signature | Cause | Fix |
|---|---|---|---|
| WMI blocked | `RPC server unavailable`, `Access is denied`, `Get-CimClass : Access is denied` | TCP 135 open but dynamic RPC range blocked | Open 135 + RPC range; or move to WinRM 5985/5986 |
| WinRM blocked | Auth failures despite valid creds | 5985/5986 closed, or WinRM not configured | Enable/allow WinRM |
| SSH blocked | Classification never completes on Linux | 22 filtered | Open path; verify with `nc -vz <ip> 22` from MID |
| SNMP misconfigured | Network gear invisible | Wrong community, v3 auth/priv mismatch | Correct credential type; SNMP walk test from MID |
| DNS broken | CI named with IP, credential affinity misses, wrong FQDN | Missing PTR/A records | Fix DNS; verify `nslookup <ip>` and `nslookup <hostname>` from MID |
| Proxy/TLS interception | Cloud/vCenter API calls fail | TLS inspection, expired cert | Trust the CA on the MID |

**Credentials**

| Failure | Signature | Cause | Fix |
|---|---|---|---|
| `NOCREDENTIALS` / `No credential found for type [Windows]` | `credentials_attempted: []` in ECC payload | No credential of that type visible to this MID | Fix type; set Applies-to to include the MID |
| `AUTHENTICATIONFAILED` | Credentials attempted and rejected | Wrong password, domain format, insufficient rights | Test from MID (`gwmi`/`New-PSSession`); fix account rights |
| Account lockout | Intermittent auth after a run | Long credential list with wrong entries tried in order | Trim list; order correctly |
| Exclusion list masking a fix | Still failing right after credential fix | MID caches failed IPs ~5 min | Wait 5 min, restart MID, or save the credential to clear cache |
| Post-clone breakage | Everything auth-fails after clone | MID/credential secrets invalidated | Re-validate MID, re-enter credential secrets |

**Payload / processing**

| Failure | Signature | Cause | Fix |
|---|---|---|---|
| Payload too large | Results discarded with warning | `mid.discovery.max_payload_size` exceeded | Set negative value to disable, or reduce page size |
| XML parse error | Sensor fails to parse input payload | Illegal characters, missing CDATA, stray whitespace before XML declaration | Fix probe output/encoding; escape or CDATA |
| Probe not found | Error referencing probe sys_id | Classifier `trigger_probes` names a deleted/renamed probe | Correct the classifier |
| Pattern/MID version mismatch | Pattern fails or silently partial | Instance pattern newer than MID support | Upgrade MID / align pattern version |
| Process noise | Thousands of process CIs | No process filter | Configure process filtering |

**CMDB / IRE**

| Failure | Signature | Cause | Fix |
|---|---|---|---|
| Duplicate CIs | Two CIs for one host | Identifier mismatch (name vs serial), or two sources, or identifier changed | Align identifiers across sources; merge and re-run |
| Missing relationships | CIs present, no edges | Wrong class pair, or relationship rejected by IRE | Verify class; verify relation type allowed |
| Business attributes reset nightly | Assignment/support group churn | Reconciliation authorises the discovery source to write non-foundational fields | Restrict the class's static/dynamic reconciliation rules to foundational attributes |
| Source keeps inserting CIs after being "restricted" | Restriction appears in reconciliation, yet new CIs still appear | The restriction belongs in an **IRE data source rule** (`cmdb_ire_data_source_rule`), not in reconciliation | Create a data source rule for that source + class |
| Dead integration freezes attributes | A retired source still owns fields; nobody else can update them | No **data refresh rule**, or dynamic reconciliation is enabled (which disables refresh rules) | Create `cmdb_datasource_staleness` entries with a realistic Effective Duration; confirm dynamic reconciliation is not in effect |
| Attribute flapping between two sources | Values oscillate run to run | Data refresh rule with Effective Duration `0` marks the CI stale immediately, so every lower-priority source is always authorised | Set a non-zero Effective Duration |
| Duplicates reappear after every merge | Merge succeeds, next run re-splits the CI | Identification entries inconsistent between sources | Align identifier entries across all sources before merging |
| Reclassification surprises | CI silently changes class, or a reclassification task appears | Auto-reclassification disabled → task generated instead of applied | Review `reclassification_task` records; decide per case |
| Wrong table | CI in `cmdb_ci` base | Classification failed / no classifier | Add classifier or pattern for the type |
| Orphaned CIs | Devices retired but CIs remain | No lifecycle/retirement handling | Implement staleness/age-out process (Discovery does not delete) |
| Concurrency churn | Constant attribute flapping | Overlapping schedules or duplicated ranges | Deconflict schedules and ranges |

**Operational**

| Failure | Signature | Cause | Fix |
|---|---|---|---|
| Mid-run instance clone | Discovery half-writes, ECC inconsistency | Clone during run | Quiesce discovery around clones |
| Throttling collapse | Instance slow under discovery load | Synchronous job processing at scale | Enable Event Framework (`discovery.use.event.processing`) and prioritise jobs |
| Window overrun | Runs never complete | Range too large for one MID/window | Shard ranges across MIDs/clusters; stagger |
| Self-discovery | MID discovers itself / odd credential results | MID host in range | Exclude the MID host from ranges |
| Silent success, stale data | Status completes, data old | Target changed but schedule disabled | Verify schedule cadence against drift rate |

### 8.3 Interactions with adjacent systems that cause confusing symptoms

- **Service Mapping** depends on Discovery being correct. If horizontal discovery misclassifies hosts, Service Mapping builds maps against wrong CIs, and the failure surfaces as "Service Mapping is broken" when the defect is a classifier.
- **Event Management** binds alerts to CIs. Duplicate CIs split one host's alerts across two records, producing "missing alerts" that are actually "alerts on the other duplicate".
- **Change Management** computes impact from relationships. Missing relationships make changes look low-risk when they are not — the most dangerous class of Discovery defect, because it fails safe-looking.
- **HAM/SAM** count from CI records. Duplicates inflate licence counts; missing software CIs deflate them. Both are financial errors with no Discovery error message.
- **Service Graph Connectors** and Discovery both write through IRE; inconsistent identifiers across the two produce systematic duplicates that reappear after every merge until identifiers are aligned.
- **CMDB health/CSDM** dashboards will flag the symptoms (unclassed CIs, missing identifiers) that Discovery's own status page reports as success.

### 8.4 Non-obvious behaviours worth knowing

- Classification runs **one probe per device** — the cheapest probe that answers wins, so a device reachable by two protocols is classified by whichever answers first, which can differ between runs if one protocol is intermittently slow.
- Exploration is scoped by the classifier, so a classifier with a truncated `trigger_probes` list produces a *partially populated* CI with no error — the most insidious failure mode because status says complete.
- IRE does not delete. Retirement, staleness and de-duplication are separate governance processes.
- Credential caches (and the exclusion list) make behaviour time-dependent: the same command run twice one minute apart can give different results.
- Discovery is idempotent per identifier. Running the same schedule twice in a row should produce one CI and a change-history entry only if something actually drifted — if it produces a second CI, identity is wrong, not the schedule.

---

## 9. Source list (all verified reachable, HTTP 200)

1. [Exploring Discovery](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_GetStartedWithDiscovery.html)
2. [Horizontal discovery process flow with probes and sensors](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoProcessFlows.html)
3. [Discovery classifiers](https://www.servicenow.com/docs/r/it-operations-management/discovery/discovery-classifiers.html)
4. [Probes and sensors](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryProbesAndSensors.html)
5. [Discovery configuration](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryConfiguration.html)
6. [Create a discovery schedule](https://www.servicenow.com/docs/r/it-operations-management/discovery/t_CreateADiscoverySchedule.html)
7. [Running discoveries](https://www.servicenow.com/docs/r/it-operations-management/discovery/running-discoveries.html)
8. [Advanced Discovery configuration](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryExtendedCapabilities.html)
9. [Discovery properties](https://www.servicenow.com/docs/r/it-operations-management/discovery/r_DiscoveryProperties.html)
10. [Discovery error messages](https://www.servicenow.com/docs/r/it-operations-management/discovery/c_DiscoveryErrorMessages.html)
11. [Identification and Reconciliation Engine (IRE)](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/ire.html)
12. [Components and process of identification and reconciliation](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/c_CompsandProcessIDandReconcil.html)
13. [Create CI types for Service Mapping and Discovery](https://www.servicenow.com/docs/r/it-operations-management/discovery-and-service-mapping-patterns/t_CreateCITypeForSM.html)
14. [Resolving MID Server issues](https://www.servicenow.com/docs/r/servicenow-platform/mid-server/r_MIDServerTroubleshooting.html)
15. [Credentials troubleshooting](https://www.servicenow.com/docs/r/platform-security/connections-and-credentials/r_CredentialTroubleshooting.html)
16. [Exploring Service Mapping](https://www.servicenow.com/docs/r/it-operations-management/service-mapping/service-mapping-get-started.html)
17. [Pattern-based discovery in Service Mapping](https://www.servicenow.com/docs/r/zurich/it-operations-management/service-mapping/pattern-based-discovery.html)
18. [CMDB Identification and Reconciliation (IRE) landing](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/c_CMDBIdentifyandReconcile.html)
19. [Exploring IRE](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/exploring-ire.html)
20. [Configuring CMDB Identification and Reconciliation](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/configuring-ire.html)
21. [Components installed with Identification and Reconciliation (rule/table map)](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/r_IWIdentAndReconcil.html)
22. [Create a data refresh rule](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/create-datasource-staleness-rule.html)
23. [Create a data refresh rule for a non-CMDB table](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/create-non-cmdb-data-refresh-rule.html)
24. [IRE support for non-CMDB tables](https://www.servicenow.com/docs/r/servicenow-platform/configuration-management-database-cmdb/ire-support-non-cmdb-tables.html)
