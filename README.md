# OMOPonFHIR-main-v54-r4 Installation and Configuration

This guide provides detailed steps to install and configure OMOPonFHIR-main-v54-r4 using **Rancher Desktop** (configured as containerd), with a PostgreSQL database for OMOPv5.4 hosted outside of Kubernetes.

  
## Step 1: Rancher Desktop Installation and Configuration

Rancher Desktop is an open-source application that provides all the essentials to setup a local Kubernetes environment for development. Rancher Desktop can be downloaded from https://rancherdesktop.io/

To follow the next instructions, I am using this version: https://github.com/rancher-sandbox/rancher-desktop/releases/download/v1.15.1/Rancher.Desktop-1.15.1.aarch64.dmg

After installation and running Rancher Desktop, make sure that "Enable Kubernetes" is selected, Container Engine: containered and Configure PATH: Automatic are marked.

## Step 2: Install PostgreSQL for OMOPv5.4

First, set up the PostgreSQL database to host the OMOP Common Data Model (CDM).

### 2.1. Pull PostgreSQL Docker Image

To pull the latest PostgreSQL Docker image, run the following command:

```
nerdctl pull postgres:latest
```

### 2.2. Run PostgreSQL Container

Run PostgreSQL using the following command, which will create a new container named `omop-postgres`:

```
nerdctl run --name omop-postgres -d -p 5432:5432 -e POSTGRES_PASSWORD="put your password here" postgres:latest
```

### 2.3. Create OMOP Database

Log in to the PostgreSQL container:

 
```
nerdctl exec -it omop-postgres psql -U postgres 
```

Create the OMOP database:

```sql
CREATE DATABASE omop_v5; 
```

### 2.4. Download and Run OMOP CDM DDL Scripts

