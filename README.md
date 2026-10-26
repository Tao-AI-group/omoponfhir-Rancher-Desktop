# OMOP CDM 5.4 + OMOPonFHIR on Rancher Desktop (Docker)

## Introduction

This guide sets up, step by step and from an empty computer, a local OMOP database with a FHIR server on top of it.

- **OMOP CDM** (Observational Medical Outcomes Partnership Common Data Model) is a standard
  database layout for clinical data: persons, conditions, procedures, drugs, measurements and so on, all coded with
  the same standard vocabularies (SNOMED, ICD, RxNorm, LOINC, CPT4, ...). Data in this layout can be analysed with
  the same queries and tools at any site.
- **FHIR** (Fast Healthcare Interoperability Resources) is a standard web interface for clinical data.
  Applications read resources such as `Patient` or `Condition` over HTTP in a standard JSON format.
- **OMOPonFHIR** is a server that offers the data of an OMOP database through a FHIR
  interface, so FHIR applications can read OMOP data without knowing the database layout.

At the end of the guide you have, on your own machine:

1. a PostgreSQL database with the OMOP CDM 5.4 tables, the OMOP standard vocabularies from Athena and your clinical
   data, loaded from CSV files and mapped to standard concepts;
2. an OMOPonFHIR server that serves this data as FHIR R4, read-only and protected by a user name and password.

Everything runs in Docker containers on Rancher Desktop. The versions tested with this guide (CDM release, vocabulary,
server code) are given in the steps. Each step has a short purpose, the commands to run and a
check that shows whether the step worked.

**Platform:** this guide was written and tested on **macOS** (Apple silicon). The commands assume a Mac terminal.

The guide has two parts:

- **Part A. OMOP database:** a PostgreSQL database with the OMOP CDM 5.4 tables, the standard vocabularies and the
  clinical data. It can be used on its own by any OMOP tool.
- **Part B. OMOPonFHIR server:** a FHIR server on top of the Part A database. Part B needs a finished Part A.

The steps are numbered from 1 in each part.

---

# Part A. OMOP database

## Step 1. Install prerequisites

**Purpose:** install the tools used by all later steps: Rancher Desktop (runs the Docker containers) and `git`
(downloads the OHDSI and OMOPonFHIR code). No separate PostgreSQL client is needed: all SQL is run with the `psql`
inside the PostgreSQL container (`docker exec`).

### 1.1 Rancher Desktop

1. Download and install Rancher Desktop from https://rancherdesktop.io/ (tested with version 1.21.0, Apple silicon).
2. Open Rancher Desktop, then **Preferences**:
   - **Container Engine:** `dockerd (moby)`. This gives the `docker` command used in this guide.
   - **Kubernetes:** not needed for this guide (both containers run in plain Docker). It can stay off.
   - **Virtual Machine:** memory at least **4 GB**, 2 CPUs. With less memory, building the OMOPonFHIR image (Part B) can fail with exit code 137.
3. Close and reopen the terminal.

If the terminal says `docker: command not found`, set **Preferences > Application > Environment > PATH configuration**
to `Automatic` and reopen the terminal.

### 1.2 git

git comes with Apple's Command Line Tools. Run `git --version`; if macOS asks to install the Command Line Tools,
accept and run the command again.

### 1.3 Check

```bash
docker version --format 'client {{.Client.Version}}  server {{.Server.Version}}'
git --version
```

Expected: docker prints a client and a server version, and `git` prints its version.

**Tested with:** macOS 26.7.1 (Apple M3, 8 GB RAM), Rancher Desktop 1.21.0 (dockerd 28.3.3),
git 2.54.0 (Apple).

---

## Step 2. Start the PostgreSQL container

**Purpose:** run PostgreSQL in a Docker container on its own Docker network (the OMOPonFHIR server joins the same
network in Part B). Three folders on the Mac are made readable inside the container, so that the `psql` inside it
can read the SQL scripts, vocabulary files and clinical files. Nothing is copied.

### 2.1 Choose the folders

Set three variables to folders of your choice (they can be anywhere, also on an external drive):

```bash
export VOCAB_DIR=/path/to/vocab     # OMOP vocabulary files from Athena (Step 6), about 5 GB
export DATA_DIR=/path/to/data       # your clinical CSV files (Step 7)
export REPOS_DIR=/path/to/repos     # the OHDSI and OMOPonFHIR code (Part A Steps 4 and 8, Part B), small
mkdir -p "$VOCAB_DIR" "$DATA_DIR" "$REPOS_DIR"
```

These variables are used in all later steps. In a new terminal, run the `export` lines again.

The database itself is stored inside Rancher Desktop's virtual disk, not in these folders. With a full vocabulary
and a cohort of about 1,200 patients it takes about 25 GB, so make sure the Mac has that much free space.

### 2.2 Set the database password

Create a file `.env` (replace `<your_password>` with a password of your choice):

```bash
echo 'POSTGRES_PASSWORD=<your_password>' > "$REPOS_DIR/.env"
chmod 600 "$REPOS_DIR/.env"
```

Do not put quotes around the password and do not commit this file to git.

### 2.3 Create the Docker network

```bash
docker network create omop-network
```

If it says `network with name omop-network already exists`, the network is already there; continue.

### 2.4 Start PostgreSQL

```bash
docker run -d \
  --name omop-postgres \
  --network omop-network \
  --env-file "$REPOS_DIR/.env" \
  -p 5432:5432 \
  -v omop-pgdata:/var/lib/postgresql/data \
  -v "$VOCAB_DIR":/vocab:ro \
  -v "$DATA_DIR":/data:ro \
  -v "$REPOS_DIR":/repos:ro \
  --shm-size=1g \
  --restart unless-stopped \
  postgres:17
```

- `-v omop-pgdata:...` keeps the database in a Docker volume, so it survives if the container is removed.
- `-v "$VOCAB_DIR":/vocab:ro` (and the same for `/data` and `/repos`) makes each folder readable inside the
  container. `:ro` means read-only: the container cannot change your files.
