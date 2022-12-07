---
layout: post
title: "Restoring RDS Postgres to Docker container"
date: 2022-12-06
categories: sql postgres rds docker
github_comments_issueid: 8
---

I wanted to make an production RDS database available to developers to use in their pipelines. The database should be up to date and, if necessary, anonymized for GDPR compliance.

Useful links:

- [psql docs](https://www.postgresql.org/docs/15/app-psql.html)
- [pg_dump docs](https://www.postgresql.org/docs/15/app-pgdump.html)

First step: get a dump of the data and restore in a container at runtime.

## Getting the dump

Assuming connectivity to the RDS Postgres database instance.

- Install postgres & aws cli on the instance where you'll be running pg_dump and uploading to S3. If you're using an Amazon Linux2 instance then you can install from amazon linux extras, otherwise download and install

- Use pg_dump to make backup locally

```(bash)
pg_dump \
  --host=my-database-instance.c4cxxxxxxpxoru.eu-west-1.rds.amazonaws.com \
  --port=5432 \
  --username=root \
  --password \
  --format plain \
  --file dump.sql my-database
```

--password flag will force a prompt for the user password.

- Upload the file to S3 and then download to local directory

```(bash)
aws s3 cp dump.sql s3://my-upload-bucket/dump.sql
```

There is the option to create a binary dump using `--format custom` which is supposed to be quicker to restore for large databases. Binary dumps are restored using `pg_restore` while plain text dumps are restored with `psql`.

## Knowing which users to create before restoring

Make sure you know which users have permissions in the database as these will need to be created before restoring the database. At the very least you'll need rdsadmin. The following command will tell you which users are configured on the instance:

```(bash)
psql \
  --host=my-database-instance.c4cxxxxxxpxoru.eu-west-1.rds.amazonaws.com \
  --port=5432 \
  --username=root \
  --password my-database \
  --command '\du'
```

Whether root works for you will depend on how you configured the RDS database. User `postgres` might work if you don't have a `root` user.

```(bash)
 Role name       | Attributes                                                 | Member of
-----------------+-----------------------------------------------------------------------
 rds_ad          | Cannot login                                               | {}
 rds_iam         | Cannot login                                               | {}
 rds_password    | Cannot login                                               | {}
 rds_replication | Cannot login                                               | {}
 rds_superuser   | Cannot login                                               | {pg_monitor,pg_signal_backend,rds_replication,rds_password}
 rdsadmin        | Superuser, Create role, Create DB, Replication, Bypass RLS+| {}
                 | Password valid until infinity                              |
 rdsrepladmin    | No inheritance, Cannot login, Replication                  | {}
 root            | Create role, Create DB                                    +| {rds_superuser}
                 | Password valid until infinity                              |

```

In my case I needed to create root and rdsadmin. Although I could just have removed the line `REVOKE ALL ON SCHEMA public FROM rdsadmin;` and not bothered to create the rdsadmin user.

## Creating the Container Image to restore at runtime

For this we need the dump.sql file and 3 files:

### restore.sh

```(bash)
#!/bin/bash

file="/tmp/dump.sql"
script="/tmp/script.sql"

dbname=my-database

echo "Restoring DB using $file"
psql -U postgres  < "$script"
psql -U postgres -d "$dbname" < "$file"
```

### script.sql

```(sql)
CREATE ROLE root LOGIN SUPERUSER PASSWORD 'rootpassword';
CREATE ROLE rdsamin;

CREATE DATABASE my-database ENCODING='UTF-8' OWNER='root';
```

### Dockerfile

```(dockerfile)
FROM postgres:latest
ENV POSTGRES_PASSWORD="rootpassword"

COPY script.sql /tmp/script.sql
COPY dump.sql /tmp/dump.sql
COPY restore.sh /docker-entrypoint-initdb.d/restore.sh
```

By copying `restore.sh` to the startup folder, the script will run when the container starts.

## Build & Launch the container

```(bash)
docker build -t my-db .

docker run -d my-db -p 5432:5432
```

The container is running in the background.

It's also possible to drop the ENV from the Dockerfile and instead pass the postgres password variable in the docker run command `docker run  -e POSTGRES_PASSWORD=rootpassword -d my-db -p 5432:5432`. Either way works and at this point we're just proving the container runs and the database can be restored.

## Check the dump has been restored

```(bash)
docker exec -it container_name /bin/bash

psql -U root -d my-database -c 'select * from "MyTable"'
```

Couple of things to note. If the tables are part of a schema that isn't `public`, you need to specify the schema in the select query `'select * from my-schema."MyTable"'`.

If the table name is not all lowercase then it needs to be quoted.

<script src="https://utteranc.es/client.js"
        repo="indigo-tangerine/indigo-tangerine.github.io"
        issue-term="title"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
