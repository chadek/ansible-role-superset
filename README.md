ansible-role-superset
=====================

Installs and configures [Apache Superset](https://superset.apache.org) in
production on bare metal: a virtualenv built from PyPI (no docker, no
`superset run`), `superset_config.py`, and gunicorn, a celery worker and celery
beat driven by systemd.

| unit                      | where                  | role                                           |
| ------------------------- | ---------------------- | ---------------------------------------------- |
| `superset.service`        | every host             | gunicorn serving `superset.app:create_app()`   |
| `superset-worker.service` | every host             | celery worker (SQL Lab async, caches, reports) |
| `superset-beat.service`   | the **primary** only   | celery beat scheduler                          |

The two stores Superset needs are handled as follows:

* **PostgreSQL** — the metadata database. Point the role at a managed service,
  or set `superset_manage_postgresql: true` and the role calls
  `geerlingguy.postgresql` with the database, the login role, the `pg_hba`
  entries, the listen address and the authentication method derived from the
  `superset_db*` variables.
* **Valkey** — celery broker, results backend and caches. Point the role at a
  managed service, or set `superset_manage_valkey: true` and the role installs
  the distribution package and writes `valkey.conf` itself. There is no
  community Valkey role worth depending on (the only one published on galaxy is
  unmaintained), and there is little to do beyond a package and a config file.

**Not in scope:** the reverse proxy in front of Superset. Give that to
`chadek.nginx_certbot`, `load_balancer`, or whatever already terminates TLS in
your infrastructure — see the example playbook below.

Requirements
------------

* A Debian or Ubuntu target with a python 3.10 → 3.12 interpreter available.
  Debian 12 ships 3.11 and Ubuntu 24.04 ships 3.12, both fine as they are.
  **Ubuntu 26.04 ships python 3.14**, which apache-superset 6.x does not support
  (it also caps `greenlet` to 3.1.1, which has no wheel for it): add the
  Canonical backports PPA and select a supported interpreter, which is checked
  before the virtualenv is built.

  ```yaml
  superset_python_ppa: "ppa:canonical-python-maintainers/python-backports"
  superset_python_interpreter: "python3.12"
  ```

  A virtualenv does not help here: `python -m venv` reuses the very interpreter
  that created it, so it isolates site-packages, not the runtime. The C
  extensions still have to match the CPython ABI.
* `geerlingguy.postgresql` and the `community.postgresql` collection, only when
  `superset_manage_postgresql` is true.
* `valkey-server` has to be installable: it is in Debian 13, in
  `bookworm-backports` (the role enables that suite itself on Debian 12), in
  Ubuntu universe from 24.04 and in main from 26.04. Only needed when
  `superset_manage_valkey` is true.
* The `community.crypto` collection and the `cryptography` python library on the
  target, only when `superset_manage_valkey_tls` is true: that is what generates
  the CA and the server certificate.

Things you must know before running it
--------------------------------------

* **The metadata database has to be shared** by every Superset host and every
  celery worker of an instance: SQLite is rejected by the preflight checks.
* **`superset_secret_key` has to be identical on every host.** It signs the
  session cookies *and* encrypts the database connection credentials stored in
  the metadata database. The role refuses to run if the hosts of
  `superset_cluster_group` disagree. Generate one with `openssl rand -base64 42`.
* Because that key is shared, **no sticky session is needed** on the reverse
  proxy.
* **Only one celery beat** may run for a given metadata database, otherwise
  scheduled reports fire several times. The role elects the first host of
  `superset_cluster_group`, runs beat there and stops it everywhere else. That
  same host is the only one running `superset db upgrade`. A standalone host
  needs no group: it is its own primary.
* A play declaring `gather_facts: false` is fine: the role gathers the minimal
  fact subset itself when `ansible_os_family` is missing, which is what the OS
  specific vars, the interpreter check and `geerlingguy.postgresql` need.
* The `ALERT_REPORTS` feature flag is on and celery beat schedules
  `reports.scheduler`, but **rendering report screenshots needs a headless
  chromium and a webdriver**. Install them and declare the `WEBDRIVER_*`
  settings through `superset_config_extra` if you rely on screenshots, or drop
  the flag from `superset_feature_flags`. Text only alerts work without it.
* `ENABLE_TEMPLATE_PROCESSING` is on, because an imported dashboard usually
  carries Jinja in the SQL of its datasets. Without it Superset hands the
  template to the SQL parser as it stands and the chart fails with `Unable to
  parse SQL / Error parsing near '{%'`. It also means a dataset author can run
  Jinja server side: drop the flag if that is not acceptable.
* The celery worker logs `Signal handler <function teardown ...> raised:
  RuntimeError('Working outside of application context.')` after every task.
  This is an upstream bug, not a misconfiguration: Superset wraps each task in
  an app context inside `AppContextTask.__call__`, so the context is already
  popped when `task_postrun` fires and `superset/tasks/celery_app.py` calls
  `db.session.remove()`. The task itself has completed, and Flask-SQLAlchemy has
  already torn the session down with that context, so the message is noise. With
  `ALERT_REPORTS` on, beat runs `reports.scheduler` every minute, hence one
  traceback a minute.

Role Variables
--------------

Everything lives in `defaults/main.yml`, the most useful ones being:

```yaml
# inventory group holding every superset host sharing one metadata database,
# used to elect the primary (schema migrations + celery beat)
superset_cluster_group: "superset"

superset_version: "6.1.0"
superset_python_interpreter: "python3"

# secrets, put them in an ansible-vault
superset_secret_key: "{{ vault_superset_secret_key | default('') }}"
superset_db:
  host: "127.0.0.1"
  port: 5432
  name: "superset"
  user: "superset"
  password: "{{ vault_superset_db_password | default('') }}"
superset_valkey:
  host: "127.0.0.1"
  port: 6379
  username: ""
  password: "{{ vault_superset_valkey_password | default('') }}"
  db_celery: 0
  db_results: 1
  db_cache: 2
  db_ratelimit: 3
superset_admin:
  username: "admin"
  firstname: "Superset"
  lastname: "Admin"
  email: "admin@example.com"
  password: "{{ vault_superset_admin_password | default('') }}"

# install the shared stores from this role (set them on one host only)
superset_manage_postgresql: false
superset_manage_valkey: false
superset_db_allowed_networks: []          # CIDRs added to pg_hba.conf
superset_valkey_bind_interface: "127.0.0.1"
superset_manage_valkey_tls: false         # local valkey serves TLS, CA generated here

superset_public_url: "https://superset.example.com"
superset_bind: { host: "0.0.0.0", port: 8088 }
superset_gunicorn: { workers: 4, worker_class: "gevent", timeout: 120, ... }
superset_celery: { enabled: true, pool: "prefork", concurrency: 4, ... }

# any raw python appended to superset_config.py
superset_config_extra: ""
```

Using a managed PostgreSQL service
----------------------------------

Leave `superset_manage_postgresql: false` and either fill the `superset_db`
fields or paste the connection string:

```yaml
superset_db_url: "postgresql://superset:secret@pg-xxxx.database.cloud.ovh.net:20184/superset"
superset_db:
  user: "superset"
  password: "{{ vault_superset_db_password }}"
  name: "superset"
  sslmode: "verify-full"       # require, verify-ca or verify-full on a managed endpoint
  ca_cert: "/usr/local/share/ca-certificates/pg-ca.crt"   # deploy it yourself
```

* SQLAlchemy needs an explicit driver, so `postgres://` and bare
  `postgresql://` are both rewritten to `postgresql+psycopg2://`.
* `sslmode` and `sslrootcert` are appended to the URI unless the pasted string
  already carries an `sslmode`.
* Preflight refuses a non-local database without a password or with
  `sslmode=disable`: that database holds the **encrypted credentials of every
  data source**, so it must never be reached in clear.
* `SQLALCHEMY_ENGINE_OPTIONS` already sets `pool_pre_ping` and a 300s
  `pool_recycle`, which is what you want against an endpoint that closes idle
  connections.

With both stores managed, the Superset hosts hold no durable state at all:
rebuilding one is a `molecule`-grade operation.

Valkey, local or managed
------------------------

Valkey is a fork of Redis 7.2.4 and speaks the same protocol, so everything on
the Superset side works unchanged — `redis-py`, `kombu`, `flask-caching` and
`cachelib` all talk to it happily, and pointing the role at a Redis instance
instead works just as well.

A local instance, installed and configured by the role:

```yaml
superset_manage_valkey: true
superset_valkey_bind_interface: "0.0.0.0"      # 127.0.0.1 for a standalone host
superset_valkey:
  host: "10.0.0.10"
  port: 6379
  password: "{{ vault_superset_valkey_password }}"   # required off loopback
```

It is deliberately plain: `noeviction`, snapshots on, `protected-mode`, and
either `requirepass` or — when `superset_valkey.username` is set — an ACL user
with the default one disabled. `superset_valkey_persistence`,
`superset_valkey_maxmemory` and `superset_valkey_config_extra` cover the rest.
It serves plaintext unless asked otherwise, so preflight rejects
`superset_valkey.tls: true` on its own: TLS from the local instance is
`superset_manage_valkey_tls`.

```yaml
superset_manage_valkey: true
superset_manage_valkey_tls: true              # worth it as soon as the bind
superset_valkey_bind_interface: "0.0.0.0"     # interface leaves the loopback
superset_valkey_tls_extra_names: []           # 'DNS:…' / 'IP:…' the other hosts dial
superset_valkey:
  host: "10.0.0.10"
  password: "{{ vault_superset_valkey_password }}"
  # tls and ca_cert are derived, setting them is an error
```

The role then generates a private CA and a server certificate under
`superset_valkey_tls_dir` (`/etc/valkey/tls`), closes the plaintext port
(`port 0` + `tls-port`) and switches the URLs to `rediss://`. Clients present no
certificate of their own (`tls-auth-clients no`), they authenticate with the ACL
password — TLS encrypts, it authenticates nobody. The certificate covers the
loopback names, the inventory hostname, the bind interface and
`superset_valkey.host`; add whatever the other hosts of the cluster dial to
`superset_valkey_tls_extra_names`, since a client verifies the certificate
against the name *it* used. Nothing is renewed while the files are there: delete
them and run the role again to roll over.

**The CA has to be readable by the superset user, and `/etc/valkey` is not.** The
Debian and Ubuntu package ships it as `0770 valkey:valkey`, which the `superset`
account cannot even traverse, so the role copies the CA to
`superset_valkey_ca_cert_file` (`/etc/superset/valkey-ca.crt`, `0640
root:superset`) and points the URLs at that copy. This is the one to distribute
to the other hosts of the cluster, which do not install valkey and need it as
their `superset_valkey.ca_cert`. Do not relax the permissions of the package
directory instead — a package upgrade puts them back.

That trap is worth knowing because of how it surfaces: `redis-py` and `psycopg2`
both report a CA they cannot open as a *connection* error, naming no file.

```text
redis.exceptions.ConnectionError: Error 13 connecting to 127.0.0.1:6379. Permission denied.
```

Errno 13 there is the failed `open()` of the CA, not a refused socket — the
`GET`/`SET` path only runs once a dashboard is opened, so the services start
clean and break on first use. `configure.yml` therefore probes every configured
CA as the superset user (`su -s /bin/sh superset -c 'test -r …'`) and fails the
play with the offending path.

A managed service, addressed by its connection string:

```yaml
superset_manage_valkey: false
# the connection string of the service, pasted as it is handed out
superset_valkey_url: "valkeys://superset:secret@valkey-xxxx.database.cloud.ovh.net:20184"
superset_valkey:
  username: "superset"
  password: "{{ vault_superset_valkey_password }}"
  tls: true
  ssl_cert_reqs: "required"
  ca_cert: "/usr/local/share/ca-certificates/valkey-ca.crt"   # deploy it yourself
  # this service exposes a single logical database
  db_celery: 0
  db_results: 0
  db_cache: 0
  db_ratelimit: 0
```

Three traps, all of which the role handles or checks:

* **The URL scheme must be `redis://` / `rediss://`.** OVHcloud hands out
  `valkeys://…`, and neither `redis-py` nor `kombu` registers a `valkey://`
  transport — celery cannot resolve it and the worker refuses to start. The role
  rewrites `valkeys://` to `rediss://` (and `valkey://` to `redis://`)
  automatically, so the provider string can be pasted verbatim.
* **Only `db 0` on some services.** Setting the four `db_*` to `0` is safe: the
  caches and the results backend already use distinct key prefixes
  (`superset_metadata_`, `superset_data_`, `superset_filter_`,
  `superset_explore_`, `superset_results`).
* **The eviction policy must be `noeviction`.** Managed caching offerings often
  default to, or recommend, `allkeys-lru`; since the policy is instance wide,
  LRU eviction can silently drop queued celery tasks (scheduled reports, async
  queries). Cache growth is bounded by the 24h `CACHE_DEFAULT_TIMEOUT` TTLs
  instead. Check it with `CONFIG GET maxmemory-policy`.

One thing to confirm with the provider: **the endpoint must not be cluster
mode.** Celery's Redis transport supports Sentinel but not Redis/Valkey Cluster.
A single HA endpoint with failover behind it is fine — verify with
`CLUSTER INFO` returning `cluster_enabled:0`.

The `managed-tls` molecule scenario reproduces the whole managed setup: a
separate host serving **TLS only PostgreSQL and TLS only Valkey**, both with an
ACL user, addressed through a `postgres://` and a `valkeys://` connection
string. It asserts that the schemes were normalised, that no database or cache
listens on the application host, that a plaintext client is refused by both
services, and that the broker, the caches, the results backend and the metadata
schema all work over TLS.

Extra python packages are installed in the same pip transaction as
`apache-superset`, so pip resolves a single consistent set. Add your database
drivers there:

```yaml
superset_pip_packages:
  - "psycopg2-binary"
  - "redis"           # the redis client is also the valkey client
  - "gunicorn"
  - "gevent<25"       # apache-superset 6.x caps greenlet to 3.1.1
  - "Pillow"
  - "rich"
  - "mysqlclient"     # to query MySQL/MariaDB sources
  - "pyhive[presto]"  # ...
```

Dependencies
------------

None declared in `meta/main.yml`: `geerlingguy.postgresql` is pulled with
`include_role` only where it is needed.

Example Playbook
----------------

Inventory:

```ini
[superset]
superset-app1
superset-app2
```

`group_vars/superset.yml` (or an ansible-vault):

```yaml
superset_cluster_group: "superset"
superset_public_url: "https://superset.example.com"
superset_secret_key: "{{ vault_superset_secret_key }}"
superset_db:
  host: "10.0.0.10"
  port: 5432
  name: "superset"
  user: "superset"
  password: "{{ vault_superset_db_password }}"
superset_valkey:
  host: "10.0.0.10"
  port: 6379
  password: "{{ vault_superset_valkey_password }}"
  db_celery: 0
  db_results: 1
  db_cache: 2
  db_ratelimit: 3
```

A single host, with its own PostgreSQL and Valkey on loopback and the vhost
delegated to `chadek.nginx_certbot`:

```yaml
- name: Superset
  hosts: superset-app1
  become: true
  vars:
    superset_manage_postgresql: true
    superset_manage_valkey: true
    superset_bind: { host: "127.0.0.1", port: 8088 }
  roles:
    - chadek.nginx_certbot
    - ansible-role-superset
```

Several hosts sharing the stores of the first one, which is also the primary —
that play comes first because the others wait on those stores:

```yaml
- name: Superset | Primary and shared stores
  hosts: superset-app1
  become: true
  vars:
    superset_manage_postgresql: true
    superset_manage_valkey: true
    superset_valkey_bind_interface: "0.0.0.0"
    superset_db_allowed_networks:
      - "10.0.0.0/24"
  roles:
    - role: ansible-role-superset

- name: Superset | Remaining hosts
  hosts: superset:!superset-app1
  become: true
  roles:
    - role: ansible-role-superset
```

Testing
-------

Three molecule scenarios:

```bash
source ~/venv/molecule/bin/activate

# all-in-one: superset + postgresql + valkey on one host, three distributions
molecule test -s default

# two superset hosts sharing the postgresql and valkey of the first one
molecule test -s cluster

# managed services lookalike: TLS only PostgreSQL and Valkey on a remote host
molecule test -s managed-tls
```

The `default` scenario runs three platforms in parallel: Debian 12, Ubuntu 24.04
and Ubuntu 26.04. The 26.04 one carries `superset_python_ppa` and
`superset_python_interpreter: python3.12`, so the interpreter workaround is
covered by the tests and not only by this README. Debian 12 covers the
`bookworm-backports` path of the Valkey install.

`default` checks that the three Superset units and Valkey run, that `/health` and the login
page answer, that the metadata schema and the admin user exist, that the celery
worker replies to a ping and that the installed version is the expected one.

`cluster` additionally checks that both hosts serve the application, that celery
beat runs on the first one **only**, that both workers joined the shared broker,
that the shared schema is migrated, and that the Valkey instance authenticates
its clients and keeps its snapshots.

License
-------

GPL-3.0-only

Author Information
------------------

Written for Green-house, inspired by the
[Apache Superset production deployment guide](https://github.com/NoneAvobeAll/Apache-Superset-Production-Deployment-Guide)
and by the official
[PyPI installation](https://superset.apache.org/admin-docs/installation/pypi/)
and [async queries](https://superset.apache.org/admin-docs/configuration/async-queries-celery/)
documentation.
