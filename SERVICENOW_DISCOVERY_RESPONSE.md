# ServiceNow Discovery Technical Reference

## A. What ServiceNow Discovery Is and What Problem It Solves

### Definition, Purpose, and Business Value
ServiceNow Discovery is a comprehensive solution for automatically discovering, identifying, and managing physical and virtual assets in your IT environment. It helps maintain an accurate and up-to-date Configuration Management Database (CMDB) by eliminating the need for manual inventory management.

### What Problem It Solves
- **CMDB Accuracy**: Ensures that the CMDB contains accurate and up-to-date information about all IT assets.
- **Manual Inventory Elimination**: Reduces the need for manual inventory processes, which are prone to errors and delays.
- **Security Visibility**: Provides visibility into all assets, enabling better security monitoring and risk management.

## B. Major Components/Subsystems Within Discovery and How They Interact

### MID Server
**Types and Capabilities**: MID Servers are lightweight agents that run on the devices to be discovered. They execute probes and send data back to the ServiceNow instance.
**Installation**: MID Servers can be installed on Windows, Linux, and macOS.
**Communication**: They communicate with the ServiceNow instance using HTTP/HTTPS.

### Discovery Engine
**Orchestration Role**: The Discovery Engine manages MID Servers, coordinates data collection, and schedules discovery tasks.
**Configuration**: It is configured through the Discovery Rules and Profiles.

### Discovery Rules
**Define What and How to Discover**: Discovery Rules specify what devices to discover and how to discover them.
**Configuration**: They are configured in the Discovery > Rules module.

### Discovery Schedules
**Control Timing and Frequency**: Schedules define when discovery tasks should run.
**Configuration**: They are configured in the Discovery > Schedules module.

### Discovery Profiles
**Group Probes and Parameters**: Profiles group related probes and parameters for efficient discovery.
**Configuration**: They are configured in the Discovery > Profiles module.

### Discovery Probes
**What They Are**: Probes are scripts that run on the MID Server to collect data from the device.
**How They Execute**: They execute on the MID Server and send the collected data back to the ServiceNow instance.
**Examples**:
- SSH
- WMI
- SNMP
- JDBC
- HTTP

### Discovery Sensors
**What They Parse**: Sensors parse the raw data collected by probes.
**How They Convert Raw Data**: They convert raw probe output into structured data that can be used by the CMDB.
**Configuration**: They are configured in the Discovery > Sensors module.

### CI Identifier / Identification Rules
**Match Discovered CIs to Existing CIs**: Identification Rules match discovered CIs to existing CIs in the CMDB.
**Configuration**: They are configured in the Discovery > Identification Rules module.

### Reconciliation Rules
**Apply Discovered Data to Existing CIs**: Reconciliation Rules apply the discovered data to existing CIs in the CMDB.
**Configuration**: They are configured in the Discovery > Reconciliation Rules module.

## C. Complete Data Flow - Step by Step from First Contact to CMDB

### Phase 1: Discovery Scheduling and Triggering
**What Happens**: Discovery schedules are triggered based on the configured recurrence patterns.
**What Triggers It**: The Discovery Engine checks the schedules and triggers the discovery tasks.
**What Outputs It Produces**: Discovery tasks are created and scheduled.

### Phase 2: MID Server Selection and Probe Dispatch
**What Happens**: The Discovery Engine selects an available MID Server and dispatches the discovery probes.
**What Triggers It**: The discovery tasks are triggered.
**What Outputs It Produces**: The MID Server executes the probes and sends the collected data back to the ServiceNow instance.

### Phase 3: Device Identification / Classification
**What Happens**: The collected data is parsed and normalized by sensors.
**What Triggers It**: The MID Server sends the collected data back to the ServiceNow instance.
**What Outputs It Produces**: The data is parsed and normalized into a structured format.

### Phase 4: Data Collection (SSH, WMI, SNMP, JDBC, API/REST Protocols)
**What Happens**: Probes execute on the MID Server to collect data from the device.
**What Triggers It**: The discovery tasks are triggered.
**What Outputs It Produces**: Raw data collected from the device.

### Phase 5: Data Parsing and Normalization by Sensors
**What Happens**: Sensors parse the raw data collected by probes and normalize it into a structured format.
**What Triggers It**: The MID Server sends the collected data back to the ServiceNow instance.
**What Outputs It Produces**: Structured data that can be used by the CMDB.

### Phase 6: Identification and Reconciliation (IRE)
**What Happens**: The Identification and Reconciliation Engine (IRE) matches the discovered CIs to existing CIs in the CMDB and applies the discovered data to the existing CIs.
**What Triggers It**: The data is parsed and normalized.
**What Outputs It Produces**: Updated or new CI records in the CMDB.

### Phase 7: CMDB Population and Relationship Mapping
**What Happens**: The updated or new CI records are populated in the CMDB, and relationships are mapped.
**What Triggers It**: The IRE completes the identification and reconciliation process.
**What Outputs It Produces**: Updated CMDB with accurate and up-to-date information about all IT assets.

## D. Configuration Locations, Triggers, Schedules, and Dependencies

