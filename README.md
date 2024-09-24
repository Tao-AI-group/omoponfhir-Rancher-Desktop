# OMOPonFHIR-main-v54-r4 Installation and Configuration

This guide provides detailed steps to install and configure OMOPonFHIR-main-v54-r4 using **Rancher Desktop** (configured as containerd), with a PostgreSQL database for OMOPv5.4 hosted outside of Kubernetes.

  
## Step 1: Rancher Desktop Installation and Configuration

Rancher Desktop is an open-source application that provides all the essentials to setup a local Kubernetes environment for development. Rancher Desktop can be downloaded from https://rancherdesktop.io/

To follow the next instructions, I am using this version: https://github.com/rancher-sandbox/rancher-desktop/releases/download/v1.15.1/Rancher.Desktop-1.15.1.aarch64.dmg

After installation and running Rancher Desktop, make sure that "Enable Kubernetes" is selected, Container Engine: containered and Configure PATH: Automatic are marked.

----------

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
nerdctl run --restart=always --name omop-postgres -d -p 5432:5432 -e POSTGRES_PASSWORD=your_password postgres:latest
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



#### 2.4.1. Install psql Locally Using Homebrew (Recommended for Mac):

If you want to run psql from your Mac’s terminal (outside the container) to interact with the PostgreSQL instance running in the container, you need to install the PostgreSQL client:

```bash
brew install postgresql
```
If brew is not installed you can install it using

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### 2.4.2. Run the DDL scripts to set up the OMOP database schema:

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

```
psql -h localhost -U postgres -d omop_v5 -c "\copy DRUG_STRENGTH FROM 'path_to_DRUG_STRENGTH.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
```
```
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT FROM 'path_to_CONCEPT.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
```
```
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT_RELATIONSHIP FROM 'path_to_CONCEPT_RELATIONSHIP.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
```
```
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT_ANCESTOR FROM 'path_to_CONCEPT_ANCESTOR.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
```
```
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT_SYNONYM FROM 'path_to_CONCEPT_SYNONYM.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
```
```
psql -h localhost -U postgres -d omop_v5 -c "\copy VOCABULARY FROM 'path_to_VOCABULARY.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
```
```
psql -h localhost -U postgres -d omop_v5 -c "\copy RELATIONSHIP FROM 'path_to_RELATIONSHIP.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
```
```
psql -h localhost -U postgres -d omop_v5 -c "\copy CONCEPT_CLASS FROM 'path_to_CONCEPT_CLASS.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;"
```
```
psql -h localhost -U postgres -d omop_v5 -c "\copy DOMAIN FROM 'path_to_DOMAIN.csv' WITH DELIMITER E'\t' CSV HEADER QUOTE E'\b' ;" 
```
Make sure all required vocabulary files are loaded.

### 2.6. Access and Modify postgresql.conf

#### 2.6.1. Identify Your PostgreSQL Container
```
nerdctl ps
```
Example output
```
CONTAINER ID    IMAGE                                COMMAND                   CREATED        STATUS    PORTS                     NAMES
fd07a56dfb9d    docker.io/library/postgres:latest    "docker-entrypoint.s…"    4 hours ago    Up        0.0.0.0:5432->5432/tcp    omop-postgres
```
#### 2.6.2. Access the PostgreSQL Container Shell
```
nerdctl exec -it <container_id_or_name> bash
```
#### 2.6.3. Locate and edit the postgresql.conf File
```
echo $PGDATA
```

Install an editor as follows:
```
apt-get update && apt-get install -y vim
```
```
vi postgresql.conf
```
Make sure listen_addresses = '*' is uncommented
#### 2.6.4. Edit the pg_hba.conf File
Add a Host Entry
```
host    all             all             0.0.0.0/0               md5
```
**Note: IP (0.0.0.0/0) is not secure. In a production environment, you should replace this with the specific IP address range of your Kubernetes cluster.**

