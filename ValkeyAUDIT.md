---
RFC: (PR number)
Status: Proposed
---

# ValkeyAUDIT Module RFC

## Abstract

The proposed Valkey Audit module, named ValkeyAudit supports the auditing of Valkey connections and commands. The module provides integration with standard external enterprise audit collection software and endpoints, e.g. syslog and splunk. The design allows for a highly custimisable deployment through configuration options of the types of events and commands to audit. 

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

### Configurability

Each type of event should be toggelable to audit
- connections and disconnections
- auth requests with redaction of password
- config commands
- key operations

For key operations, the following options should be configurable
- disable logging of operation payload
- configurable size of payload

### Performance

- asynchronous processing where possible
- efficient data structures for event capture
- memory limits for local buffers
- I/O throttling for external transmissions

### Security 

- removal of potentially sensitive payloads
- removal of sensistive authentication information
- encryption of audit log data in transit
- access control for audit commands

### Compatibility

- support for various Valkey deployment models
- integration with different external log collection systems
- standard formats for log records e.g. RFC5424 syslog protocol

## Specification

Valkey modules can subscribe to Valkey server events. Client events like connections and disconnections can be handled through the ValkeyModuleEvent_ClientChange server event. 

Valkey server command execution can be plugged into by registering command filters. The filter applies in all execution paths including:

1. Invocation by a client.
2. Invocation through [`ValkeyModule_Call()`](https://valkey.io/topics/modules-api-ref/#ValkeyModule_Call) by any module.
3. Invocation through Lua `server.call()`.
4. Replication of a command from a primary.

### Components

#### Command filter

- hooks into the Valkey command execution pipeline
- captures commands in all execution paths
- collects metadata about each operation
- configurable command categories

#### Event capture

- subscribes to client connection and disconnection events
- subscribes to AUTH operations
- captures failed connection attempts
- configurable on/of for each category

#### Event/command Processor

- formats captured events into structured audit records
- applies filtering based on configuration
- enriches events with additional context

### Transport Layer

- forwards audit events to external systems
- supports multiple transport protocols, primarily TCP for syslog-NG integration
- handles retries, backpressure, and connection management

### Configuration Manager

- manages module settings via Valkey module API
- provides dynamic reconfiguration capabilities
- stores persistent configuration in Valkey

### On module load

- load configuration from Valkey config or default values
- initialize data structures and verify audit destination
- register command filters with valkey core as per config
- subscribe to connection type events as per config

## Audit Event Types

### DB Commands captured data

- Timestamp
- Command name, key and arguments
    - with option not to capture payload
    - payload length limit configurable
- Source IP/port
- Target IP/port
- Username
- Database ID
- Command status (success/failure)

### Database Disconnects captured data

- Timestamp
- Client ID
- Source IP/port
- Target IP/port
- Database ID
- Disconnect status (graceful/error)
- Disconnect reason (if available)

### Database Connection attempts captured data

- Timestamp
- Client ID
- Source IP/port
- Target IP/port
- Database ID
- Connection status (success/failure)
- Failure reason (if applicable)

### Authentication Requests captured data

- Timestamp
- Client ID
- Authentication action (new/existing connection)
- Source IP/port
- Target IP/port
- Username
- Database ID
- ACL rules used for verification
- Authentication status (success/failure)

## External Transport

### Transport Protocols

- TCP (primary for syslog-NG integration)
- Optional extensions for UDP, HTTP, or message queues

### Message Formats

- Syslog format (RFC 5424)
- in a later phase, JSON
- in a later phase, configurable custom formats

### Transport Features

- buffering with configurable limits
- retry logic with exponential backoff
- TLS support for encrypted transmission
- authentication mechanisms for secure deliver

### Commands 

CONFIG SET AUDIT.CONFIG <parameter> <value>

#### Command Filtering

- command category
- status (success/failure)
- on/off for payload

#### Performance Settings

- Buffer sizes
- flush intervals for writing to target
- payload length limit configurable

#### Transport Configuration

- endpoint configuration : initially syslog target