### Exact Navigation Paths
- **Discovery Rules**: Discovery > Rules
- **Discovery Schedules**: Discovery > Schedules
- **Discovery Profiles**: Discovery > Profiles
- **Discovery Probes**: Discovery > Probes
- **Discovery Sensors**: Discovery > Sensors
- **CI Identifier / Identification Rules**: Discovery > Identification Rules
- **Reconciliation Rules**: Discovery > Reconciliation Rules

### Schedule Types and Recurrence Patterns
- **One-time**: Runs once at a specific time.
- **Recurring**: Runs at specified intervals (e.g., daily, weekly, monthly).

### Dependencies Between Discovery and Other Processes
- **CMDB**: Discovery feeds data into the CMDB.
- **Event Management**: Discovery triggers events.
- **Service Mapping**: Discovery provides CI relationships.
- **Orchestration**: Discovery triggers workflows.

## E. INSERT vs UPDATE BEHAVIOR

### When Discovery Creates NEW CI Records vs UPDATES Existing Ones
- **Insert**: When a new CI is discovered and no matching CI is found in the CMDB.
- **Update**: When an existing CI is discovered and a matching CI is found in the CMDB.

### How CI Identification Rules Determine Match vs No-Match
- **Match**: When the CI Identifier rules match the discovered CI to an existing CI in the CMDB.
- **No-Match**: When the CI Identifier rules do not match the discovered CI to any existing CI in the CMDB.

### How Reconciliation Rules Determine What Gets Updated vs What Stays
- **Update**: When the Reconciliation Rules determine that the discovered data should be applied to the existing CI.
- **Stay**: When the Reconciliation Rules determine that the existing CI should remain unchanged.

### What Happens When Discovery Data Conflicts with Existing CMDB Data
- **Conflict Resolution**: Discovery uses the Reconciliation Rules to resolve conflicts between the discovered data and existing CMDB data.

### System Properties or Configuration That Control Insert/Update Behavior
- **discovery.ignore_existing**: Controls whether existing CIs should be ignored during discovery.
- **discovery.update_existing**: Controls whether existing CIs should be updated during discovery.

## F. CLASS-LEVEL vs ATTRIBUTE-LEVEL SCOPE

### Class-Level Scoping
**What It Controls**: Controls the scope of discovery rules and probes at the class level.
**How It's Configured**: Configured in the Discovery Rules and Profiles.
**Examples**: Discovery rules for servers, workstations, and network devices.

### Attribute-Level Scoping
**What It Controls**: Controls the scope of discovery rules and probes at the attribute level.
**How It's Configured**: Configured in the Discovery Rules and Profiles.
**Examples**: Discovery rules for specific attributes such as IP address, hostname, and OS version.

### What Controls Each Level
- **Discovery Rules**: Define the scope of discovery at both class and attribute levels.
- **System Properties**: Control the default behavior of discovery at both class and attribute levels.

## G. RELATED WORKFLOWS AND INTEGRATIONS THAT DISCOVERY DEPENDS ON OR FEEDS INTO

### CMDB
**How Discovery Feeds Data into CMDB**: Discovery populates the CMDB with accurate and up-to-date information about all IT assets.

### Event Management
**How Discovery Triggers Events**: Discovery triggers events when new devices are discovered or when existing devices are updated.

### Service Mapping
**How Discovery Provides CI Relationships**: Discovery provides CI relationships, enabling better service mapping and orchestration.

### Orchestration
**How Discovery Triggers Workflows**: Discovery triggers workflows based on the discovered data, enabling automated responses to changes in the IT environment.

## H. EDGE CASES, FAILURE MODES, AND INTERACTIONS WITH ADJACENT SYSTEMS

### New Devices Not Yet in CMDB
**What Happens**: Discovery creates new CI records for devices not yet in the CMDB.
**Failure Mode**: If the discovery fails, the new device will not be added to the CMDB.

### Dynamic Environments (Cloud, Containers)
**What Happens**: Discovery can be configured to discover devices in dynamic environments such as cloud and containers.
**Failure Mode**: If the discovery fails, the dynamic devices will not be added to the CMDB.

### Network Partitions
**What Happens**: Discovery may fail if there are network partitions.
**Failure Mode**: If the discovery fails, the devices in the partition will not be added to the CMDB.

### IP Address Changes
**What Happens**: Discovery can be configured to handle IP address changes.
**Failure Mode**: If the discovery fails, the device may not be updated in the CMDB.

### Credential Failures
**What Happens**: Discovery may fail if the credentials are incorrect.
**Failure Mode**: If the discovery fails, the device may not be added to the CMDB.

### Probe Timeouts
**What Happens**: Discovery may fail if the probes time out.
**Failure Mode**: If the discovery fails, the device may not be added to the CMDB.

### Sensor Parse Errors
**What Happens**: Discovery may fail if the sensors fail to parse the data.
**Failure Mode**: If the discovery fails, the device may not be added to the CMDB.

### Identification Failures
**What Happens**: Discovery may fail if the identification rules fail to match the discovered CI to an existing CI.
**Failure Mode**: If the discovery fails, the device may not be added to the CMDB.

### Reconciliation Failures
**What Happens**: Discovery may fail if the reconciliation rules fail to apply the discovered data to the existing CI.
**Failure Mode**: If the discovery fails, the device may not be added to the CMDB.