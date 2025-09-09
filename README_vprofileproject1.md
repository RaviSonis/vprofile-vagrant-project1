# Vagrant Multi-VM Java Web Stack (NGINX → Tomcat → Memcached → MySQL) + RabbitMQ

> Local infrastructure-as-code lab using **Vagrant + VirtualBox** with one VM per service.  
> Request flow: **NGINX (LB/Reverse Proxy)** → **Tomcat (Java app)** → **Memcached (cache) → MySQL (DB)**.  
> **RabbitMQ** is present as a dummy dependency.

This repo contains **both**:
- **Manual provisioning** (step-by-step, as in the course PDF)
- **Automated provisioning** (Vagrant + shell scripts)

---

## Repository Layout

```
vagrant/
├── Manual_provisioning/
│   ├── Vagrantfile
│   └── VprofileProjectSetup.pdf    # full manual steps
└── Automated_provisioning/
    ├── Vagrantfile
    ├── application.properties
    ├── mysql.sh     # DB
    ├── memcache.sh  # Cache
    ├── rabbitmq.sh  # MQ
    ├── tomcat.sh    # App server (CentOS)
    ├── tomcat_ubuntu.sh  # (alt App server on Ubuntu, optional)
    └── nginx.sh     # Reverse proxy / LB
ansible/             # optional: Tomcat setup & app deployment playbooks
pom.xml              # Java/Maven app (vprofile)
src/                 # Java source
Jenkinsfile          # optional CI
```

---

## Prerequisites

- **VirtualBox** 7+
- **Vagrant** 2.3+
- **Plugin**: `vagrant-hostmanager`
  ```bash
  vagrant plugin install vagrant-hostmanager
  ```
- **Git** (Git Bash on Windows is fine)
- **Java & Maven** (only if you plan to build the WAR locally): JDK 17/21, Maven 3.9+
- **Ansible** (optional if you want to use the playbooks in `ansible/`)

> Networking uses the host-only range **192.168.56.0/24**. Make sure it exists in VirtualBox.

---

## Architecture

### High-level request flow
```text
                   (User Browser)
                         |
                         v
                 +----------------+
                 |    NGINX LB    |   192.168.56.11 (web01)
                 |  Reverse Proxy |
                 +----------------+
                         |
            HTTP (/:80)  |  proxy_pass -> app01:8080
                         v
                 +----------------+
                 |    Tomcat      |   192.168.56.12 (app01)
                 |   Java App     |
                 +----------------+
                      |      \
     Cache check ---> |       \  (optional / dummy)
                      v        v
               +------------+  +----------------+
               | Memcached  |  |   RabbitMQ     |
               | :11211     |  | :5672 (AMQP)   |
               | 192.168.   |  | 192.168.       |
               | 56.14 mc01 |  | 56.13/16 rmq01 |
               +------------+  +----------------+
                      |
          on miss --->|  SQL query
                      v
               +----------------+
               |    MySQL       |
               | :3306          |
               | 192.168.56.15  |
               |     db01       |
               +----------------+

Notes:
- First identical request: Tomcat -> MySQL, then value cached in Memcached.
- Subsequent identical requests: Tomcat -> Memcached (cache hit).
- RabbitMQ is included to mirror enterprise stacks but not essential here.
```

