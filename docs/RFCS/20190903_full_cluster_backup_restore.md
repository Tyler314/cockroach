- Feature Name: Full-Cluster Backup Restore
- Status: In Progress
- Start Date: 2019-09-03
- Authors: Tyler Roberts
- RFC PR: N/A
- Cockroach Issue: N/A

# Summary
To add additional cluster information and metadata about the cluster that is to
be backed up & restored.


# Motivation
CockroachDB's BACKUP & RESTORE is currently limited to backing up and restoring
the data and schemas of tables and databases. There are additional pieces of
information that are not captured in this process, and as a result, the BACKUP
and RESTORE process is a much more manual process than it could be.

We want to include information pertaining to the cluster being backed up and
restored, along with what a normal backup of tables and databases already does.
Thus, allowing a client to restore this additional metadata along with their
tables and databases, without needing to do any of it manually.

# Detailed Design
## Goals
The goal is to include these additional pieces of metadata with the backup and
restore process. The `BackupDescriptor` is the crux of this process, containing
all of the essential pieces of information required to perform a backup, and
consequently, a restore. The majority of the additional data and metadata live
within the internal `system` and `crdb_internal` tables. The metadata can be
gathered through SQL queries already supported by CRDB. Listed below is the
metadata of interest, and the queries used to obtain it:

- SQL users: `SELECT * FROM system.users`
- SQL roles, User role memberships: `SELECT * FROM system.role_members`
- Grants to users on
	- Databases: `SHOW GRANTS ON DATABASE db`
	- Tables: `SHOW GRANTS ON TABLE db.t`
- Table zone configuration information: `SELECT * FROM system.zones`
- Cluster settings: `SELECT * FROM crdb_internal.cluster_settings`
- Cluster job history: `SELECT * FROM crdb_internal.jobs`

Gathering log, audit, and certificate information is also useful and relevant
for a cluster. To gather this information, we can use the `flags.Lookup`
function with the appropriate argument to get the full path of the directory
where these files live:
- Cluster log information: `flag.Lookup("log-dir").Value`
- Cluster audit information: `flag.Lookup("sql-audit-dir").Value`
- Certificate information: `flag.Lookup("certs-dir").Value`

We also wish to include some environment variables that are relevant to the
CRDB BACKUP. All of the relevant environment variables are prepended with
`COCKROACH`, and can be obtained in a straightforward way using `os.Environ()`.

## Proposal
### Backup
When a backup is initiated, a set of tables and/or databases is the data that
is being backed up. One of the first things that is done, is the parsing of the
query statement to determine which tables and/or databases they are.

To perform the SQL queries, the `sql.InternalExecutor` type will be used. To
keep track of the metadata associated with these tables and databases, a
`map[metaData][]tree.Datums` will be used to store the rows (as `[]tree.Datums`)
returned from the `sql.InternalExecutor`. It will be included within the
`BackupDescriptor` created in the function `backupPlanHook`.
The `metaData` type is defined as follows:

```go
type metaData int

const (
	_ metaData = iota
	users
	role_members
	table_grants
	db_grants
	zones
	cluster_settings
	jobs
)
```
This will allow an organized way to lookup and store the internal tables of
interest.

For the relevant environment variables, a `map[string]string` can be used where
the environment variable serves as the key, and the value of the environment
variable serves as the value of the map.

For storing log, audit, and certificate files, a `map[string]string` will be
used. Where the key serves as the relative path to the log files (relative to
specified log directory), and the value will be the contents of the log file.

For both environment variables and log files, it may make more sense to have
these be optionally backed up. As the client may not want to overwrite these,
especially the environment variables, into the environment they are restoring
to. Thus, it may be a good idea to consider adding an additional flag to BACKUP
that allows the client to specify if they want to backup these or not.

### Restore
Restoring this metadata is not as simple as creating the tables to be restored
into, and adding the sst files to the monolithic key value store. These tables
should already exist in the new environment being restored into, and precaution
needs to be taken when adding to these preexisting tables.

Each of the rows of the tables, of the metadata to be restored have a unique
identifier that we can use to check against the preexisting tables. While
performing this check, we can either make the decision to update a subset of the
columns to match the metadata being restored from backup, replace the row
entirely, or simply ignore it. The identifiers that can be used are listed as
follows for the metadata tables:

- SQL users: `username`
- SQL roles, User role memberships: `role`
- Grants to users on
	- Databases: `database_name`
	- Tables: `table_name`
- Table zone configuration information: `id`
- Cluster settings: `variable`

Cluster job history should be fine to be augmented to the `crdb_internal.jobs`
table.

When restoring the log, audit, and certificate information, as well as the
environment variables, it's not obvious whether or not these should be
overwritten or not in the case of duplicates. For the files, a simply solution
would be to simply append an identifier of some sort (such as `_restored`) to
the end of the file names. For the environment variables, it may be safe to
assume that if the same ones exist in the restored environment, that they should
be left alone and not overwritten.


