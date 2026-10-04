# Ansible Role: galera

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-galera)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-galera)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-galera)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-galera/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-galera/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-galera/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-galera/actions/workflows/main.yml?query=branch%3Amain)

Install and configure a MariaDB Galera cluster with encrypted replication and
state transfers.

## Purpose

Install and configure MariaDB Galera nodes using distribution packages. The
jomrr.mariadb dependency installs and secures MariaDB; this role adds Galera
replication, encrypted mariabackup state transfers, explicit initial bootstrap
and readiness checks.

## Scope

### Managed

- MariaDB through jomrr.mariadb at main, the Galera provider, backup tools and
  socat.
- Cluster identity, seed addresses, node identity, InnoDB and ROW binlog
  settings, and the Galera write-set cache.
- TLS for replication, incremental state transfers and full state transfers
  using existing certificates.
- Local mysql socket authentication and backup privileges for passwordless state
  transfers.
- Enabled, running MariaDB with a Primary component and a Synced, ready node
  after convergence.

### Not Managed

- Certificate issuance or deployment, firewall policy, load balancers and backup
  schedules.
- Automatic disaster recovery, quorum overrides, data migration and
  major-version upgrades.

## Requirements

- Use the same MariaDB major version on all nodes. Distribution packages on
  AlmaLinux, Debian, Fedora, openSUSE Leap, openSUSE Tumbleweed and Ubuntu are
  supported, with their default mysql service account, data directory and Unix
  socket. Keep mariadb_service set to mariadb because the distribution bootstrap
  helper starts that unit.
- Provision a dedicated cluster CA and node certificates with serverAuth and
  clientAuth usage before applying the role. Each node certificate must include
  galera_node_address as an IP subject alternative name for SST peer
  verification. Files and parent directories must be accessible to mysql,
  including SELinux or AppArmor policy.
- Each node needs a reachable IPv4 address. Permit TCP 4567, 4568 and 4444 and
  UDP 4567 between cluster members only. The role currently supports IPv4
  replication and SST; DNS seed names must resolve to IPv4 addresses.
- Apply with strategy linear, serial 1 and any_errors_fatal true. For an initial
  cluster, process the bootstrap node first. Use at least three nodes for quorum
  resilience; retain a Primary component during rolling changes.

## Dependencies

```yaml
collections:
  - name: ansible.posix
    version: '>=2.0.0'
  - name: community.general
    version: '>=12.0.0'
  - name: community.crypto
    version: '>=3.0.0'
  - name: ansible.mariadb
    version: '>=6.0.0,<7.0.0'
roles:
  - name: jomrr.mariadb
    src: https://github.com/jomrr/ansible-role-mariadb.git
    scm: git
    version: main
```

## Role Variables

### `galera_backup`

Type: `bool`. Required: `false`.

Back up the previous Galera configuration before replacing it, including the
distribution file on RPM systems.

Default:

```yaml
galera_backup: true
```

### `galera_cluster_name`

Type: `str`. Required: `true`.

Shared cluster identifier containing letters, digits, dots, underscores or
hyphens.

### `galera_cluster_members`

Type: `list`. Required: `true`.

Nonempty list of cluster seed IPv4 addresses or DNS names, without ports. Never
use an empty gcomm address.

### `galera_node_address`

Type: `str`. Required: `true`.

Local IPv4 address reachable by every cluster member for replication, IST and
SST.

### `galera_node_name`

Type: `str`. Required: `false`.

Unique node identifier; defaults to the inventory hostname.

Default:

```yaml
galera_node_name: '{{ inventory_hostname }}'
```

### `galera_bootstrap`

Type: `bool`. Required: `false`.

Explicitly initialize a new cluster on this node only if no grastate.dat exists.
Not a disaster recovery switch.

Default:

```yaml
galera_bootstrap: false
```

### `galera_tls_ca_file`

Type: `path`. Required: `true`.

Existing PEM CA bundle trusted by all nodes, readable by mysql. Absolute path
without whitespace or option delimiters.

### `galera_tls_cert_file`

Type: `path`. Required: `true`.

Existing PEM certificate with server and client TLS usage and
galera_node_address as an IP subject alternative name.
Must be signed by the cluster CA and readable by mysql.

### `galera_tls_key_file`

Type: `path`. Required: `true`.

Existing unencrypted PEM private key, restricted to root and mysql. Absolute
path without whitespace or option delimiters.

### `galera_gcache_size`

Type: `str`. Required: `false`.

Galera write-set cache size with a K, M or G suffix. Changing it restarts the
node.

Default:

```yaml
galera_gcache_size: 128M
```

### `galera_sst_donor`

Type: `str`. Required: `false`.

Comma-separated preferred node names. A trailing comma permits other donors;
empty selects any suitable donor.

Default:

```yaml
galera_sst_donor: ''
```

## Managed Files

