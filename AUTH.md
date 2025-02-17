---
RFC: (PR number)
Status: (Change to Proposed when it's ready for review)
---

# ValkeyAUTH Module RFC

## Abstract

The proposed Valkey AUTH module, named ValkeyAUTH, supports authentication to Valkey through external to Valkey managed authentication mechanisms. The module provides seamless integration between Valkey's authentication system and external enterprise authentication infrastructure, allowing organizations to leverage existing identity management systems. The design emphasizes security, scalability, and minimal performance impact while providing robust user authentication and role-based access control.

## Motivation

Organizations running Valkey in enterprise environments frequently need to integrate with existing identity management systems so that this can be centrally managed. This module addresses these challenges by:

- eliminating the need for external authentication proxies
- reducing operational complexity
- providing native Active Directory integration
- enabling centralized user and permission management
- maintaining Valkey's performance characteristics
- maintaining Valkey's existing authenthication workflow
- supporting compliance requirements for centralized authentication

The work done in this Redis Pull Request #11659 means authentication can be handed of to a Valkey module without affecting the Valkey core authentication.

## Design considerations

### Performance Impact

- minimize authentication overhead through efficient caching
- optimize LDAP queries and connection management
- implement connection pooling to reduce latency
- balance security with performance requirements
- ensure minimal impact on Valkey command execution

### Scalability

- support for large user bases and group hierarchies
- efficient handling of concurrent authentication requests
- horizontal scalability in clustered environments
- resource usage optimization for high-load scenarios
- graceful degradation under heavy load

### Security

- zero trust security model implementation
- secure credential handling and storage
- protection against common attack vectors
- audit trail for authentication events

### Maintainability

- clear separation of concerns
- modular design for easy updates and inclusion of new protocols
- comprehensive logging and monitoring
- simple configuration management
- easy troubleshooting and debugging

### Compatibility

- support for various Valkey deployment models
- backward compatibility with existing Valkey features
- integration with different LDAP providers
- support for multiple authentication methods
- compatibility with existing Valkey clients

## Specification (Required)

A more detailed description of the feature, including the reasoning behind the design choices.

### Commands (Optional)

If any new commands are introduced:

1. Command name
   - **Request**
   - **Response**

### Authentication and Authorization (Optional)

If there are any changes around introducing new ACL command/categories for user access control.

### Append-only file (Optional)

If there are any changes around the persistence mechanism of every write operation.

### RDB (Optional)

If there are any changes in snapshotting mechanisms like new data type, version, etc.

### Configuration (Optional)

If there are any configuration changes introduced to enable/disable/modify the behavior of the feature.

### Keyspace notifications (Optional)

If there are any events to be introduced or modified to observe activity around the dataset.

### Cluster mode (Optional)

If there is any special handling for this feature (e.g., client redirection, Sharded PubSub, etc) in cluster mode or if there are any new cluster bus extensions or messages introduced, list out the changes.

### Module API (Optional)

If any new module APIs are needed to implement or support this feature.

### Replication (Optional)

If there are any changes required in the replication mechanism between a primary and replica.

### Networking (Optional)

If there are any changes introduced in the RESP protocol (RESP), client behavior, new server-client interaction mechanism (TCP, RDMA), etc.

### Dependencies (Optional)

If there are any new dependency libraries required to support the feature. Existing dependencies are jemalloc, lua, etc. If the library needs to be vendored into the project, please add supporting reason for it.

### Benchmarking (Optional)

If there are any benchmarks performed and preliminary results (add the hardware/software setup) are available to share or a set of scenarios identified to measure the feature's performance. 

### Testing (Optional)

If there are any test scenarios planned to ensure the feature's stability and validate its behavior.

### Observability (Optional)

If there are any new metrics/stats to be introduced to observe behavior or measure the performance of the feature.

### Debug mechanism (Optional)

If there is any debug mechanism introduced to support admin/operators for maintaining the feature.

## Appendix (Optional)

Links to related material such as issues, pull requests, papers, or other references.
