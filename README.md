# Airbnb data transformation with dbt, Snowflake, and AWS

This repository contains a dbt project that transforms Airbnb booking, host, and
listing data in Snowflake. The dbt project expects the raw data to already be
available in Snowflake; it does not currently upload files to S3 or load data
from AWS.

## Data flow

```text
Optional upstream ingestion: AWS S3 -> Snowflake AIRBNB.STAGING
                                      |
                                      v
                         dbt bronze -> silver -> gold
```

The configured Snowflake sources are `AIRBNB.STAGING.BOOKINGS`,
`AIRBNB.STAGING.HOSTS`, and `AIRBNB.STAGING.LISTINGS`. dbt creates or updates
models in `AIRBNB.BRONZE`, `AIRBNB.SILVER`, and `AIRBNB.GOLD`. The Gold layer
also contains snapshots.

## Repository layout

| Path | Purpose |
| --- | --- |
| `airbnb_aws_dbt_proj/models/sources/sources.yml` | Declares the three Snowflake staging tables |
| `airbnb_aws_dbt_proj/models/bronze/` | Loads and incrementally filters source data |
| `airbnb_aws_dbt_proj/models/silver/` | Applies cleaned and transformed model logic |
| `airbnb_aws_dbt_proj/models/gold/` | Builds the Gold OBT and fact models |
| `airbnb_aws_dbt_proj/snapshots/` | Defines booking, host, and listing snapshots |
| `airbnb_aws_dbt_proj/macros/` | Shared SQL transformations |
| `pyproject.toml`, `uv.lock` | Python and dbt dependency configuration |

## Prerequisites