- `--shm-size=1g` gives PostgreSQL enough shared memory for building indices on the large vocabulary tables.
- `-p 5432:5432` lets tools on the Mac connect to the database. If port 5432 is already in use, use for example
  `-p 5433:5432`.

### 2.5 Check

```bash
docker ps --filter name=omop-postgres
docker exec omop-postgres psql -U postgres -c "SELECT version();"
docker exec omop-postgres ls -d /vocab /data /repos
```

Expected: the container is `Up`, PostgreSQL prints its version (17.x), and `ls` lists `/vocab`, `/data` and `/repos`.

**Tested with:** `postgres:17` image, PostgreSQL 17.11 at the time of testing.

---

## Step 3. Create the OMOP database

**Purpose:** create an empty database `omop_v5` that will hold the OMOP CDM tables (Step 4).

```bash
docker exec omop-postgres psql -U postgres -c "CREATE DATABASE omop_v5;"
```

### 3.1 Check

```bash
docker exec omop-postgres psql -U postgres -c "\l omop_v5"
```

Expected: one row for `omop_v5`, owner `postgres`, encoding `UTF8`.

---

## Step 4. Create the OMOP CDM 5.4 tables

**Purpose:** create the 39 empty OMOP CDM 5.4 tables in `omop_v5`, using the official PostgreSQL scripts from the
OHDSI CommonDataModel repository (release v5.4.2).

### 4.1 Download the CommonDataModel scripts

```bash
git clone --branch v5.4.2 --depth 1 https://github.com/OHDSI/CommonDataModel.git "$REPOS_DIR/CommonDataModel"
```

git prints a note about a 'detached HEAD'; this is normal when downloading a fixed release.

The PostgreSQL scripts are in `$REPOS_DIR/CommonDataModel/inst/ddl/5.4/postgresql/`.

### 4.2 Create the tables

The scripts contain the placeholder `@cdmDatabaseSchema`. It is replaced with the schema name `public` while the
script is sent to the `psql` inside the container (the file itself is not changed):

```bash
sed 's/@cdmDatabaseSchema/public/g' \
  "$REPOS_DIR/CommonDataModel/inst/ddl/5.4/postgresql/OMOPCDM_postgresql_5.4_ddl.sql" \
  | docker exec -i omop-postgres psql -U postgres -d omop_v5 -q -v ON_ERROR_STOP=1
```

`ON_ERROR_STOP=1` stops at the first error, so a problem is not hidden in the output.

The indices are created later (Step 8), after the data is loaded, because loading into indexed tables is much slower.

### 4.3 Check

```bash
docker exec omop-postgres psql -U postgres -d omop_v5 -c \
  "SELECT count(*) FROM information_schema.tables WHERE table_schema='public' AND table_type='BASE TABLE';"
```

Expected: `39`.

---

## Step 5. Adjust columns for the source data

**Purpose:** make four columns of the standard CDM tables accept values that occur in the vocabulary and in the
clinical data, so that the loads in Steps 6 and 7 do not fail.

| Column | Standard CDM 5.4 | Changed to | Reason |
|---|---|---|---|
| `concept.concept_name` | `varchar(255)` | `text` | allows long concept names |
| `concept_synonym.concept_synonym_name` | `varchar(1000)` | `text` | some Athena synonyms are longer than 1,000 characters |
| `drug_exposure.drug_source_value` | `varchar(50)` | `varchar(255)` | drug source values in the source data are up to about 150 characters |
| `drug_exposure.drug_exposure_end_date` | `NOT NULL` | allows NULL | the source data has no drug end dates |

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
ALTER TABLE concept ALTER COLUMN concept_name TYPE text;
ALTER TABLE concept_synonym ALTER COLUMN concept_synonym_name TYPE text;
ALTER TABLE drug_exposure ALTER COLUMN drug_source_value TYPE varchar(255);
ALTER TABLE drug_exposure ALTER COLUMN drug_exposure_end_date DROP NOT NULL;
SQL
```

Expected: `ALTER TABLE` printed four times.

Commands with `<<'SQL'` ... `SQL` send several lines of SQL at once. The last line `SQL` must start at the
beginning of the line, with no spaces before it; otherwise the terminal shows `heredoc>` and keeps waiting.

### 5.1 Check

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 <<'SQL'
SELECT table_name, column_name, data_type, character_maximum_length, is_nullable
FROM information_schema.columns
WHERE (table_name, column_name) IN (
  ('concept', 'concept_name'),
  ('concept_synonym', 'concept_synonym_name'),
  ('drug_exposure', 'drug_source_value'),
  ('drug_exposure', 'drug_exposure_end_date'))
ORDER BY 1, 2;
SQL
```

Expected: `concept_name` and `concept_synonym_name` are `text`, `drug_source_value` is `character varying` with
length 255, and `drug_exposure_end_date` has `is_nullable` = `YES`.

---

## Step 6. Load the OMOP vocabularies

**Purpose:** fill the nine vocabulary tables (CONCEPT, CONCEPT_RELATIONSHIP, CONCEPT_ANCESTOR, ...) with the OMOP
Standardized Vocabularies from OHDSI Athena. All clinical codes in Step 7 refer to these concepts.

### 6.1 Download the vocabularies from Athena

1. Create a free account at https://athena.ohdsi.org/ and sign in.
2. Open **Download** and select all vocabularies that are not marked "License required", plus **CPT4**
   (marked "EULA required"; it is covered by your UMLS account, see 6.2). Selecting all of them makes sure every code
   in your source data has a concept. Request the download. Athena sends an e-mail with a link to a zip file when it
   is ready; for a large selection this can take an hour or more.