Download the OMOP CDM DDL scripts from [OHDSI CommonDataModel](https://github.com/OHDSI/CommonDataModel) repository or follow the instructions in the [OMOPv5.4 Setup repository](https://github.com/omoponfhir/omopv5_4_setup).

```bash
git clone https://github.com/OHDSI/CommonDataModel.git
```
Note: The previous command may ask to install developer tools. Install it and re-run the command



#### 2.4.1 Install psql Locally Using Homebrew (Recommended for Mac):

If you want to run psql from your Mac’s terminal (outside the container) to interact with the PostgreSQL instance running in the container, you need to install the PostgreSQL client:

```bash
brew install postgresql
```
If brew is not installed you can install it using

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### 2.4.2 Run the DDL scripts to set up the OMOP database schema:

```bash
psql -h localhost -U postgres -d omop_v5 -f /path/to/OMOPCDM_postgresql_5.4_ddl.sql
```
Note: you may encounter an error, please run the following command, first:

```
sed -i '' 's/@cdmDatabaseSchema/public/g' /path/to/OMOPCDM_postgresql_5.4_ddl.sql
```
You may want to run the same steps for the indices file:  OMOPCDM_postgresql_5.4_indices.sql


### 2.5. Load OMOP Vocabularies

1.  Download vocabularies from [Athena OHDSI](https://athena.ohdsi.org/).
2.  Load the vocabulary files into the `omop_v5` database.

```bash
psql -h localhost -U postgres -d omop_v5 -c "\copy DRUG_STRENGTH FROM 'path_to_DRUG_STRENGTH.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT FROM 'path_to_CONCEPT.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT_RELATIONSHIP FROM 'path_to_CONCEPT_RELATIONSHIP.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT_ANCESTOR FROM 'path_to_CONCEPT_ANCESTOR.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT_SYNONYM FROM 'path_to_CONCEPT_SYNONYM.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
psql -h localhost -U postgres -d omop_v5 -c "\copy VOCABULARY FROM 'path_to_VOCABULARY.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
psql -h localhost -U postgres -d omop_v5 -c "\copy RELATIONSHIP FROM 'path_to_RELATIONSHIP.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT_CLASS FROM 'path_to_CONCEPT_CLASS.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
psql -h localhost -U postgres -d omop_v5 -c "\copy DOMAIN FROM 'path_to_DOMAIN.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;" 
```
Make sure all required vocabulary files are loaded.

----------

## Step 3: Deploy OMOPonFHIR in Kubernetes

### 3.1. Clone OMOPonFHIR Repository

Clone the `omoponfhir-main-v54-r4` repository:

```bash
git clone https://github.com/omoponfhir/omoponfhir-main-v54-r4.git
cd omoponfhir-main-v54-r4
```

### 3.2. Build OMOPonFHIR Docker Image

Since Rancher Desktop uses containerd, you’ll build the OMOPonFHIR image using `nerdctl`:

```bash
nerdctl build -t omoponfhir:latest .
```

### 3.3. Configure Kubernetes Deployment

Edit the `omoponfhir-deployment.yaml` file to ensure that the environment variables (specifically JDBC connection details) are set properly to connect to your PostgreSQL database running outside Kubernetes:

```yaml
env:
  - name: JDBC_URL
    value: jdbc:postgresql://host.docker.internal:5432/omop_v5
  - name: JDBC_USERNAME
    value: postgres
  - name: JDBC_PASSWORD
    value: mayopassword
  - name: JDBC_DATA_SCHEMA
    value: omopv54
  - name: JDBC_VOCABS_SCHEMA
    value: vocab
```

Make sure to set the `JDBC_URL` to `host.docker.internal` since PostgreSQL is running on the host machine.

### 3.4. Deploy OMOPonFHIR

Deploy the OMOPonFHIR application in Kubernetes:

```bash
kubectl apply -f omoponfhir-deployment.yaml
kubectl apply -f omoponfhir-service.yaml
```

### 3.5. Expose OMOPonFHIR Service

Expose the OMOPonFHIR service to be accessible from outside Kubernetes. In the service configuration file (`omoponfhir-service.yaml`), ensure you use a `NodePort` service to expose the service on `localhost:30080`:

```
apiVersion: v1
kind: Service
metadata:
  name: omoponfhir-service
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
  selector:
    app: omoponfhir
```

Apply the service configuration:

```bash
kubectl apply -f omoponfhir-service.yaml
```

----------

## Step 4: Access OMOPonFHIR and PostgreSQL

### 4.1. Access OMOPonFHIR

Once OMOPonFHIR is running, you can access the FHIR API at:

```bash
http://localhost:30080/fhir/ 
```

Test resources like `Observation`, `Patient`, etc. by navigating to:

```bash
http://localhost:30080/fhir/Observation
http://localhost:30080/fhir/Patient
```

### 4.2. Access PostgreSQL from Local Machine

If `psql` is not installed locally, you can use `nerdctl` to connect to PostgreSQL:

1.  **Connect to PostgreSQL container:**
    
    ```bash
    nerdctl exec -it omop-postgres psql -U postgres
    ```
    
2.  **Run SQL queries directly from within the container:**
    
    ```bash
    \c omop_v5
    SELECT * FROM observation LIMIT 10;
    ``` 
    

Alternatively, you can install `psql` locally using **Homebrew**:

```bash
brew install libpq
brew link --force libpq
```

Then access PostgreSQL using:

```bash
psql -h localhost -U postgres -d omop_v5
```

----------

## Step 5: Troubleshooting

1.  **OMOPonFHIR Not Accessible:**
    
    -   Ensure the correct port (`30080`) is exposed in the `omoponfhir-service.yaml` file.
    -   Check pod and service statuses using `kubectl get pods` and `kubectl get services`.
2.  **Database Connection Issues:**
    
    -   Verify PostgreSQL is running using `nerdctl ps`.
    -   Ensure the correct `JDBC_URL`, `JDBC_USERNAME`, and `JDBC_PASSWORD` values are set in your OMOPonFHIR deployment.
3.  **Database Load Failures:**
    
    -   Check for errors during vocabulary loading, especially around missing `concept` IDs.
