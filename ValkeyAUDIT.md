---
RFC: (PR number)
Status: Proposed
---

# ValkeyAUDIT Module RFC

## Abstract

The proposed Valkey Audit module, named ValkeyAudit supports the auditing of Valkey connections and commands. The module provides integration with standard external enterprise audit collection software and endpoints, e.g. syslog and imperva. The design allows for a highly custimisable deployment through configuration options of the types of events and commands to audit. 

Arguably, the work would be better placed in Valkey core but a separate Audit module allows for 
- a faster and more isolated implementation 
- less impact on core performance
- the number of Valkey users that would require auditing does not warrant a core implementation

## Motivation

Organizations running Valkey in enterprise environments frequently need to provide information about which users and clients connect, attempt to connect and disconnect to Valkey as well as information on which users ran which command. This module addresses these challenges by providing the options to log the following and provide this information in standard formats to external systems:

- data manipulation operations
- config operations
- connections and disconnections
- authentication requests

## Design considerations

There are three concerns that need to be considered in the implementation of this module: Configurability, Performance, and Security. Since performance is the main requirement for most Valkey applications, this will be the primary concern. 

### Configurability

Regarding the configurability concern, there are two configuration groups: the audit output configuration, and the audit events configuration. For completeness, auditing can be turned on or off with an option.

#### Protocol
The module supports several protocols to store the audit logs, and the choice of which protocol to use should be dynamically configurable by the module:
- A filesystem audit log _file_ protocol
- The _syslog_ (https://www.rfc-editor.org/rfc/rfc5424.html) protocol
- _TCP_ protocol with associated config parameters

#### Format
The format of the audit output should also be configurable by the user:
- _text_ format
- _csv_ format
- _json_ format

#### Events
Regarding the audit events configuration, the module allows to toggle the following event categories to include or exclude from auditing:
- connections and disconnections
- auth requests with redaction of password
- config commands
- key operations
- other commands not included in the above

For key operations, the following options are configurable:
- disable logging of operation payload
- configurable size of payload

Additionally there is an option to always log config commands. This is useful for logging unusual behaviour, for example for an application user you may turn of all events but always log config commands as they are not expected from an application user.

### Performance

Auditing should have as little as impact as possible on performance. Command auditing should therefore take any exclusion rules into account as early as possible.

The logging of audit events must be done as fast as possible, and therefore the event log must be kept in memory, and be asynchronously flushed using one of the configured output protocols. The memory buffer used to store the audit events must be limited by a configurable parameter. 

### Security 

There are a few aspects that will be taken into consideration regarding security:
- removal of potentially sensitive payloads
- removal of sensistive authentication information
- encryption of audit log data in transit (only applies to network protocols)

## Specification

### Connections
Valkey modules can subscribe to Valkey server events. Client connections and disconnections events can be handled through the `ValkeyModuleEvent_ClientChange` server event. 

### Authorisations
Authorisations can be handled through the `ValkeyModule_RegisterAuthCallback` callback function which is fired for all authorisations. This callback allows for the authentication attempt to be logged but not the success or failure of the authentication.

### Commands
Valkey server command execution can be plugged into by registering command filters through `ValkeyModule_RegisterCommandFilter`. The filter applies in all execution paths including:

1. Invocation by a client.
2. Invocation through [`ValkeyModule_Call()`](https://valkey.io/topics/modules-api-ref/#ValkeyModule_Call) by any module.
3. Invocation through Lua `server.call()`.
4. Replication of a command from a primary.

When the installed hooks are invoked an audit event is generated and stored in a circular memory buffer.

Each event has a well defined structure, There are several event types to be logged:
- Connection
- Authentication
- Command: key_op, config, other

For each event there is a set of fields that are commmon to all types:
- Timestamp
- Source IP/port
- Target IP/port
- Connection ID
- Username
- Event type

Then there is a set of fields specific to each event type:
- *Connection Type*
  - Connection/Disconnection
  - Connection status (success/failure)
  - Failure reason (if applicable)
- *Authentication Type*
  - Username
  - Client ID
  - Authentication status (success/failure)
  - Failure reason (if applicable)
- *Command Type*
  - Command name
  - Command key
  - Command args/payload (might be empty if payload capture is disabled, and payload length limit is configurable)

#### Command status (success/failure) 
In the initial version this will not be available since the module command filter executes before the command execution by the Valkey main server. A modification to valkey server would be needed to add a callback to get the success/failure of the command execution.

The background thread flushes the events from the memory buffer to the configured destination.

### Module Configuration

The configuration options for this module will be registered using the `ValkeyModule_RegisterStringConfig` API function, which will allow the user to set and get the options using the `CONFIG SET` and `CONFIG GET` commands.

The list of configuration options is the following:

#### General options
- `audit.enabled`: whether the logging of audit events is enabled (default `true`)
- `audit.format`: the format of audit log messages `text, csv, tcp`
- `audit.always_audit_config` : whether to always audit config commands (default `false`)
- `audit.protocol` : file, tcp, syslog

#### Protocol options
- `audit.protocol file <loc>` : file protocol and destination
- `audit.syslog <syslog_facility>` : syslog protocol and facility (default: `LOG_LOCAL0`)
- `audit.tcp <host:port>` : TCP protocol and destination
- `audit.tcp_timeout_ms` : timeout to establish connection
- `audit.tcp_max_retries` : maximum connection retries
- `audit.tcp_buffer_on_disconnect` : buffer audit entries if the connection has failed
- `audit.tcp_reconnect_on_failure` : automatically try to reconnect on connection failure

#### Exclusion
- `audit.events`: the comma separated list of event types to audit. Possible values `all, none, connections, auth, config, keys, other`, (default: all)
- `audit.excluderules`: the comma separated list exclusion rules. A rule can be one of : username, username@IP, @IP

#### Commands 
- `audit.payload_disable`: either true or false (default: `false`)
- `audit.payload_maxsize`: size in bytes of payload to record

#### `AUDIT.STATUS`

Returns the information about the module configuration, the status of the memory circular buffer, and some statistics of the background threads, in the form of a dictionary value.

Example:

- logging
  - file: enabled
    - enabled: `true`
    - path: `<file path>`
  - syslog:
    - enabled: `true`
    - facility: `<facility>`
  - stats:
    - events:
      - current: `<number of events in memory>`
      - total: `<total number of events logged>`
      - connect: `<total number of connect events logged>`
      - disconnect: `<total number of disconnect events logged>`
      - auth: `<total number of auth events logged>`
      - command: `<total number of command events logged>`
    - throughput
      - last minute:
        - logged: `<number of events logged per second>`
        - flushed: `<number of events flushed per second>`
      - last hour:
        - logged: `<number of events logged per minute>`
        - flushed: `<number of events flushed per minute>`

  
