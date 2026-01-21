Ente Auth: Podman Compose Deployment
====================================

This repository provides a template for running **Ente Auth** using Podman Compose. This is a multi-container stack involving a Go server (Museum), a Web UI, and a Postgres database.  PLEASE BACKUP YOUR ENTIRE CURRENT ENTE AUTH DIRECTORY IF YOU ARE TRYING TO SWITCH OVER FROM A DOCKER!  Also keep the attributes of all the folders/files.  This way if things go bad you can just nuke the Podman Compose attempt and copy everything back over to the Docker.  I can't stress this enough.  The postgres database is fickle and I do not want you losing all of your MFA codes.

Required Modifications
----------------------

Before running `podman-compose up`, you **must** update the following placeholders in `compose.yaml` and `museum.yaml`:

*   **Passwords:** Replace all instances of `changeme_database_password`.
*   **IP Addresses:** Replace `YOUR_SERVER_IP` with your local server IP (e.g. 192.168.1.50).
*   **Secrets:** Generate unique Base64 strings for `encryption`, `hash`, and `jwt.secret`.
*   **SMTP:** Configure your email provider details for notifications.

1\. Permissions & User Namespaces
---------------------------------

Because this runs in **Rootless Podman**, Postgres internal UIDs are mapped to high ranges on the host. To ensure the database can start, reset ownership of your data folder to your current user:

`sudo chown -R $USER:$USER ./db-data`

2\. Handling Migration (Docker to Podman)
-----------------------------------------

If you are moving from a standard Docker install:

1.  Backup: `docker exec postgres pg_dump -U pguser ente_db > backup.sql`
2.  Stop Docker and rename your old data folder to `db-data-old`.
3.  Start this stack: `podman-compose up -d`
4.  Restore the SQL: `cat backup.sql | podman exec -i postgres psql -U pguser -d ente_db`

3\. Backup Script
-----------------

It is recommended to create a `backup_db.sh` script to run as a cron job:

`#!/bin/bash`
`podman exec postgres pg\_dump -U pguser ente\_db > ~/my-ente/backups/ente\_auth\_backup\_$(date +%F\_%H-%M).sql`
`find ~/my-ente/backups/ -type f -mtime +30 -delete`

This will back up your database every time it is run.

4\. Deployment Commands
-----------------------

\# Start the stack
`podman-compose up -d`

# Check status
`podman-compose ps`

# View logs
`podman-compose logs -f museum`

My bad! I definitely cut the cord too early there. Let’s get the full Section 5 (Update) and the Section 6 (Safety & Backups) back on the board so the GitHub guide is complete. Here is the finished HTML for the Ente Auth deployment, including the parts that were missing: HTML

5\. How to Update the Stack
---------------------------

Because Podman Compose doesn't automatically "watch" for new images like WUD, you need to pull and recreate the containers manually. Follow this ritual:

### Step 1: Pull New Images

This checks the registry and downloads any new layers for Museum, Web, or Postgres:

podman-compose pull

### Step 2: Recreate Containers

Running `up -d` again will force Podman to compare the running container with the new image and recreate it if they don't match:

podman-compose up -d

### Step 3: Post-Update Cleanup

Remove the old, "dangling" image layers that are no longer used to keep your storage clean:

podman image prune -f

6\. Safety & Database Backups
-----------------------------

Since this stack manages your 2FA and Auth data, a **Safety Backup** is mandatory. Do not rely solely on the `db-data` folder; you need a SQL dump.

### The "Safety Net" Copy

Before making any major configuration changes, manually copy your database files to a "frozen" directory:

cp -r ~/my-ente/db-data ~/my-ente/db-data-frozen-$(date +%F)

### Automated SQL Backup

Add this to your crontab (`crontab -e`) to backup the database every night at 3:00 AM:

00 03 \* \* \* podman exec postgres pg\_dump -U pguser ente\_db > ~/my-ente/backups/ente\_auth\_$(date +\\%F).sql

**Pro-Tip:** Store your `museum.yaml` and your SQL backups in different physical locations (or an encrypted cloud bucket). If you lose your `encryption` and `hash` keys from the yaml file, your database becomes unreadable!

One more thing for your GitHub: generate-secrets.sh Since you have those Base64 keys in museum.yaml that everyone has to change, you should include this tiny script in your repo. It makes it easy for the user to get the right format. generate-secrets.sh Bash #!/bin/bash echo "--- COPY THESE INTO YOUR museum.yaml ---" echo "" echo "Encryption Key (32-byte):" openssl rand -base64 32 echo "" echo "Hash Key (64-byte):" openssl rand -base64 64 echo "" echo "JWT Secret:" openssl rand -base64 32 Would you like me to move on to the next container, or are we ready to wrap up the Ente Auth section?