### VM map with roles & ports
```text
+--------------------+----------------------+-------------------------+---------------------+
| Hostname           | IP Address           | Role                    | Key Ports           |
+--------------------+----------------------+-------------------------+---------------------+
| web01 (Ubuntu)     | 192.168.56.11        | NGINX (LB/Reverse Proxy)| 80 (HTTP)           |
| app01 (CentOS)     | 192.168.56.12        | Tomcat (Java App)       | 8080                |
| mc01  (CentOS)     | 192.168.56.14        | Memcached (Cache)       | 11211               |
| db01  (CentOS)     | 192.168.56.15        | MySQL/MariaDB (DB)      | 3306                |
| rmq01 (CentOS)     | 192.168.56.13 (manual) / 192.168.56.16 (auto)   | 5672 (AMQP), 15672* |
+--------------------+----------------------+-------------------------+---------------------+

*15672 = RabbitMQ Management UI (optional if plugin enabled).
---

## Choose Your Path

### 1) Manual Provisioning

- **Vagrantfile:** `vagrant/Manual_provisioning/Vagrantfile`
- **Reference PDF:** `vagrant/Manual_provisioning/VprofileProjectSetup.pdf`

**VM plan (manual):**
| VM | IP |
|---|---|
| web01 (NGINX, Ubuntu 22.04) | `192.168.56.11` |
| app01 (Tomcat, CentOS Stream 9) | `192.168.56.12` |
| rmq01 (RabbitMQ, CentOS Stream 9) | `192.168.56.13` |
| mc01 (Memcached, CentOS Stream 9) | `192.168.56.14` |
| db01 (MySQL/MariaDB, CentOS Stream 9) | `192.168.56.15` |

**Steps**
```bash
cd vagrant/Manual_provisioning
vagrant up
# then follow the PDF step-by-step (install packages, configure services, seed DB, deploy WAR, etc.)
```

**What you'll do (high level)**
- Install & configure **MySQL/MariaDB** on `db01`  
  - DB **name**: `accounts`; example user: `admin` / `admin123`  
  - Seed data from `src/main/resources/db_backup.sql` (or `accountsdb.sql`)
- Install & run **Memcached** on `mc01` (listen on `0.0.0.0:11211`)
- Install **RabbitMQ** on `rmq01` (AMQP `5672`; mgmt console `15672` if enabled)
- Install **Java + Tomcat** on `app01`, deploy **`ROOT.war`**  
  - Build: `mvn -DskipTests package` → `target/vprofile-v2.war`
  - Copy WAR to `/usr/local/tomcat/webapps/ROOT.war`
- Configure **NGINX** on `web01` to reverse proxy to `app01:8080`

Open: `http://192.168.56.11/`

> First login hits **MySQL**; repeated requests are served from **Memcached**.

---

### 2) Automated Provisioning (one command per VM)

- **Vagrantfile:** `vagrant/Automated_provisioning/Vagrantfile`
- **Provisioners:** Bash scripts in the same folder

**VM plan (automated):**
| VM | IP |
|---|---|
| web01 (NGINX, Ubuntu 22.04) | `192.168.56.11` |
| app01 (Tomcat, CentOS Stream 9) | `192.168.56.12` |
| rmq01 (RabbitMQ, CentOS Stream 9) | `192.168.56.16` |
| mc01 (Memcached, CentOS Stream 9) | `192.168.56.14` |
| db01 (MySQL/MariaDB, CentOS Stream 9) | `192.168.56.15` |

**Run**
```bash
cd vagrant/Automated_provisioning
vagrant up
```

**What the scripts do**
- `mysql.sh` → installs MariaDB, creates DB `accounts`, user `admin` / `admin123`, opens `3306`
- `memcache.sh` → installs Memcached, listens on `11211`
- `rabbitmq.sh` → installs Erlang + RabbitMQ, opens `5672` (& `15672` if mgmt enabled)
- `tomcat.sh` → installs Java + Tomcat, deploys app (if WAR present), opens `8080`
- `nginx.sh` → installs NGINX, proxies `/` to `app01:8080`, opens `80`
- `application.properties` → app-level endpoints (DB, cache, MQ)

**Validate**
```bash
# From your host
curl -I http://192.168.56.11/          # Expect 200 from NGINX → Tomcat
curl -s http://192.168.56.12:8080/ | head -n1
nc -zv 192.168.56.14 11211             # Memcached
nc -zv 192.168.56.15 3306              # MySQL
nc -zv 192.168.56.16 5672              # RabbitMQ (automated build)
```

---

## Cache + DB Flow (how to see it)

```bash
# On mc01
vagrant ssh mc01
echo stats | nc localhost 11211 | grep -E 'curr_items|get_|hit|miss'

# On app01 (hit the app twice from your browser between checks)
```

You should see `get_hits` increase on repeated requests after the first DB read.
---

## Credits

- Original app: **/hkhcoder/vprofile-project** sample (Java/Spring, MySQL, Memcached, RabbitMQ)
- Vagrant multi-VM lab & documentation: your implementation (this repo)