#### 2.6.5. Restart PostgreSQL Service Inside the Container or Restart the Container

```
nerdctl restart omop-postgres
```

----------

## Step 3: Deploy OMOPonFHIR in Kubernetes

### 3.1. Clone OMOPonFHIR Repository

Clone the `omoponfhir-main-v54-r4` repository:

```bash
git clone --recurse-submodules https://github.com/mcc-ad/UTH-omoponfhir-main-v54-r4.git
cd UTH-omoponfhir-main-v54-r4
```

### 3.2. Update pom.xml:

A.	Go inside the omoponfhir-r4-server sub-folder and edit the POM.xml file

B.	Make sure to add the following dependencies to it:
```
		<dependency>
			<groupId>org.apache.logging.log4j</groupId>
			<artifactId>log4j-api</artifactId>
			<version>2.8.2</version>
		</dependency>
		<dependency>
			<groupId>org.apache.logging.log4j</groupId>
			<artifactId>log4j-core</artifactId>
			<version>2.8.2</version>
		</dependency>
```
### 3.3. Build OMOPonFHIR Docker Image

Since Rancher Desktop uses containerd, you’ll build the OMOPonFHIR image using `nerdctl`:

If you will run the OMOPonFHIR application inside a container, excecute:

```bash
nerdctl build -t omoponfhir:latest .
```
**Note: If you got an error similar to "error: failed to solve: process "/bin/sh -c mvn clean install" did not complete successfully: exit code: 137
FATA[0249] no image was built  ",  increase the virtual machine memory size in Rancher Desktop from 2 GB to 4 GB**

If you will run the OMOPonFHIR application inside a pod managed by kubernetes, execute:

```
nerdctl -n k8s.io build -t omoponfhir:latest .
```

**Note: In the following steps I will assume that the OMOPonFHIR application will run as a pod managed by kubernetes**

### 3.4. Configure Kubernetes Deployment

#### 3.4.1. Create the YAML files for Kubernetes. 

This defines the deployment and service configuration for OMOPonFHIR. 

Create a file called omoponfhir-deployment.yaml. Edit the file to ensure that the environment variables (specifically JDBC connection details) are set properly to connect to your PostgreSQL database:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: omoponfhir
spec:
  replicas: 1
  selector:
    matchLabels:
      app: omoponfhir
  template:
    metadata:
      labels:
        app: omoponfhir
    spec:
      containers:
      - name: omoponfhir
        image: omoponfhir:latest
        imagePullPolicy: Never  # Set to Never or IfNotPresent to avoid always pulling from a registry
        ports:
        - containerPort: 8080
        env:
        - name: JDBC_URL
          value: "jdbc:postgresql://ipaddress:5432/omop_v5"
        - name: JDBC_USERNAME
          value: "postgres"
        - name: JDBC_PASSWORD
          value: "your_password"
        - name: JDBC_DRIVER
          value: "org.postgresql.Driver"
        - name: JDBC_DATASOURCENAME
          value: "org.postgresql.ds.PGSimpleDataSource"
        - name: SERVERBASE_URL
          value: "http://localhost:8080/fhir"
        - name: JDBC_POOLSIZE
          value: "10"  # Set the desired pool size
        - name: JDBC_DATA_SCHEMA
          value: "public"
        - name: JDBC_VOCABS_SCHEMA
          value: "public"
        - name: AUTH_BEARER
          value: "12345"
        - name: AUTH_BASIC
          value: "client:secret"
---
apiVersion: v1
kind: Service
metadata:
  name: omoponfhir-service
spec:
  selector:
    app: omoponfhir
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30080
  type: NodePort
```

#### 3.5.1. Deploy OMOPonFHIR

Deploy the OMOPonFHIR application in Kubernetes:

```bash
kubectl apply -f omoponfhir-deployment.yaml
```

You may run the following to get the pod name and make sure its status is "Running"
```
kubectl get pods
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
