# Docker Desktop with Zabbix

![Alt](image/docker.webp)

## Setup docker desktop in windows 11

Since you are using Windows 11, here is a simple guide to install and start using Docker Desktop.

### Check WSL 2

Open PowerShell as Administrator and run:

```powershell
wsl --install
```

Restart your computer if requested.

Then check:

```powershell
wsl --status
```

You should see that WSL 2 is available.

### Install Docker Desktop

Download Docker Desktop from:

Docker Desktop for Windows from website

Run the installer.
Make sure Use WSL 2 instead of Hyper-V is selected.
Complete the installation.
Restart Windows if requested.
Open Docker Desktop.

Wait until Docker Desktop shows that the Docker Engine is running.

### Docker Desktop + WSL

recommend enabling Docker integration with Debian or ubuntu.

In Docker Desktop:

Settings → Resources → WSL Integration

Enable:

✅ Enable integration with my default WSL distro
✅ Debian/or ubuntu

### Install Ubuntu

Open PowerShell as Administrator and run:

```powershell
wsl --install -d Ubuntu
```

To see a list of all available Linux distributions that can be installed from the Microsoft Store or via command line, use:

```powershell
wsl --list --online
```

Wait for the installation to finish. Then restart your computer if Windows asks you to.

After installation

Open Ubuntu from the Start Menu, or run:

```powershell
wsl -d Ubuntu
```

The first time Ubuntu opens, it will ask you to create:

1. A Linux username
2. A password

Then check again in PowerShell:

```powershell
wsl -l -v
```

You should see something like:

```markdown
NAME              STATE           VERSION
Ubuntu            Running         2
docker-desktop    Running         2
```

After Ubuntu is installed, we can configure Docker Desktop + Ubuntu integration step by step.

Then, inside Debian/ubuntu, test:

```powershell
wsl -d ubuntu
docker --version
docker ps
```

or stop wsl:

```powershell
wsl --shutdown
```

Important

You generally do not need to install Docker Engine separately inside Debian when using Docker Desktop with WSL integration.

The recommended structure is:

```markdown
Windows 11
│
├── Docker Desktop
│    └── Docker Engine
│
└── WSL 2
     └── Debian
          ├── Docker CLI
          ├── Docker Compose
          └── Your project files
```

### Using Docker Compose test

Create a folder:

```powershell
mkdir my-docker-project
cd my-docker-project
```

Create a file named:

```powershell
nano compose.yaml
```

Example:

```yml
services:
  nginx:
    image: nginx:latest
    container_name: my-nginx
    ports:
      - "8080:80"
    restart: unless-stopped
```

Start it:

```powershell
docker compose up -d
```

Check:

```powershell
docker compose ps
```

Stop and remove the containers:

```powershell
docker compose down
```

### Test Docker

Open PowerShell and run:

```powershell
docker --version
```

Then:

```powershell
docker run hello-world
```

If successful, Docker will download a test container and display a success message.

## new zabbix docker compose

For your new Zabbix Docker 7.4.13, with PostgreSQL 17.10 and the new Zabbix Docker network 172.26.0.0/16, use this docker-compose.yml.

```markdown
OLD Zabbix 7.4.11
    │
    └── OLD PostgreSQL 17.10
             │
             │ pg_dump
             ▼
      zabbix_backup.dump
             │
             │ pg_restore
             ▼
NEW PostgreSQL 17.10
             │
             ▼
NEW Zabbix 7.4.13
             │
             ▼
NEW Zabbix Web
```

New Docker network:

172.26.0.0/16

Docker default bridge:

172.28.0.0/16

Existing Docker networks remain untouched.

clean, simple new Zabbix installation with just:

- One zabbix folder
- One docker-compose.yml
- PostgreSQL 17.10
- Zabbix 7.4.13
- Web interface
- Docker network 172.26.0.0/16

Use this simple setup.

### Create folder

```bash
mkdir zabbix
cd zabbix
```

### Create Compose file

```bash
nano docker-compose.yml
```

Paste:

```yaml
services:

  postgres:
    image: postgres:17.10
    container_name: zabbix-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ChangeThisPassword
    volumes:
      - postgres_data:/var/lib/postgresql/data

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-7.4.13
    container_name: zabbix-server
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ChangeThisPassword
    ports:
      - "10051:10051"

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-7.4.13
    container_name: zabbix-web
    restart: unless-stopped
    depends_on:
      - postgres
      - zabbix-server
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ChangeThisPassword
      ZBX_SERVER_HOST: zabbix-server
      PHP_TZ: Asia/Phnom_Penh
    ports:
      - "8080:8080"

networks:
  default:
    name: zabbix_default
    driver: bridge
    ipam:
      config:
        - subnet: 172.26.0.0/16

volumes:
  postgres_data:
    name: zabbix_postgres_data
```

