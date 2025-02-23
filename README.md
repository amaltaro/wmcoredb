This repository contains the code for creating the WMAgent database.

# WMBS Database Schema

## Overview
WMBS (Workload Management Bookkeeping Service) provides the database schema for managing workloads and jobs.

## Database Schema Initialization

The WMBS database schema can be initialized using either MariaDB or Oracle backends. The schema files are organized as follows:

```
src/
├── python/
│   └── db/
│       ├── create_wmbs.py
│       └── execute_wmbs_sql.py
└── sql/
    └── init/
        ├── oracle/
        │   ├── create_wmbs_tables.sql
        │   ├── create_wmbs_indexes.sql
        │   └── initial_wmbs_data.sql
        └── mariadb/
            ├── create_wmbs_tables.sql
            ├── create_wmbs_indexes.sql
            └── initial_wmbs_data.sql
```

## Schema changes

For Oracle, replacing `INTEGER` by `NUMBER(11)`. INTEGER is equivalent to `NUMBER(38)`, which can store up to 38 digits. Hence, `NUMBER(11)` can store up to 11 digits.

While `VARCHAR` is an ANSI standard data type, it has not been fully implemented in Oracle and its use is discouraged. The `VARCHAR2` is Oracle's proprietary data type and it is the recommended type for variable-length character strings.

Oracle 12c introduced the `IDENTITY` column feature as a simpler alternative to the sequence/trigger combination for auto-incrementing columns. The syntax:

```sql
id NUMBER GENERATED BY DEFAULT AS IDENTITY
```

This is equivalent to MariaDB's:
```sql
id INTEGER AUTO_INCREMENT
```

Key benefits of using IDENTITY:
- Simpler syntax and maintenance
- No need for separate sequence objects
- No need for trigger definitions
- Automatically handles concurrent inserts
- Compatible with Oracle 12c and later versions

# Logs

Some relevant logs from the WMAgent 2.3.9.2 installation:
```
Start: Performing init_agent
init_agent: triggered.
Initializing WMAgent...
init_wmagent: MYSQL database: wmagent has been created
DEBUG:root:Log file ready
DEBUG:root:Using SQLAlchemy v.1.4.54
INFO:root:Instantiating base WM DBInterface
DEBUG:root:Tables for WMCore.WMBS created
DEBUG:root:Tables for WMCore.Agent.Database created
DEBUG:root:Tables for WMComponent.DBS3Buffer created
DEBUG:root:Tables for WMCore.BossAir created
DEBUG:root:Tables for WMCore.ResourceControl created
checking default database connection
default database connection tested
...
_sql_write_agentid: Preserving the current WMA_BUILD_ID and HostName at database: wmagent.
_sql_write_agentid: Creating wma_init table at database: wmagent
_sql_write_agentid: Inserting current Agent's build id and hostname at database: wmagent
_sql_dumpSchema: Dumping the current SQL schema of database: wmagent to /data/srv/wmagent/2.3.9/config/.wmaSchemaFile.sql
Done: Performing init_agent
```
# WMAgent DB Initialization

It starts in the CMSKubernetes [init.sh](https://github.com/dmwm/CMSKubernetes/blob/master/docker/pypi/wmagent/init.sh#L465) script, which executes `init_agent()` method from the CMSKubernetes [manage](https://github.com/dmwm/CMSKubernetes/blob/master/docker/pypi/wmagent/bin/manage#L112) script.

The database optios are enriched dependent on the database flavor, such as:
```bash
    case $AGENT_FLAVOR in
        'mysql')
            _exec_mysql "create database if not exists $wmaDBName"
            local database_options="--mysql_url=mysql://$MDB_USER:$MDB_PASS@$MDB_HOST/$wmaDBName "
        'oracle')
            local database_options="--coredb_url=oracle://$ORACLE_USER:$ORACLE_PASS@$ORACLE_TNS "
```

It then executes WMCore code, calling a script called [wmagent-mod-config](https://github.com/dmwm/WMCore/blob/master/bin/wmagent-mod-config).

with command line arguments like:
```bash
    wmagent-mod-config $database_options \
                       --input=$WMA_CONFIG_DIR/config-template.py \
                       --output=$WMA_CONFIG_DIR/config.py \
```

which internally parses the command line arguments into `parameters` and modifies the standard [WMAgentConfig.py](https://github.com/dmwm/WMCore/blob/master/etc/WMAgentConfig.py), saving it out as the new WMAgent configuration file, with something like:
```
    cfg = modifyConfiguration(cfg, **parameters)
    saveConfiguration(cfg, outputFile)
```

With the WMAgent configuration file properly updated, named `config.py`, now the `manage` script calls [wmcore-db-init](https://github.com/dmwm/WMCore/blob/master/bin/wmcore-db-init), with arguments like:

```bash
wmcore-db-init --config $WMA_CONFIG_DIR/config.py --create --modules=WMCore.WMBS,WMCore.Agent.Database,WMComponent.DBS3Buffer,WMCore.BossAir,WMCore.ResourceControl;
```

This `wmcore-db-init` script itself calls the [WMInit.py](https://github.com/dmwm/WMCore/blob/master/src/python/WMCore/WMInit.py) script, executing basically the next four commands:
```python
wmInit = WMInit()
wmInit.setLogging('wmcoreD', 'wmcoreD', logExists = False, logLevel = logging.DEBUG)
wmInit.setDatabaseConnection(dbConfig=config.CoreDatabase.connectUrl, dialect=dialect, socketLoc = socket)
wmInit.setSchema(modules, params = params)
```

In summary, the WMAgent database schema is an aggregation of the schema defined under each of the following WMAgent python directories:
```
WMCore.WMBS             --> originally under src/python/db/wmbs
WMCore.Agent.Database   --> originally under src/python/db/agent
WMCore.BossAir          --> originally under src/python/db/bossair
WMCore.ResourceControl  --> originally under src/python/db/resourcecontrol
WMComponent.DBS3Buffer  --> originally under src/python/db/dbs3buffer
```

The `wmcore-db-init` script itself calls the [WMInit.py](https://github.com/dmwm/WMCore/blob/master/src/python/WMCore/WMInit.py) script, executing basically the next four commands:
```python
wmInit = WMInit()
wmInit.setLogging('wmcoreD', 'wmcoreD', logExists = False, logLevel = logging.DEBUG)
```