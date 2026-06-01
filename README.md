# m2vcheck

A small Bash script to verify that a Debian server (10 → 13) meets the
official system requirements for a given Adobe Commerce / Magento 2
version.

It detects every relevant component (PHP, Composer, MariaDB, MySQL,
Redis, Valkey, OpenSearch, Elasticsearch, RabbitMQ, Varnish, Nginx,
Apache, PHP extensions) and compares each detected version against the
spec [published by Adobe](https://experienceleague.adobe.com/en/docs/commerce-operations/installation-guide/system-requirements).

```text
╭────────────────────────────────────────────────────────────────╮
│  m2vcheck · Adobe Commerce / Magento 2.4.9                     │
│  Debian GNU/Linux 12 (bookworm) · prod-web-01                  │
╰────────────────────────────────────────────────────────────────╯

▸ Langage & outils
  ✓ PHP            8.5.6          ┊ 8.5
  ✓ Composer       2.9.4          ┊ 2.9.3+

▸ Base de données  ✓ groupe couvert
  ✓ MariaDB        11.8.2         ┊ 11.8,12.3
  ○ MySQL          —              ┊ 8.4

...

────────────────────────────────────────────────────────────────
  ✓ CONFORME  Magento 2.4.9  · 12 ok · ext: 22/22
────────────────────────────────────────────────────────────────
```

## Why

Adobe ships a long matrix of supported versions per Magento release.
Auditing a fleet of servers by hand against that matrix is tedious and
error-prone. `m2vcheck` reduces this to one command and a clear
pass/fail verdict, with the right components grouped (DB, cache, search,
web) and missing pieces highlighted.

## Install

The script has no runtime dependencies beyond what ships with a default
Debian server: `bash`, `grep -P`, `curl`, `sort -V`, `dpkg-query`. No
sudo required to run a check.

```bash
sudo curl -fsSL https://raw.githubusercontent.com/mabt/m2vcheck/main/m2vcheck \
     -o /usr/local/bin/m2vcheck && sudo chmod +x /usr/local/bin/m2vcheck
```

Or clone and copy:

```bash
git clone https://github.com/mabt/m2vcheck.git
sudo install -m 755 m2vcheck/m2vcheck /usr/local/bin/m2vcheck
```

## Usage

```bash
m2vcheck 2.4.9     # check the current server against Magento 2.4.9 requirements
m2vcheck --list    # list supported Magento versions
m2vcheck --help    # usage
```

### Exit codes

| Code | Meaning                                              |
| ---- | ---------------------------------------------------- |
| `0`  | Server is compliant                                  |
| `1`  | At least one required component is missing or wrong  |
| `2`  | Usage error (unknown Magento version, missing arg)   |

This makes `m2vcheck` easy to drop into CI, Ansible, or a provisioning
script:

```yaml
- name: Verify server matches Magento 2.4.8 requirements
  ansible.builtin.command: m2vcheck 2.4.8
  changed_when: false
```

## What is checked

For each Magento version (2.4.4 → 2.4.9), the script verifies:

- **Language & tooling** — PHP, Composer
- **Database** (at least one) — MariaDB, MySQL
- **Cache / sessions** (at least one) — Valkey, Redis
- **Search engine** (at least one) — OpenSearch, Elasticsearch
- **Web server** (at least one) — Nginx, Apache
- **Optional components** — RabbitMQ, Varnish
- **PHP extensions** — `bcmath`, `ctype`, `curl`, `dom`, `fileinfo`,
  `gd`, `hash`, `iconv`, `intl`, `mbstring`, `openssl`, `pcre`,
  `pdo_mysql`, `simplexml`, `soap`, `sockets`, `sodium`, `tokenizer`,
  `xmlwriter`, `xsl`, `zip`, `opcache`

Markers:

| Glyph | Meaning                                           |
| ----- | ------------------------------------------------- |
| `✓`   | Installed and compliant                           |
| `✗`   | Installed but wrong version                       |
| `○`   | Absent (blocking only if no alternative covers it)|
| `⚠`   | Installed but not listed by Adobe for this version|

## Detection notes

- **OpenSearch / Elasticsearch** are detected via the local REST API on
  `http://localhost:9200/`. If your instance listens elsewhere or
  requires TLS / auth, the search-engine check will report `○` even
  when the service is running. Edit the `detect_opensearch` /
  `detect_elasticsearch` functions to point at your endpoint.
- **MariaDB vs MySQL** are disambiguated by parsing `mysql --version`
  output (`Distrib X.Y.Z-MariaDB`).
- **RabbitMQ** is queried via `dpkg-query` first (no sudo, no daemon
  startup) and falls back to `rabbitmqctl version`.
- **PHP extensions** are read from `php -m` (CLI). If the CLI SAPI
  differs from FPM, install the same extensions for both.

## Adding a new Magento version

Open `m2vcheck` and add a new branch to the `requirements()` `case`
statement at the top:

```bash
2.4.10)
    cat <<EOF
php=8.5,8.6
composer=2.9.3+
mariadb=12.3
mysql=8.4
opensearch=3
valkey=9
rabbitmq=4.2
varnish=8
nginx=1.28
apache=2.4
EOF
    ;;
```

Spec syntax:

- `X.Y` — installed `major.minor` must match
- `X` — installed `major` must match
- `X.Y.Z+` — installed version must be ≥ `X.Y.Z`
- Comma-separated list — any item matching is enough
- A component **absent** from the list means "not supported / not
  listed" for that Magento version; presence on the server will be
  flagged with `⚠`

## Source of truth

All version data comes from Adobe's official page (updated for each
patch release):

<https://experienceleague.adobe.com/en/docs/commerce-operations/installation-guide/system-requirements>

The script prints this URL at the bottom of every run.

## License

MIT
