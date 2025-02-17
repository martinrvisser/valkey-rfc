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
- TLS 1.2+ required for AD communication
- certificate validation
- secure credential storage
- encrypted cache entries

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

### High Availabilty

- multiple AD server support
- connection failover
- cache replication
- graceful degradation

### Error Handling

- Invalid credentials
- Network timeouts
- AD server unavailable
- Cache corruption
- Configuration errors
- Group resolution failures

## Specification

### Authentication flow

1. Client sends AUTH command
2. Module extracts username/password
3. Check credential cache
4. If not cached:
    - connect to AD server
    - perform bind operation
    - search for user group membership
    - cache successful result
    - set TTL
5. Pass AUTH/NOAUTH to Valkey core
6. Return auth response

### module load

- load configuration from Valkey config or default values
- initialize data structures and connection pools
- register command hooks with valkey core
- establish initial AD server connections
- load cached credentials if persistence enabled
- register monitoring callbacks

### Commands 

1 AUTH.CONFIG SET <parameter> <value>

- sets configuration parameters for the AD authentication module
- parameters include server addresses, timeouts, cache settings, etc.
- returns simple string reply: OK on success

2 AUTH.CONFIG GET <parameter>

- gets current configuration for specified parameter
- returns bulk string reply: parameter value

3 AUTH.RELOAD

- reloads configuration from config file
- reestablishes connections with AD servers
- returns simple string reply: OK on success

4 AUTH.STATUS

- returns health and statistics information
- format: nested hash with connection states, cache stats, auth counts

5 AUTH.FLUSHCACHE

- clears the credential cache
- options to flush all or specific users
- returns integer reply: number of entries flushed

### Valkey Command Hooks

#### Authentication Command Hook

- intercepts AUTH command
- processes credentials through AD binding
- maintains original Valkey auth functionality for local accounts

#### ACL Command Hook

- integrates with ACL commands
- maps AD groups to Valkey ACLs
- preserves ACL inheritance and evaluation logic

### internal data structures

#### Configuration Store

- hash table for configuration parameters
- persistent across restarts via RDB or config file
- thread-safe access with read-write locks

#### Credential Cache

- LRU cache with configurable TTL
- encrypted storage with secure key management
- size-limited with automatic eviction
- concurrent access optimized

#### Group Mapping Table

- hash table mapping AD groups to Valkey ACLs
- configurable refresh interval
- support for nested group resolution

#### Connection Pool

- LDAP connection objects with state tracking
- priority queue based on connection health
- automatic connection cycling
- size limits and idle timeouts

### Configuration

#### Server Configuration

- AUTH.servers: list of AD server addresses (comma-separated)
- AUTH.port: LDAP port (default: 636 for LDAPS)
- AUTH.use_ssl: whether to use LDAPS (default: true)
- AUTH.timeout.connect: connection timeout in ms (default: 1000)
- AUTH.timeout.search: search operation timeout in ms (default: 2000)
- AUTH.retry.max: maximum retry attempts (default: 3)
- AUTH.retry.delay: delay between retries in ms (default: 500)

#### Authentication Configuration

- AUTH.basedn: nase DN for user searches
- AUTH.binddn: service account DN for initial bind
- AUTH.bindpw: service account password (stored securely)
- AUTH.user_filter: LDAP filter for user searches
- AUTH.group_filter: LDAP filter for group membership queries
- AUTH.default_permissions: default ACL for authenticated users

#### Cache Configuration

- AUTH.cache.enabled: enable credential caching (default: true)
- AUTH.cache.ttl: cache entry TTL in seconds (default: 300)
- AUTH.cache.size: maximum cache size (default: 10000)
- AUTH.cache.persist: persist cache across restarts (default: false)
- AUTH.cache.encryption: encryption method for cached credentials

#### Security Configuration

- AUTH.tls.cert: path to client certificate
- AUTH.tls.key: path to client key
- AUTH.tls.ca: path to CA certificate
- AUTH.tls.verify: verify server certificates (default: true)
- AUTH.tls.ciphers: allowed cipher suites
- AUTH.tls.version: minimum TLS version (default: TLS1.2)

### Metrics and Monitoring

#### Exported Metrics
- authentication attempts (success/failure)
- cache performance (hit ratio, size, evictions)
- AD connection status (latency, errors, timeouts)
- group resolution performance
- resource utilization (memory, CPU)

#### Health Indicators
- AD server connectivity
- cache integrity
- configuration consistency
- error rates and patterns
- response time percentiles

### Error Handling and Logging

#### Error Categories
- configuration errors (invalid parameters, inconsistent settings)
- connection errors (network failures, timeouts, rejected connections)
- authentication errors (invalid credentials, expired accounts)
- authorization errors (insufficient permissions, group mapping failures)
- internal errors (memory allocation, thread management)

#### Logging Levels

- ERROR: critical issues requiring immediate attention
- WARN: potential issues that don't prevent operation
- INFO: normal operational events
- DEBUG: detailed debugging information
- TRACE: full protocol-level logging for troubleshooting

### Cluster mode

TBD :
- this should follow the existing authentication rules for Valkey Cluster
- configuration should be sync-ed across all nodes
- authentication cache should be consistent across nodes 

### Replication

TDB :
- this should follow the existing authentication rules for Valkey primaries and replicas
- authentication cache should be consistent across primary and replica

### Dependencies 

- ldap library