3. Unzip the file into `$VOCAB_DIR`. It contains `CONCEPT.csv`, `CONCEPT_ANCESTOR.csv`, `CONCEPT_CLASS.csv`,
   `CONCEPT_RELATIONSHIP.csv`, `CONCEPT_SYNONYM.csv`, `DOMAIN.csv`, `DRUG_STRENGTH.csv`, `RELATIONSHIP.csv`,
   `VOCABULARY.csv`, and, if CPT4 was selected, `CONCEPT_CPT4.csv` (CPT4 concepts without names) and the CPT4 tool
   (`cpt4.jar`, `cpt.sh`).

The release you downloaded is shown in the first data line of `VOCABULARY.csv` (vocabulary `None`):

```bash
grep '^None' "$VOCAB_DIR/VOCABULARY.csv"
```

### 6.2 Add the CPT4 concept names

For licensing reasons, Athena delivers CPT4 concepts without names. The CPT4 tool adds them to `CONCEPT.csv`
using your UMLS API key (free account at https://uts.nlm.nih.gov/; the key is on your UMLS profile page).
The tool needs Java; here it runs in a temporary Java container, so Java does not have to be installed:

```bash
docker run --rm -v "$VOCAB_DIR":/vocab -w /vocab eclipse-temurin:17-jre \
  java -Dumls-apikey=<your_umls_api_key> -jar cpt4.jar 5
```

This changes `CONCEPT.csv`; do not interrupt it. Check that every CPT4 concept now has a name: the first command
below should print the number of rows in `CONCEPT_CPT4.csv` minus the header line, and the second `0`:

```bash
awk -F'\t' '$4=="CPT4"' "$VOCAB_DIR/CONCEPT.csv" | wc -l
awk -F'\t' '$4=="CPT4" && $2==""' "$VOCAB_DIR/CONCEPT.csv" | wc -l
```

Skip this step if you did not select CPT4.

### 6.3 Load the files

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
\timing on
\copy concept FROM '/vocab/CONCEPT.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
\copy concept_relationship FROM '/vocab/CONCEPT_RELATIONSHIP.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
\copy concept_ancestor FROM '/vocab/CONCEPT_ANCESTOR.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
\copy concept_synonym FROM '/vocab/CONCEPT_SYNONYM.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
\copy concept_class FROM '/vocab/CONCEPT_CLASS.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
\copy domain FROM '/vocab/DOMAIN.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
\copy drug_strength FROM '/vocab/DRUG_STRENGTH.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
\copy relationship FROM '/vocab/RELATIONSHIP.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
\copy vocabulary FROM '/vocab/VOCABULARY.csv' WITH (FORMAT csv, DELIMITER E'\t', HEADER, QUOTE E'\b')
SQL
```

- The files are tab-separated. `QUOTE E'\b'` sets the quote character to one that never occurs in the files, so
  quotation marks inside concept names are read as normal text.
- Each table prints `COPY <number of rows>` and the time taken. Loading is fast because the indices are created
  later (Step 8); in our test the largest tables (CONCEPT_RELATIONSHIP, CONCEPT_ANCESTOR) took less than 30 seconds
  each and all nine about one minute.
- If a table fails, fix the problem, empty that table (`TRUNCATE <table>;`) and load it again.
- If `\copy` reports `Operation not permitted` for a file in `/vocab`, the container cannot read `$VOCAB_DIR`.
  This can happen when the folder was replaced after the container was started, or when unzipping created it with
  owner-only permissions. Make the folder readable and restart the container, then load again:

  ```bash
  chmod 755 "$VOCAB_DIR"
  docker restart omop-postgres
  docker exec omop-postgres ls /vocab
  ```

### 6.4 Check

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 <<'SQL'
SELECT 'concept' AS table_name, count(*) FROM concept UNION ALL
SELECT 'concept_relationship', count(*) FROM concept_relationship UNION ALL
SELECT 'concept_ancestor', count(*) FROM concept_ancestor UNION ALL
SELECT 'concept_synonym', count(*) FROM concept_synonym UNION ALL
SELECT 'concept_class', count(*) FROM concept_class UNION ALL
SELECT 'domain', count(*) FROM domain UNION ALL
SELECT 'drug_strength', count(*) FROM drug_strength UNION ALL
SELECT 'relationship', count(*) FROM relationship UNION ALL
SELECT 'vocabulary', count(*) FROM vocabulary;
SELECT vocabulary_version FROM vocabulary WHERE vocabulary_id = 'None';
SQL
```

Expected: no table is empty, and each count equals the number of `COPY` rows printed in 6.3. The last query shows
the vocabulary release.

## Step 7. Load the clinical data

The clinical CSV files usually do not have the same columns, in the same order, as the CDM tables, so they cannot be
copied into the CDM tables directly. Each file is therefore loaded in two steps:

1. Into a **staging table**: a temporary table created only for loading, with exactly the columns of the CSV file in
   the same order. Values that need conversion (for example dates stored as text) are kept as text here.
2. From the staging table into the CDM table with `INSERT ... SELECT`, which puts each column in its CDM place and
   converts the types. CDM columns that are not in the file stay empty (NULL).

The staging table is temporary and disappears when the command ends. Each block starts by emptying the CDM table
(`TRUNCATE`), so running a block again reloads the table instead of adding the rows a second time. If your files
have other columns or another column order, change the staging table and the `INSERT` accordingly.

### 7.1 Add the files

Copy the clinical CSV files into `$DATA_DIR`. The container sees this folder as `/data`. Check that the container
sees the files:

```bash
docker exec omop-postgres ls -l /data
```

### 7.2 PERSON

CSV columns: `person_id, gender_concept_id, year_of_birth, month_of_birth, day_of_birth, birth_datetime,
race_source_value, race_concept_id, ethnicity_source_value, ethnicity_concept_id, care_site_id, gender_source_value`

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
TRUNCATE person;
CREATE TEMP TABLE person_staging (
    person_id INT,
    gender_concept_id INT,
    year_of_birth INT,
    month_of_birth INT,
    day_of_birth INT,
    birth_datetime TEXT,
    race_source_value TEXT,
    race_concept_id INT,
    ethnicity_source_value TEXT,
    ethnicity_concept_id INT,
    care_site_id INT,
    gender_source_value TEXT
);
\copy person_staging FROM '/data/PERSON.csv' WITH (FORMAT csv, HEADER)
INSERT INTO person (
    person_id, gender_concept_id, year_of_birth, month_of_birth, day_of_birth, birth_datetime,
    race_concept_id, ethnicity_concept_id, care_site_id,
    gender_source_value, race_source_value, ethnicity_source_value
)
SELECT
    person_id, gender_concept_id, year_of_birth, month_of_birth, day_of_birth, birth_datetime::timestamp,
    race_concept_id, ethnicity_concept_id, care_site_id,
    gender_source_value, race_source_value, ethnicity_source_value
FROM person_staging;
SQL
```

Check: the number of persons should equal the number of lines in `PERSON.csv` minus the header line.

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -c "SELECT count(*) AS persons FROM person;"
```

### 7.3 DEATH

CSV columns: `person_id, death_datetime, death_date, death_type_concept_id`

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
TRUNCATE death;
CREATE TEMP TABLE death_staging (
    person_id INT,
    death_datetime TEXT,
    death_date TEXT,
    death_type_concept_id INT
);
\copy death_staging FROM '/data/DEATH.csv' WITH (FORMAT csv, HEADER)
INSERT INTO death (person_id, death_date, death_datetime, death_type_concept_id)
SELECT person_id, death_date::date, death_datetime::timestamp, death_type_concept_id
FROM death_staging;
SQL
```

Check: the number of deaths should equal the number of lines in `DEATH.csv` minus the header line.

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -c "SELECT count(*) AS deaths FROM death;"
```

### 7.4 CONDITION_OCCURRENCE

CSV columns: `person_id, visit_occurrence_id, condition_occurrence_id, condition_concept_id, condition_start_date,
condition_type_concept_id, condition_source_value`

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
TRUNCATE condition_occurrence;
CREATE TEMP TABLE condition_occurrence_staging (
    person_id INT,
    visit_occurrence_id INT,
    condition_occurrence_id INT,
    condition_concept_id INT,
    condition_start_date TEXT,
    condition_type_concept_id INT,
    condition_source_value TEXT
);
\copy condition_occurrence_staging FROM '/data/CONDITION_OCCURRENCE.csv' WITH (FORMAT csv, HEADER)
INSERT INTO condition_occurrence (
    condition_occurrence_id, person_id, condition_concept_id, condition_start_date,
    condition_type_concept_id, visit_occurrence_id, condition_source_value
)
SELECT
    condition_occurrence_id, person_id, condition_concept_id, condition_start_date::date,
    condition_type_concept_id, visit_occurrence_id, condition_source_value
FROM condition_occurrence_staging;
SQL
```

`condition_source_concept_id` is not in the file; it is filled from the vocabulary later (Step 7.8).

Check: the number of rows should equal the number of lines in `CONDITION_OCCURRENCE.csv` minus the header line.

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -c "SELECT count(*) AS conditions FROM condition_occurrence;"
```

### 7.5 PROCEDURE_OCCURRENCE

CSV columns: `provider_id, procedure_source_value, modifier_source_value, person_id, visit_occurrence_id,
procedure_occurrence_id, procedure_concept_id, procedure_date, procedure_type_concept_id`

The `provider_id` column is read into the staging table but not copied: the PROVIDER table is not loaded, so the
provider IDs would not refer to anything.

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
TRUNCATE procedure_occurrence;
CREATE TEMP TABLE procedure_occurrence_staging (
    provider_id TEXT,
    procedure_source_value TEXT,
    modifier_source_value TEXT,
    person_id INT,
    visit_occurrence_id INT,
    procedure_occurrence_id INT,
    procedure_concept_id INT,
    procedure_date TEXT,
    procedure_type_concept_id INT
);
\copy procedure_occurrence_staging FROM '/data/PROCEDURE_OCCURRENCE.csv' WITH (FORMAT csv, HEADER)
INSERT INTO procedure_occurrence (
    procedure_occurrence_id, person_id, procedure_concept_id, procedure_date, procedure_type_concept_id,
    visit_occurrence_id, procedure_source_value, modifier_source_value
)
SELECT
    procedure_occurrence_id, person_id, procedure_concept_id, procedure_date::date, procedure_type_concept_id,
    visit_occurrence_id, procedure_source_value, modifier_source_value
FROM procedure_occurrence_staging;
SQL
```

`procedure_source_concept_id` is not in the file; it is filled from the vocabulary later (Step 7.8).

Check: the number of rows should equal the number of lines in `PROCEDURE_OCCURRENCE.csv` minus the header line.

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -c "SELECT count(*) AS procedures FROM procedure_occurrence;"
```

### 7.6 DRUG_EXPOSURE

CSV columns: `drug_type_concept_id, drug_concept_id, drug_exposure_id, drug_source_concept_id, visit_occurrence_id,
person_id, drug_source_value, drug_exposure_start_datetime`

The file has a start date and time but no separate start date, so `drug_exposure_start_date` is taken from the date
part of `drug_exposure_start_datetime`. The file has no end date; `drug_exposure_end_date` stays empty (Step 5 allows
this).

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
TRUNCATE drug_exposure;
CREATE TEMP TABLE drug_exposure_staging (
    drug_type_concept_id INT,
    drug_concept_id INT,
    drug_exposure_id INT,
    drug_source_concept_id INT,
    visit_occurrence_id INT,
    person_id INT,
    drug_source_value TEXT,
    drug_exposure_start_datetime TEXT
);
\copy drug_exposure_staging FROM '/data/DRUG_EXPOSURE.csv' WITH (FORMAT csv, HEADER)
INSERT INTO drug_exposure (
    drug_exposure_id, person_id, drug_concept_id, drug_exposure_start_date, drug_exposure_start_datetime,
    drug_type_concept_id, visit_occurrence_id, drug_source_value, drug_source_concept_id
)
SELECT
    drug_exposure_id, person_id, drug_concept_id, drug_exposure_start_datetime::timestamp::date,
    drug_exposure_start_datetime::timestamp,
    drug_type_concept_id, visit_occurrence_id, drug_source_value, drug_source_concept_id
FROM drug_exposure_staging;
SQL
```

Check: the number of rows should equal the number of lines in `DRUG_EXPOSURE.csv` minus the header line.

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -c "SELECT count(*) AS drug_exposures FROM drug_exposure;"
```

### 7.7 MEASUREMENT

CSV columns: `measurement_type_concept_id, value_as_concept_id, measurement_concept_id, measurement_source_value,
measurement_id, person_id, visit_occurrence_id, measurement_date, measurement_source_concept_id, unit_source_value,
value_as_number, operator_concept_id, value_source_value, unit_concept_id`

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
TRUNCATE measurement;
CREATE TEMP TABLE measurement_staging (
    measurement_type_concept_id INT,
    value_as_concept_id INT,
    measurement_concept_id INT,
    measurement_source_value TEXT,
    measurement_id INT,
    person_id INT,
    visit_occurrence_id INT,
    measurement_date TEXT,
    measurement_source_concept_id INT,
    unit_source_value TEXT,
    value_as_number NUMERIC,
    operator_concept_id INT,
    value_source_value TEXT,
    unit_concept_id INT
);
\copy measurement_staging FROM '/data/MEASUREMENT.csv' WITH (FORMAT csv, HEADER)
INSERT INTO measurement (
    measurement_id, person_id, measurement_concept_id, measurement_date, measurement_type_concept_id,
    operator_concept_id, value_as_number, value_as_concept_id, unit_concept_id, visit_occurrence_id,
    measurement_source_value, measurement_source_concept_id, unit_source_value, value_source_value
)
SELECT
    measurement_id, person_id, measurement_concept_id, measurement_date::date, measurement_type_concept_id,
    operator_concept_id, value_as_number, value_as_concept_id, unit_concept_id, visit_occurrence_id,
    measurement_source_value, measurement_source_concept_id, unit_source_value, value_source_value
FROM measurement_staging;
SQL
```

Check: the number of rows should equal the number of lines in `MEASUREMENT.csv` minus the header line.

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -c "SELECT count(*) AS measurements FROM measurement;"
```

### 7.8 Fill the concept IDs from the source codes

Some rows have no standard concept yet (`condition_concept_id` or `procedure_concept_id` is `0`), and the source
concept IDs are empty. They are filled from the vocabulary in two steps:

1. **Source concept:** find the concept whose code equals the source value (`condition_source_value`,
   `procedure_source_value`) in the source vocabulary: ICD9CM or ICD10CM for conditions, CPT4 for procedures.
2. **Standard concept:** follow the source concept's valid `Maps to` link to the standard concept.

Rules, so that the result is the same every time:

- A code that exists in both ICD9CM and ICD10CM is taken from ICD9CM for dates before 1 October 2015 and from
  ICD10CM from that date on (the date the US switched to ICD-10-CM). If the code exists only in the other vocabulary,
  that one is used.
- Only valid `Maps to` links are used (`invalid_reason IS NULL`).
- If a source concept maps to more than one standard concept, the one with the lowest `concept_id` is used, so the
  number of rows does not change.
- Rows whose code is not found, or whose source concept has no valid `Maps to` link, keep concept `0`.

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 <<'SQL'
\timing on
-- conditions, step 1a: source concept from the vocabulary that matches the date
UPDATE condition_occurrence co
SET condition_source_concept_id = c.concept_id
FROM concept c
WHERE co.condition_concept_id = 0
  AND co.condition_source_concept_id IS NULL
  AND c.concept_code = co.condition_source_value
  AND c.vocabulary_id = CASE WHEN co.condition_start_date < DATE '2015-10-01' THEN 'ICD9CM' ELSE 'ICD10CM' END;
-- conditions, step 1b: codes that exist only in the other vocabulary
UPDATE condition_occurrence co
SET condition_source_concept_id = c.concept_id
FROM concept c
WHERE co.condition_concept_id = 0
  AND co.condition_source_concept_id IS NULL
  AND c.concept_code = co.condition_source_value
  AND c.vocabulary_id IN ('ICD9CM', 'ICD10CM');
-- conditions, step 2: standard concept through 'Maps to'
UPDATE condition_occurrence co
SET condition_concept_id = m.concept_id_2
FROM (SELECT concept_id_1, min(concept_id_2) AS concept_id_2
      FROM concept_relationship
      WHERE relationship_id = 'Maps to' AND invalid_reason IS NULL
      GROUP BY concept_id_1) m
WHERE co.condition_concept_id = 0
  AND m.concept_id_1 = co.condition_source_concept_id;
-- procedures, step 1: source concept from CPT4
UPDATE procedure_occurrence po
SET procedure_source_concept_id = c.concept_id
FROM concept c
WHERE po.procedure_source_concept_id IS NULL
  AND c.concept_code = po.procedure_source_value
  AND c.vocabulary_id = 'CPT4';
-- procedures, step 2: standard concept through 'Maps to'
UPDATE procedure_occurrence po
SET procedure_concept_id = m.concept_id_2
FROM (SELECT concept_id_1, min(concept_id_2) AS concept_id_2
      FROM concept_relationship
      WHERE relationship_id = 'Maps to' AND invalid_reason IS NULL
      GROUP BY concept_id_1) m
WHERE po.procedure_concept_id = 0
  AND m.concept_id_1 = po.procedure_source_concept_id;
SQL
```

Each `UPDATE` prints the number of rows it changed. Running the block again changes nothing, because it only fills
rows that are still empty or `0`.

Check: for each table, the total number of rows, how many have a source concept, how many still have concept `0`,
and how many source concepts do not match their own source code (this last number must be `0`):

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 <<'SQL'
SELECT 'condition_occurrence' AS table_name, count(*) AS total_rows,
       count(co.condition_source_concept_id) AS with_source_concept,
       count(*) FILTER (WHERE co.condition_concept_id = 0) AS concept_0,
       count(*) FILTER (WHERE co.condition_source_concept_id IS NOT NULL
                          AND c.concept_code IS DISTINCT FROM co.condition_source_value) AS source_mismatch
FROM condition_occurrence co
LEFT JOIN concept c ON c.concept_id = co.condition_source_concept_id
UNION ALL
SELECT 'procedure_occurrence', count(*),
       count(po.procedure_source_concept_id),
       count(*) FILTER (WHERE po.procedure_concept_id = 0),
       count(*) FILTER (WHERE po.procedure_source_concept_id IS NOT NULL
                          AND c.concept_code IS DISTINCT FROM po.procedure_source_value)
FROM procedure_occurrence po
LEFT JOIN concept c ON c.concept_id = po.procedure_source_concept_id;
SQL
```

## Step 8. Create the indices

**Purpose:** make queries fast. The official indices script from the same CommonDataModel release creates the
indices and then sorts (`CLUSTER`) the main tables by their most used index.

```bash
sed 's/@cdmDatabaseSchema/public/g' \
  "$REPOS_DIR/CommonDataModel/inst/ddl/5.4/postgresql/OMOPCDM_postgresql_5.4_indices.sql" \
  | docker exec -i -e PGOPTIONS='-c maintenance_work_mem=512MB' omop-postgres \
    psql -U postgres -d omop_v5 -q -v ON_ERROR_STOP=1
```

- This is the slowest step: the large vocabulary tables (CONCEPT_RELATIONSHIP, CONCEPT_ANCESTOR) take the longest.
  Do not interrupt it.
- `CLUSTER` rewrites each table, so for a while the largest table needs about twice its size in free disk space.
- `maintenance_work_mem=512MB` gives PostgreSQL more memory for building the indices, which makes it faster. It
  applies only to this command.

### 8.1 Check

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 <<'SQL'
SELECT count(*) AS indices FROM pg_indexes WHERE schemaname = 'public';
SELECT count(*) AS clustered_tables FROM pg_index WHERE indisclustered;
SQL
```

Expected: `70` indices and `32` clustered tables.

This completes Part A.

---

# Part B. OMOPonFHIR server

All steps in Part B use the folders and the `.env` file from Part A, Step 2. In a new terminal, run the `export`
lines from Part A, Step 2.1 again.

## Step 1. Add the OMOPonFHIR tables and views

**Purpose:** OMOPonFHIR needs a few extra objects in the database: two tables (`f_person`, `f_cache`), two views
(`f_immunization_view`, `f_observation_view`), and longer text columns in `observation`. Their scripts are in the
OMOPonFHIR repository `omopv5_4_setup`.

### 1.1 Download the scripts

```bash
git clone https://github.com/omoponfhir/omopv5_4_setup.git "$REPOS_DIR/omopv5_4_setup"
git -C "$REPOS_DIR/omopv5_4_setup" checkout bed014d1e78f82e63e37623883a6216e21a46819
```

The second command fixes the version of the scripts (the version tested with this guide). Git reports a
"detached HEAD"; that is expected.

### 1.2 Run the scripts

The container sees `$REPOS_DIR` as `/repos`, so the scripts are run directly from there. The order matters: the
`observation` columns must be changed before `f_observation_view` is created, because a column cannot be changed
once a view uses it.

```bash
docker exec omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 \
  -f /repos/omopv5_4_setup/omoponfhir_f_person_ddl.txt \
  -f /repos/omopv5_4_setup/omoponfhir_f_cache_ddl.txt \
  -f /repos/omopv5_4_setup/omoponfhir_v5.2_f_immunization_view_ddl.txt \
  -f /repos/omopv5_4_setup/observation_column_size_change_ddl.txt \
  -f /repos/omopv5_4_setup/omoponfhir_v5.4_f_observation_ddl.txt
```

Expected output: `CREATE TABLE` twice, `CREATE VIEW`, `ALTER TABLE` twice, `CREATE VIEW`.

The immunization view script is named v5.2, but it is the one the repository provides and it works with CDM 5.4.

### 1.3 Fill f_person

OMOPonFHIR finds patients through `f_person`: a patient that is in `person` but not in `f_person` is not returned
by the FHIR server. `f_person` holds data that OMOP does not have, such as names. Add one row per patient, with only
the `person_id`; the other columns stay empty:

```bash
docker exec omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 \
  -c "INSERT INTO f_person (person_id) SELECT person_id FROM person;"
```

Expected: `INSERT 0 <n>`, where `<n>` is the number of patients in `person`.

The `omopv5_4_setup` repository also has scripts (`fhir_names/`) that give every patient a random made-up name. They
are not used here: the names would be fake, and different each time the scripts are run.

### 1.4 Check

```bash
docker exec -i omop-postgres psql -U postgres -d omop_v5 <<'SQL'
SELECT (SELECT count(*) FROM f_person) AS f_person_rows, (SELECT count(*) FROM person) AS person_rows;
SELECT table_name, table_type FROM information_schema.tables
WHERE table_schema = 'public' AND table_name LIKE 'f\_%' ORDER BY table_name;
SELECT column_name, data_type, character_maximum_length FROM information_schema.columns
WHERE table_name = 'observation' AND column_name IN ('observation_source_value', 'value_as_string');
SELECT (SELECT count(*) FROM f_observation_view) AS f_observation_view_rows,
       (SELECT count(*) FROM measurement) + (SELECT count(*) FROM observation) AS measurement_plus_observation_rows;
SELECT count(*) AS f_immunization_view_rows FROM f_immunization_view;
SQL
```

Expected: the same number for `f_person_rows` and `person_rows`; `f_cache` and `f_person` (`BASE TABLE`),
`f_immunization_view` and `f_observation_view` (`VIEW`); both `observation` columns `character varying` with length
500; the two row counts of the fourth query are equal; the last query returns a number without an error.

## Step 2. Download the OMOPonFHIR server code

**Purpose:** get the source code of the OMOPonFHIR server (FHIR R4 on OMOP CDM 5.4). The image is built from it in
Step 3. The repository has three parts as git submodules (`omoponfhir-r4-server`, `omoponfhir-omopv5-r4-mapping`,
`omoponfhir-omopv5-sql`).

```bash
git clone https://github.com/mcc-ad/UTH-omoponfhir-main-v54-r4.git "$REPOS_DIR/UTH-omoponfhir-main-v54-r4"
git -C "$REPOS_DIR/UTH-omoponfhir-main-v54-r4" checkout 468e4568961dcc72b1a2b37bc2d25be109d77192
git -C "$REPOS_DIR/UTH-omoponfhir-main-v54-r4" submodule update --init
```

The second command fixes the version of the code (the version tested with this guide); git reports a
"detached HEAD", which is expected. The third command downloads the three submodules at the versions this commit
refers to.

### 2.1 Check

```bash
git -C "$REPOS_DIR/UTH-omoponfhir-main-v54-r4" log -1 --format=%H
git -C "$REPOS_DIR/UTH-omoponfhir-main-v54-r4" submodule status
```

Expected: the first command prints `468e4568961dcc72b1a2b37bc2d25be109d77192`. The second prints three lines:

```
 5fe5b610920633e0c4aacf16fae94566eea30b10 omoponfhir-omopv5-r4-mapping (...)
 b826188c13ad0afb4e01660df450fcca663f1aa2 omoponfhir-omopv5-sql (...)
 199a52ad8c47bfdff7b6a0d15044a9945a45ddf3 omoponfhir-r4-server (...)
```

Each line must start with a space. A `-` at the start means the submodule was not downloaded; a `+` means it is at a
different version. In both cases run the `submodule update --init` command again.

## Step 3. Build the OMOPonFHIR image

**Purpose:** compile the server code (Maven) and package it in a Tomcat web server image. Both happen inside Docker,
as described in the repository's `Dockerfile`, so Java and Maven are not needed on the Mac.

### 3.1 Fix the log4j versions

Without this change the image builds, but the server does not start: the log shows
`NoSuchMethodError: ... org.apache.logging.log4j.util.LoaderUtil.getClassLoaders()`. The reason is that the code
gets two parts of the log4j library in different versions (`log4j-api` 2.19.0 and `log4j-core` 2.17.1). Setting
both to the same version fixes it.

Open `$REPOS_DIR/UTH-omoponfhir-main-v54-r4/omoponfhir-r4-server/pom.xml` in a text editor, find the dependency
`logback-classic`, and add these lines directly after its closing `</dependency>`:

```xml
		<dependency>
			<groupId>org.apache.logging.log4j</groupId>
			<artifactId>log4j-api</artifactId>
			<version>2.17.1</version>
		</dependency>
		<dependency>
			<groupId>org.apache.logging.log4j</groupId>
			<artifactId>log4j-core</artifactId>
			<version>2.17.1</version>
		</dependency>
```

Older instructions add the same two dependencies with version 2.8.2. Do not use 2.8.2: it has a known security
problem (CVE-2021-44228, "Log4Shell"). Version 2.17.1 fixes it and is the version the code itself asks for.

Check the change:

```bash
git -C "$REPOS_DIR/UTH-omoponfhir-main-v54-r4/omoponfhir-r4-server" diff
```

Expected: only the 10 lines above, marked with `+`.

### 3.2 Build the image

```bash
docker build -t omoponfhir:mcc-ad "$REPOS_DIR/UTH-omoponfhir-main-v54-r4"
```

- The image is named `omoponfhir:mcc-ad`, after the repository it is built from (Step 2).
- The first build downloads the build tools and all libraries and takes several minutes.
- If the build stops with `exit code: 137`, Docker ran out of memory: give the Rancher Desktop virtual machine at least
  4 GB (Part A, Step 1.1) and run the command again.
- The build may end with the warning `FromAsCasing: 'as' and 'FROM' keywords' casing do not match`. It is about the
  spelling in the `Dockerfile` and can be ignored.

### 3.3 Check

```bash
docker image ls omoponfhir:mcc-ad
```

Expected: one line with the image `omoponfhir`, tag `mcc-ad`.

**Tested with:** Tomcat 11.0.27 and Java 21.0.12 (from the `tomcat:jre21` base image named in the `Dockerfile`, at
the time of testing); Maven build step about 2 minutes on Apple M3; image about 900 MB.

`tomcat:jre21` always points to the newest Tomcat with Java 21, so a later build may use a newer Tomcat. To check
which one an image has: `docker exec omoponfhir /usr/local/tomcat/bin/version.sh | grep -E "Server number|JVM Version"`.

## Step 4. Start the OMOPonFHIR server

**Purpose:** run the image from Step 3 as a container on `omop-network`, next to the database container. All settings
are passed as environment variables when the container starts; the image itself is not changed.

### 4.1 Set the passwords for the server

Create a second file, `omoponfhir.env`, next to the `.env` from Part A, Step 2.2:

```bash
cat > "$REPOS_DIR/omoponfhir.env" <<'EOF2'
JDBC_PASSWORD=<your_password>
AUTH_BASIC=<fhir_user>:<fhir_password>
AUTH_BEARER=<fhir_token>
EOF2
chmod 600 "$REPOS_DIR/omoponfhir.env"
```

Replace the placeholders, without quotes:

- `<your_password>`: the database password from Part A, Step 2.2.
- `<fhir_user>:<fhir_password>`: a user name and password of your choice for logging in to the FHIR server. The
  password must not contain `:`.
- `<fhir_token>`: a token of your choice for programs that log in with `Authorization: Bearer ...`. A random one can be
  made with `openssl rand -hex 16`.

If `AUTH_BASIC` or `AUTH_BEARER` is missing, the server falls back to built-in values that are the same for every
installation (`client_omop:secret` and `12345`), so always set both. Do not commit this file to git.

### 4.2 Start the container

```bash
docker run -d \
  --name omoponfhir \
  --network omop-network \
  --env-file "$REPOS_DIR/omoponfhir.env" \
  -e JDBC_URL=jdbc:postgresql://omop-postgres:5432/omop_v5 \
  -e JDBC_USERNAME=postgres \
  -e JDBC_DATASOURCENAME=org.postgresql.ds.PGSimpleDataSource \
  -e JDBC_POOLSIZE=10 \
  -e JDBC_DATA_SCHEMA=public \
  -e JDBC_VOCABS_SCHEMA=public \
  -e SERVERBASE_URL=http://localhost:8080/fhir \
  -e FHIR_READONLY=True \
  -p 127.0.0.1:8080:8080 \
  --restart unless-stopped \
  omoponfhir:mcc-ad
```

- `omop-postgres:5432` is the database container's name on `omop-network`, so no IP address is needed. Port 5432 is
  the port inside the network, even if you changed the port on the Mac in Part A, Step 2.4.
- `JDBC_DATA_SCHEMA` and `JDBC_VOCABS_SCHEMA` say where the clinical and the vocabulary tables are; in Part A both are
  in the schema `public`.
- `FHIR_READONLY=True` lets the server read the database but not write to it. This is also the built-in default; it
  is set here so that it is visible.
- `-p 127.0.0.1:8080:8080` makes the server available at `http://localhost:8080`, but only on this Mac, not to other
  computers on the network. This matters because the web page at `http://localhost:8080/` (the HAPI FHIR test page)
  does not ask for a login: it sends the `AUTH_BASIC` login from `omoponfhir.env` itself. If port 8080 is already in
  use, use for example `-p 127.0.0.1:8081:8080` and change `SERVERBASE_URL` to match.
- If a container named `omoponfhir` already exists (for example from an earlier try), remove it first with
  `docker rm -f omoponfhir`. This removes only the container, not the image or the database.

### 4.3 Check

The server needs up to a minute to start. Then:

```bash
docker ps --filter name=omoponfhir
docker logs omoponfhir 2>&1 | grep -E "Server startup in|SEVERE"
```

Expected: the container is `Up`, and the log contains `Server startup in [...] milliseconds` and no `SEVERE` lines.

If there are `SEVERE` lines with `NoSuchMethodError ... LoaderUtil.getClassLoaders()`, the log4j fix from Step 3.1 is
missing: make the change, build again (Step 3.2), and start the container again (Step 4.2).

### 4.4 If the data changes later

The server stores the number of results of each search in the table `f_cache` and does not recalculate it by itself.
If you change the data in the database after the server has run (for example, load more clinical data in Part A,
Step 7, or fill `f_person` again), empty the cache, or the server keeps returning the old numbers (for example, no
patients):

```bash
docker exec omop-postgres psql -U postgres -d omop_v5 -v ON_ERROR_STOP=1 -c "DELETE FROM f_cache;"
```

This removes only the stored numbers, not data. The server does not need to be restarted.

## Step 5. Test the FHIR server

**Purpose:** check that the server answers FHIR requests, reads the OMOP data, and asks for the login from Step 4.1.
The commands print only HTTP status codes and resource types, no patient data.

### 5.1 Server description (no login needed)

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/fhir/metadata
curl -s http://localhost:8080/fhir/metadata | grep -oE '"fhirVersion" ?: ?"[^"]+"'
```

Expected: `200`, and `"fhirVersion": "4.0.1"` (FHIR R4). The server description (`CapabilityStatement`) is public on
every FHIR server; all other requests need a login.

### 5.2 Requests without login are refused

```bash
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:8080/fhir/Patient?_count=1"
```

Expected: `401` (not authorized).

### 5.3 Read a patient with the user name and password

The login is read from `omoponfhir.env`, so it is not typed on the command line:

```bash
AUTH_BASIC=$(sed -n 's/^AUTH_BASIC=//p' "$REPOS_DIR/omoponfhir.env")
curl -s -o /dev/null -w '%{http_code}\n' -u "$AUTH_BASIC" "http://localhost:8080/fhir/Patient?_count=1"
curl -s -u "$AUTH_BASIC" "http://localhost:8080/fhir/Patient?_count=1" | grep -oE '"resourceType" ?: ?"[A-Za-z]+"' | sort | uniq -c
unset AUTH_BASIC
```

Expected: `200`, then `1` × `Bundle` (the search result) and `1` × `Patient` (one patient from the `person` table).

### 5.4 Read a patient with the token

```bash
AUTH_BEARER=$(sed -n 's/^AUTH_BEARER=//p' "$REPOS_DIR/omoponfhir.env")
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $AUTH_BEARER" "http://localhost:8080/fhir/Patient?_count=1"
unset AUTH_BEARER
```

Expected: `200`.

### 5.5 The server does not write

```bash
AUTH_BASIC=$(sed -n 's/^AUTH_BASIC=//p' "$REPOS_DIR/omoponfhir.env")
curl -s -o /dev/null -w '%{http_code}\n' -u "$AUTH_BASIC" -X POST \
  -H "Content-Type: application/fhir+json" -d '{"resourceType":"Patient"}' http://localhost:8080/fhir/Patient
unset AUTH_BASIC
```

Expected: `405` (method not allowed), because `FHIR_READONLY=True` (Step 4.2). To check that nothing was written:

```bash
docker exec omop-postgres psql -U postgres -d omop_v5 -c "SELECT count(*) AS person_rows FROM person;"
```

Expected: the same number of patients as before.