Save and exit.

### Start

```bash
docker compose up -d
```

### Check

```bash
docker ps
```

You should see:

```markdown
zabbix-postgres
zabbix-server
zabbix-web
```

Check Zabbix IP:

```bash
docker inspect zabbix-server --format '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}: IP={{$v.IPAddress}} Gateway={{$v.Gateway}}{{end}}'
```

It should be:

```markdown
zabbix_default: IP=172.26.0.x Gateway=172.26.0.1
```

### Open Zabbix

```markdown
http://localhost:8080
```

Default login:

```markdown
Username: Admin
Password: zabbix
```

This creates a new empty Zabbix database. After it is working, we can restore your old Zabbix 7.4.11 database into this installation.

## new zabbix and restore postgres database

Yes. Let's start from the beginning, assuming:

You are using Docker Desktop/WSL.
Your old PostgreSQL = 17.10.
You already created a folder called zabbix.
You want to create a fresh backup and restore it into the new PostgreSQL 17.10.
Your current zabbix_backup.dump is bad/incomplete/outdated, so we will not use it.

Step 1 — Go to your zabbix folder

In WSL/Linux:

```bash
cd ~/zabbix
```

Check:

```bash
pwd
ls -lh
```

Step 2 — Find your Docker containers in both docker

Run:

```bash
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

You need to identify:

```markdown
OLD PostgreSQL container
NEW PostgreSQL container
```

For example:

```markdown
zabbix-postgres-old    postgres:17.10
zabbix-postgres        postgres:17.10
```

Don't continue until you know which one is OLD and which one is NEW.

### Part A — Create a fresh backup from OLD PostgreSQL

Step 3 — Check the old database

Replace OLD_POSTGRES with your old PostgreSQL container name:

- edit OLD_POSTGRES name to match your zabbix_postgres container
  
```bash
docker exec -it OLD_POSTGRES \
psql -U zabbix -d zabbix \
-c "SELECT COUNT(*) FROM hosts;"
```

You should get something like:

```markdown
 count

-------
    50
```

This confirms your old devices are still there.

Step 4 — Create the new backup

Make sure you are inside your zabbix folder:

```bash
cd ~/zabbix
```

Then:

```bash
docker exec OLD_POSTGRES \
pg_dump -U zabbix -Fc zabbix > zabbix_backup_new.dump
```

For example, if your old container is called zabbix-postgres:

```bash
docker exec zabbix-postgres \
pg_dump -U zabbix -Fc zabbix > zabbix_backup_new.dump
```

Wait until the command finishes.

Step 5 — Check the backup

```bash
ls -lh zabbix_backup_new.dump
```

You should see something like:

```markdown
-rw-r--r-- 1 ... 600M ... zabbix_backup_new.dump
```

The size can be different from your old 544 MB backup.

### Part B — Test the backup BEFORE restoring

Step 6 — Test the backup

Run:

```bash
docker run --rm \
-v "$(pwd):/backup" \
postgres:17.10 \
pg_restore -l /backup/zabbix_backup_new.dump | head -20
```

If successful

You should see something similar to:

```markdown
;
; Archive created at ...
;
;     dbname: zabbix
;     TOC Entries: ...
;
...
TABLE ...
TABLE DATA ...
```

This is very important.

Don't restore until this command works.

### Part C — Prepare NEW PostgreSQL

Now we work with the new PostgreSQL.

Step 7 — Make sure the new PostgreSQL is running

```bash
docker start zabbix-postgres
```

If it is already running, that's fine.

Check:

```bash
docker exec -it zabbix-postgres psql --version
```

You want:

```markdown
psql (PostgreSQL) 17.10
```

Step 8 — Stop Zabbix Server

Before replacing the database, stop the new Zabbix server:

```bash
docker stop zabbix-server
```

If your Zabbix server has another container name, use that name.

or

go to zabbix folder:

```bash
cd ~/zabbix
docker compose down
```

then:

```bash
docker network rm zabbix_default
```

- If zabbix_default network not duplicate with internal network, do not need to remove.
- Because if duplicate added devices to zabbix from that range will not work.
- If it says the network doesn't exist, that's okay.

Now start only PostgreSQL:

```bash
docker compose up -d zabbix-postgres
```

Check:

```bash
docker ps
```

You should see:

```markdown
zabbix-postgres
```

Verify postgreSQL

Check the version:

```bash
docker exec -it zabbix-postgres psql --version
```

Expected:

```markdown
psql (PostgreSQL) 17.10
```

Check the network:

```bash
docker inspect zabbix-postgres --format '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}: IP={{$v.IPAddress}} Gateway={{$v.Gateway}}{{end}}'
```

Expected something like:

```markdown
zabbix_default: IP=172.26.0.2 Gateway=172.26.0.1
```

### Part D — Create an empty database

Step 9 — Connect to PostgreSQL

Important: connect to postgres, not zabbix.

```bash
docker exec -it zabbix-postgres \
psql -U zabbix -d postgres
```

You should see:

```markdown
postgres=#
```

Step 10 — Remove connections to zabbix

Inside PostgreSQL:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'zabbix'
AND pid <> pg_backend_pid();
```