- Python 3.13 or newer.
- [uv](https://docs.astral.sh/uv/) installed for installing the locked Python dependencies.
- A Snowflake account, a warehouse, and a role with the permissions described
  under [Snowflake setup](#snowflake-setup).
- The three populated staging tables listed under [Expected source data](#expected-source-data).

## Local setup (Windows PowerShell)

Run these commands from the repository root:

```powershell
uv sync
New-Item -ItemType Directory -Force "$HOME\.dbt" | Out-Null
notepad "$HOME\.dbt\profiles.yml"
```

Add a profile named `airbnb_aws_dbt_proj` to
`%USERPROFILE%\.dbt\profiles.yml`. For password authentication, a minimal profile
is:

```yaml
airbnb_aws_dbt_proj:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE', 'COMPUTE_WH') }}"
      database: AIRBNB
      schema: DBT_DEV
      threads: 4
```

Set the environment variables in the same PowerShell session before running
dbt. Replace the placeholders with values provided by your Snowflake
administrator:

```powershell
$env:SNOWFLAKE_ACCOUNT = "<your-snowflake-account-identifier>"
$env:SNOWFLAKE_USER = "<your-snowflake-user>"
$env:SNOWFLAKE_PASSWORD = "<your-snowflake-password>"
$env:SNOWFLAKE_ROLE = "<your-dbt-role>"
$env:SNOWFLAKE_WAREHOUSE = "COMPUTE_WH"
```

Use your organization's approved authentication method if it requires SSO or
key-pair authentication instead of a password. Do not put real credentials in
this repository or commit them to Git.

Check the connection and dbt project configuration:

```powershell
uv run dbt debug --project-dir .\airbnb_aws_dbt_proj --profiles-dir "$HOME\.dbt"
```

Build the project and its selected dependencies:

```powershell
uv run dbt build --project-dir .\airbnb_aws_dbt_proj --profiles-dir "$HOME\.dbt"
```

To build only the `fact` model and its upstream dependencies:

```powershell
uv run dbt build --select +fact --project-dir .\airbnb_aws_dbt_proj --profiles-dir "$HOME\.dbt"
```

`dbt build` runs selected models, snapshots, and tests in dependency order.
`dbt run` runs models only; it does not run snapshots. If you choose to use
`dbt run`, run `dbt snapshot` separately when the snapshot tables need to be
created or updated.

## Expected source data

Create and populate these Snowflake tables before building the dbt models:

| Table | Columns used by this project |
| --- | --- |
| `AIRBNB.STAGING.BOOKINGS` | `BOOKING_ID`, `LISTING_ID`, `BOOKING_DATE`, `NIGHTS_BOOKED`, `BOOKING_AMOUNT`, `SERVICE_FEE`, `CLEANING_FEE`, `BOOKING_STATUS`, `CREATED_AT` |
| `AIRBNB.STAGING.LISTINGS` | `LISTING_ID`, `HOST_ID`, `PROPERTY_TYPE`, `ROOM_TYPE`, `CITY`, `COUNTRY`, `ACCOMMODATES`, `BEDROOMS`, `BATHROOMS`, `PRICE_PER_NIGHT`, `CREATED_AT` |
| `AIRBNB.STAGING.HOSTS` | `HOST_ID`, `HOST_NAME`, `HOST_SINCE`, `IS_SUPERHOST`, `RESPONSE_RATE`, `CREATED_AT` |

Use compatible Snowflake types for each field. In particular, IDs should be
consistent across related tables, and `CREATED_AT` should be a timestamp
because the incremental models and snapshots use it to identify changes.

## Snowflake setup

Ask a Snowflake administrator to prepare the following for the development
environment:

1. An `AIRBNB` database and a compute warehouse (the profile example uses
   `COMPUTE_WH`).
2. The `STAGING` schema and the three source tables above, populated with data.
3. The `BRONZE`, `SILVER`, and `GOLD` schemas for dbt output. The project
   configures these as custom schemas, so models are written to these schema
   names rather than to a schema such as `DBT_DEV_BRONZE`.
4. A dedicated dbt role and user. Grant the role permission to use the
   warehouse and database, read the staging tables, create the required tables
   in the output schemas, and read/update existing incremental and snapshot
   tables. Follow your organization's least-privilege policy; avoid using
   `ACCOUNTADMIN` for routine dbt runs.
5. A Snowflake account identifier and the approved authentication details for
   the local dbt profile.

The project does not create the Snowflake account, warehouse, database, role,
or raw staging tables. These are one-time environment prerequisites.

## AWS setup (only if S3 is your upstream data source)

There is no AWS SDK, S3 configuration, IAM role, or Snowflake storage
integration defined in this repository. The dbt models connect to Snowflake and
read the staging tables; they do not connect to AWS directly. Therefore, AWS
credentials are not needed to run dbt as currently configured.

If your wider data pipeline uses Airbnb files in S3, configure that upstream
ingestion separately:

1. Create an S3 bucket and a clearly scoped prefix for the source files. Keep
   the bucket private and use your organization's encryption and retention
   requirements.
2. Create an IAM role for the ingestion path with access limited to the
   required bucket and prefix. Do not use AWS root credentials or commit access
   keys.
3. Configure the S3-to-Snowflake connection using a Snowflake storage
   integration and an AWS trust relationship, following your organization's
   approval process. Do not embed AWS keys in dbt SQL or profiles.
4. Load and validate the files into the three `AIRBNB.STAGING` tables before
   running dbt. Loading can be done by an existing ingestion service, Snowpipe,
   or another separately configured process.

Until that ingestion exists, arrange for the staging tables to be populated by
another approved method. Creating an AWS bucket alone will not make the dbt
models able to read its files.

## Troubleshooting

- **Profile not found:** confirm the profile key is exactly
  `airbnb_aws_dbt_proj`, and that the command includes the correct
  `--profiles-dir`.
- **Snowflake object does not exist or is not authorized:** confirm the source
  tables exist in `AIRBNB.STAGING`, output schemas exist, and the active role
  has the required privileges.
- **Gold snapshot tables are missing:** run `dbt build` (or `dbt snapshot`)
  before using a command that runs only models. `dbt run` does not execute
  snapshots.
- **Missing columns or invalid types:** compare the staging table definitions
  with [Expected source data](#expected-source-data).

## Useful references

- [dbt documentation](https://docs.getdbt.com/)
- [dbt Snowflake setup](https://docs.getdbt.com/docs/core/connect-data-platform/snowflake-setup)
- [Snowflake documentation](https://docs.snowflake.com/)
- [AWS S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)