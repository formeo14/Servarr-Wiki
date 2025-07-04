---
title: Readarr Configuring PostgreSQL Database (Retired)
description: Configuring Readarr with a Postgres Database
published: true
date: 2025-05-29T21:30:46.018Z
tags: 
editor: markdown
dateCreated: 2022-07-25T22:49:56.668Z
---

# Announcement: Retirement of Readarr

We would like to announce that the [Readarr project](https://github.com/Readarr/Readarr) has been retired. This difficult decision was made due to a combination of factors: the project's metadata has become unusable, we no longer have the time to remake or repair it, and the community effort to transition to using Open Library as the source has stalled without much progress.

Third-party metadata mirrors exist, but as we're not involved with them at all, we cannot provide support for them. Use of them is entirely at your own risk. The most popular mirror appears to be [rreading-glasses](https://github.com/blampe/rreading-glasses).

Without anyone to take over Readarr development, we expect it to wither away, so we still encourage you to seek alternatives to Readarr.

## Key Points

- Effective Immediately: The retirement takes effect immediately. Please stay tuned for any possible further communications.
- Support Window: We will provide support during a brief transition period to help with troubleshooting non metadata related issues.
- Alternative Solutions: Users are encouraged to explore and adopt any other possible solutions as alternatives to Readarr.
- Opportunities for Revival: We are open to someone taking over and revitalizing the project. If you are interested, please get in touch.
- Gratitude: We extend our deepest gratitude to all the contributors and community members who supported Readarr over the years.

Thank you for being part of the Readarr journey. For any inquiries or assistance during this transition, please contact our team.

Sincerely,
The Servarr Team

# Readarr and Postgres

This document will go over the key points of migrating and setting up Postgres support in Readarr.

> Readarr v0.1.1.1408 or newer required
{.is-info}

This guide was been created by the amazing [Roxedus](https://github.com/Roxedus).

> Postgres databases are NOT backed up by Readarr, any backups must be implemented and maintained by the user
{.is-danger}

> Note that while the community migration guide is only written for **Postgres 14**. Users have **reported no issues with Postgres 15-17 inclusive**. Please note that the migration details below may not work with Postgres 15+.  **If one wishes to use a newer Postgres version than 14 they should start the application's database from scratch OR upgrade after the unsupported community migration is executed**.
{.is-info}

## Setting up Postgres

 First, we need a Postgres instance. This guide is written for usage of the `postgres:14` Docker image.

 > Do not even think about using the `latest` tag! {.is-danger}

```bash
docker create --name=postgres14 \
    -e POSTGRES_PASSWORD=qstick \
    -e POSTGRES_USER=qstick \
    -e POSTGRES_DB=readarr-main \
    -p 5432:5432/tcp \
    -v /path/to/appdata/postgres14:/var/lib/postgresql/data \
    postgres:14
```

## Creation of database

Readarr needs three databases, the default names of these are:

- `readarr-main`   This is used to store all configuration and history
- `readarr-log`    This is used to store events that produce a logentry
- `readarr-cache`    This is used to store GoodReads cache

> Readarr will not create the databases for you. Make sure you create them ahead of time{.is-warning}

Create the databases mentioned above using your favorite method - for example [pgAdmin](https://www.pgadmin.org/) or [Adminer](https://www.adminer.org/).

You can give the databases any name you want but make sure `config.xml` file has the correct names. For further information see [schema creation](/readarr/postgres-setup#schema-creation).

### Schema creation

 We need to tell Readarr to use Postgres. The `config.xml` should already be populated with the entries we need:

```xml
<PostgresUser>qstick</PostgresUser>
<PostgresPassword>qstick</PostgresPassword>
<PostgresPort>5432</PostgresPort>
<PostgresHost>postgres14</PostgresHost>
```

If you want to specify a database name then should also include the following configuration:

```xml
<PostgresMainDb>MainDbName</PostgresMainDb>
<PostgresLogDb>LogDbName</PostgresLogDb>
<PostgresCacheDb>CacheDbName</PostgresCacheDb>
```

Only **after creating** all three databases you can start the Readarr migration from SQLite to Postgres.

## Migrating data

> If you do not want to migrate a existing SQLite database to Postgres then you are already finished with this guide! {.is-info}

> Migrating an existing sqlite3 database is unsupported, and this script may not work without modifications which we cannot assist you with. We support only new installs using postgres. {.is-warning}

To migrate data we can use [PGLoader](https://github.com/dimitri/pgloader). It does, however, have some gotchas:

- By default transactions are case-insensitive, we use `--with "quote identifiers"` to make them sensitive.
- The version packaged in Debian and Ubuntu's apt repo are too old for newer versions of Postgres (Roxedus has not tested packages in other distros).
  Roxedus [built a binary](https://github.com/Roxedus/Pgloader-bin) to enable this support (no code modification was needed, simply had to be built with updated dependencies).

> Do not drop any tables in the Postgres instance {.is-danger}

Before starting a migration please ensure that you have run Readarr against the created Postgres databases **at least once** successfully. Begin the migration by doing the following:

1. Stop Readarr
1. Open your preferred database management tool and connect to the Postgres database instance
1. Run the following commands:

```SQL
DELETE FROM "QualityProfiles";
DELETE FROM "QualityDefinitions";
DELETE FROM "DelayProfiles";
DELETE FROM "MetadataProfiles";
```

1. Start the migration by using either of these options:

    - ```bash
      pgloader --with "quote identifiers" --with "data only" readarr.db 'postgresql://qstick:qstick@localhost/readarr-main'
      ```

    - ```bash
      docker run --rm -v /absolute/path/to/readarr.db:/readarr.db:ro --network=host ghcr.io/roxedus/pgloader --with "quote identifiers" --with "data only" /readarr.db "postgresql://qstick:qstick@localhost/readarr-main"
      ```

    > If you experience an error using pgloader it could be due to your DB being too large, to resolve this try adding `--with "prefetch rows = 100" --with "batch size = 1MB"` to the above command {.is-warning}

    > With these handled, it is pretty straightforward after telling it to not mess with the scheme using `--with "data only"`
    {.is-info}

2. For those having the issues POST-MIGRATION from SQLite run the following:

    ```postgres
    select setval('public."AuthorMetadata_Id_seq"', (SELECT MAX("Id")+1 FROM "AuthorMetadata"));
    select setval('public."Authors_Id_seq"', (SELECT MAX("Id")+1 FROM "Authors"));
    select setval('public."Blacklist_Id_seq"', (SELECT MAX("Id")+1 FROM "Blocklist"));
    select setval('public."BookFiles_Id_seq"', (SELECT MAX("Id")+1 FROM "BookFiles"));
    select setval('public."Books_Id_seq"', (SELECT MAX("Id")+1 FROM "Books"));
    select setval('public."Commands_Id_seq"', (SELECT MAX("Id")+1 FROM "Commands"));
    select setval('public."Config_Id_seq"', (SELECT MAX("Id")+1 FROM "Config"));
    select setval('public."CustomFilters_Id_seq"', (SELECT MAX("Id")+1 FROM "CustomFilters"));
    select setval('public."CustomFormats_Id_seq"', (SELECT MAX("Id")+1 FROM "CustomFormats"));
    select setval('public."DelayProfiles_Id_seq"', (SELECT MAX("Id")+1 FROM "DelayProfiles"));
    select setval('public."DownloadClients_Id_seq"', (SELECT MAX("Id")+1 FROM "DownloadClients"));
    select setval('public."DownloadClientStatus_Id_seq"', (SELECT MAX("Id")+1 FROM "DownloadClientStatus"));
    select setval('public."DownloadHistory_Id_seq"', (SELECT MAX("Id")+1 FROM "DownloadHistory"));
    select setval('public."Editions_Id_seq"', (SELECT MAX("Id")+1 FROM "Editions"));
    select setval('public."ExtraFiles_Id_seq"', (SELECT MAX("Id")+1 FROM "ExtraFiles"));
    select setval('public."History_Id_seq"', (SELECT MAX("Id")+1 FROM "History"));
    select setval('public."ImportListExclusions_Id_seq"', (SELECT MAX("Id")+1 FROM "ImportListExclusions"));
    select setval('public."ImportLists_Id_seq"', (SELECT MAX("Id")+1 FROM "ImportLists"));
    select setval('public."ImportListStatus_Id_seq"', (SELECT MAX("Id")+1 FROM "ImportListStatus"));
    select setval('public."Indexers_Id_seq"', (SELECT MAX("Id")+1 FROM "Indexers"));
    select setval('public."IndexerStatus_Id_seq"', (SELECT MAX("Id")+1 FROM "IndexerStatus"));
    select setval('public."Metadata_Id_seq"', (SELECT MAX("Id")+1 FROM "Metadata"));
    select setval('public."MetadataFiles_Id_seq"', (SELECT MAX("Id")+1 FROM "MetadataFiles"));
    select setval('public."MetadataProfiles_Id_seq"', (SELECT MAX("Id")+1 FROM "MetadataProfiles"));
    select setval('public."NamingConfig_Id_seq"', (SELECT MAX("Id")+1 FROM "NamingConfig"));
    select setval('public."Notifications_Id_seq"', (SELECT MAX("Id")+1 FROM "Notifications"));
    select setval('public."PendingReleases_Id_seq"', (SELECT MAX("Id")+1 FROM "PendingReleases"));
    select setval('public."QualityDefinitions_Id_seq"', (SELECT MAX("Id")+1 FROM "QualityDefinitions"));
    select setval('public."QualityProfiles_Id_seq"', (SELECT MAX("Id")+1 FROM "QualityProfiles"));
    select setval('public."ReleaseProfiles_Id_seq"', (SELECT MAX("Id")+1 FROM "ReleaseProfiles"));
    select setval('public."RemotePathMappings_Id_seq"', (SELECT MAX("Id")+1 FROM "RemotePathMappings"));
    select setval('public."RootFolders_Id_seq"', (SELECT MAX("Id")+1 FROM "RootFolders"));
    select setval('public."ScheduledTasks_Id_seq"', (SELECT MAX("Id")+1 FROM "ScheduledTasks"));
    select setval('public."Series_Id_seq"', (SELECT MAX("Id")+1 FROM "Series"));
    select setval('public."SeriesBookLink_Id_seq"', (SELECT MAX("Id")+1 FROM "SeriesBookLink"));
    select setval('public."Tags_Id_seq"', (SELECT MAX("Id")+1 FROM "Tags"));
    select setval('public."Users_Id_seq"', (SELECT MAX("Id")+1 FROM "Users"));
    ```

3. Start Readarr