Then:

```sql
DROP DATABASE zabbix;
```

Then:

```sql
CREATE DATABASE zabbix OWNER zabbix;
```

Then:

```sql
\q
```

Now you have:

```markdown
NEW PostgreSQL 17.10
        ↓
EMPTY database
        ↓
zabbix
```

### Part E — Copy the NEW backup

Step 11 — Copy backup into the new PostgreSQL container

From your zabbix folder:

```bash
docker cp zabbix_backup_new.dump \
zabbix-postgres:/tmp/zabbix_backup.dump
```

Check:

```bash
docker exec -it zabbix-postgres \
ls -lh /tmp/zabbix_backup.dump
```

The size should match your host file.

For example:

```markdown
Host:
600M zabbix_backup_new.dump

Container:
600M /tmp/zabbix_backup.dump
```

### Part F — Restore

Step 12 — Restore the database

Now:

```bash
docker exec -it zabbix-postgres \
pg_restore \
-U zabbix \
-d zabbix \
--verbose \
/tmp/zabbix_backup.dump
```

This can take some time because your database is large.

You will see many lines such as:

```markdown
processing item ...
creating TABLE ...
processing data for table ...
Don't stop it while it is running.
```

### Part G — Verify your devices

Step 13 — Check hosts

After pg_restore finishes:

```bash
docker exec -it zabbix-postgres \
psql -U zabbix -d zabbix \
-c "SELECT COUNT(*) FROM hosts;"
```

Then:

```bash
docker exec -it zabbix-postgres \
psql -U zabbix -d zabbix \
-c "SELECT hostid, host, name FROM hosts LIMIT 20;"
```

You should see your old devices.

### Part H — Start Zabbix 7.4.13

Step 14 — Start Zabbix

```bash
docker start zabbix-server
```

Or if you're using Compose:

```bash
docker compose up -d
```

Then check:

```bash
docker logs -f zabbix-server
```

Your complete process

```markdown
OLD PostgreSQL 17.10
        │
        │ pg_dump
        ▼
~/zabbix/zabbix_backup_new.dump
        │
        │ test with pg_restore
        ▼
Valid backup
        │
        │ docker cp
        ▼
NEW PostgreSQL 17.10
        │
        │ pg_restore
        ▼
zabbix database
        │
        ▼
Zabbix Server 7.4.13
        │
        ▼
Old devices/configuration/history
```

### change docker desktop bridge ip address

If you want to change the default bridge

Edit Docker daemon configuration.

On Docker Desktop, this is usually done through Docker Engine settings.

Add/change:

```json
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "bip": "172.18.0.1/16"
```

For example:

```json
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "bip": "172.28.0.1/16"
```

Then Apply & Restart Docker.

After Docker restarts:

```bash
docker network inspect bridge
```

You should see:

```markdown
172.28.0.0/16
```

However, for your Zabbix problem

I suggest:

```markdown
Docker bridge:
172.28.0.0/16

Zabbix:
172.26.0.0/16
```

### zabbix or other ip address

if zabbix or other duplicate with internal router vlan network.

go to the compose folder and file

Change your Compose network to:

```yml
networks:
  default:
    name: zabbix_default
    driver: bridge
    ipam:
      config:
        - subnet: 172.26.0.0/16
```

Then:

```bash
docker compose down
```

Remove the old Zabbix network:

```bash
docker network rm zabbix_default
```

Then recreate:

```bash
docker compose up -d
```

Check zabbix network:

```bash
docker inspect zabbix-server --format '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}: IP={{$v.IPAddress}} Gateway={{$v.Gateway}}{{end}}'
```

or check all networks in docker:

```bash
docker network inspect $(docker network ls -q) --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}} {{end}}'
```

You should get something like:

```markdown
zabbix_default: IP=172.26.0.3 Gateway=172.26.0.1
```