- `Debian and Ubuntu: /etc/mysql/mariadb.conf.d/99-galera.cnf.`
- `AlmaLinux, Fedora and openSUSE: /etc/my.cnf.d/galera.cnf.`
- `openSUSE: /etc/systemd/system/mariadb.service.d/galera.conf.`
- `The dependency also manages its MariaDB server snippet and database
  security.`

## Check Mode

Supported on an installed, running cluster; first-install check mode cannot
query a simulated MariaDB server.

- Configuration and service changes are predicted. Bootstrap is not executed and
  readiness polling is skipped.
- Read-only data-directory discovery runs in check mode to locate the native
  Galera state file.

## Service Behavior

Installs and starts MariaDB, configures Galera, then bootstraps a fresh
designated node or joins an existing Primary component. An unchanged run does
not restart the service. Configuration changes restart only the current node.

### Handlers

- Reload systemd before starting or restarting an openSUSE node when its service
  override changes.
- Restart MariaDB when the Galera snippet changes, except during the explicitly
  requested first bootstrap.

## Security Notes

- Replication uses socket.ssl with CA verification; SST separately uses
  mariabackup encrypt=3 and the same CA and credentials. Only trusted cluster
  members should hold certificates from this CA. Client SQL TLS remains
  controlled by mariadb_tls_ variables.
- SST authenticates locally as mysql using unix_socket with backup privileges;
  no SST password is stored. The role owns mysql@localhost authentication and
  grants. Root retains the dependency's local socket authentication.
- The dependency keeps client TCP bound to loopback by default. Set
  mariadb_bind_address and the dependency's TLS inputs explicitly when
  applications connect remotely. Galera and SST use galera_node_address
  independently.

## Operational Notes

- On openSUSE the service override connects the native galera_new_cluster
  environment file to mariadbd, retaining the distribution's database
  installation and upgrade hooks. The default single instance and Unix socket
  are preserved.
- galera_backup defaults to true. Replacing a changed configuration creates a
  timestamped backup through the template module, including the original
  distribution galera.cnf on RPM systems. Set galera_backup to false to disable
  these backups.
- Install requirements.yml with ansible-galaxy role install -r requirements.yml
  and collections.yml with ansible-galaxy collection install -r collections.yml.
  The complete MariaDB role is fetched directly from its GitHub repository at
  main and installed under the local name jomrr.mariadb; no Galaxy publication
  is required. Galera executes its tasks after validating cluster inputs. Set
  mariadb_ variables in inventory or the playbook to override the dependency's
  defaults.
- Set galera_bootstrap true on exactly one node for initial creation, with all
  other members new or stopped. Remove the flag after creation. The role never
  rewrites grastate.dat, safe_to_bootstrap or the stored cluster UUID. An
  existing state file prevents bootstrap even when the flag remains true. Never
  remove it to force recovery.
- After a complete outage, recover the most advanced safe node using MariaDB's
  documented recovery procedure before applying this role. It deliberately fails
  when normal startup cannot join a Primary component.
- Joining a cluster can replace a node's local data through SST. Start with
  empty joiners and preserve backups of any existing databases. Put application
  database and account declarations on an established cluster after its initial
  formation.
- The mariabackup donor remains available apart from a brief backup lock.
  Major-version upgrades can require a different transfer procedure and are
  outside this role's lifecycle.
- The native mariadbd --defaults-file validator reads the candidate snippet
  before installation. Provider and SST TLS options are also checked during
  startup and actual transfers; the validator does not validate every external
  snippet.
- Certificate renewal at unchanged paths requires an externally coordinated
  rolling restart. Retain quorum and wait for each node to become Synced before
  proceeding. Readiness here checks the local node, not the final expected
  cluster size.
- The community.crypto collection creates disposable certificates for the
  Molecule scenarios.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Three-node cluster inventory

Certificates must already be present. Process the bootstrap node first.
Remove its bootstrap flag after the cluster has formed.

```yaml
---
all:
  children:
    database_cluster:
      hosts:
        db1:
          galera_node_address: 10.20.30.11
          galera_bootstrap: true
        db2:
          galera_node_address: 10.20.30.12
        db3:
          galera_node_address: 10.20.30.13
      vars:
        galera_cluster_name: application
        galera_cluster_members: [10.20.30.11, 10.20.30.12, 10.20.30.13]
        galera_tls_ca_file: /etc/mysql/tls/ca.crt
        galera_tls_cert_file: /etc/mysql/tls/node.crt
        galera_tls_key_file: /etc/mysql/tls/node.key
```

### Initialize or update nodes sequentially

Retain this execution order for initial convergence and rolling configuration updates.

```yaml
---
- name: Configure the database cluster
  hosts: database_cluster
  gather_facts: true
  strategy: linear
  serial: 1
  any_errors_fatal: true
  roles:
    - role: jomrr.galera
```

## References

- [Galera installation](https://mariadb.com/docs/galera-cluster/galera-management/installation-and-deployment/getting-started-with-mariadb-galera-cluster)
- [MariaDB backup SST](https://mariadb.com/docs/galera-cluster/high-availability/state-snapshot-transfers-ssts-in-galera-cluster/mariadb-backup-sst-method)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
