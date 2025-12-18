### Apache Hive to Neo4j via Apache Hop

This project demonstrates how to move data from **Apache Hive** into **Neo4j** using **Apache Hop**.  
Everything is containerised with `docker-compose` so you can bring up a small Hadoop + Hive + Neo4j + Hop stack locally.

---

### 1. Prerequisites

- **Docker** and **Docker Compose** installed and running
- At least **4 GB** of free RAM (more is better for Hadoop + Neo4j)
- `git` (if you cloned this repo)

All commands below assume your working directory is:

```bash
cd "/Users/vishavbangotra/Documents/GitHub/Apache to Neo4J/Hive2Neo4jviaHop"
```

---

### 2. Project Layout (high level)

- **`docker-compose.yml`**: brings up all containers:
  - `namenode`, `datanode` – HDFS
  - `hive-metastore`, `hive-metastore-postgresql`, `hive-server` – Hive
  - `neo4j` – Neo4j database
  - `hop` – Apache Hop web UI
- **`data/`**: CSV and HQL files used to populate Hive (`movies.csv`, `persons.csv`, `roles.csv`, `data.hql`).
- **`hdfs/`**: HDFS data for namenode and datanode.
- **`hive/`**: Hive metastore/PostgreSQL data.
- **`neo4j/`**: Neo4j configuration and data directories.
- **`hop/files/`**: Apache Hop project files and a convenient shared directory between host and Hop container.

---

### 3. Starting the stack

From the `Hive2Neo4jviaHop` directory:

```bash
docker compose up -d
```

This will start:

- Hadoop NameNode (50070)
- DataNode (50075)
- Hive Metastore + PostgreSQL (9083)
- HiveServer2 (10000)
- Neo4j (7474 HTTP, 7687 Bolt)
- Hop Web (8080)

You can check that everything is running:

```bash
docker ps
```

You should see containers named `namenode`, `datanode`, `hive-metastore`, `hive-server`, `neo4j`, and `hop`.

To stop the stack:

```bash
docker compose down
```

---

### 4. Accessing the UIs

- **Hop Web**: open `http://localhost:8080` in your browser.
- **Neo4j Browser**: open `http://localhost:7474`.  
  Default credentials are usually `neo4j` / `neo4j` on first run (you will be prompted to change the password).
- **Hadoop NameNode UI**: `http://localhost:50070`

---

### 5. Loading data into Hive

The `data/` directory contains:

- `movies.csv`
- `persons.csv`
- `roles.csv`
- `data.hql` – Hive script to create tables and load the CSV data.

To run the Hive script inside the `hive-server` container:

```bash
docker exec -it hive-server /opt/hive/bin/beeline \
  -u "jdbc:hive2://localhost:10000/default" \
  -n hive \
  -p hive \
  -f /data/data.hql
```

> **Note**: The exact username/password may vary by image; adjust if you have customised them in `hadoop-hive.env`.

After this, you should have Hive tables populated with the sample movie/person/role data.

---

### 6. Making the Hive JDBC driver available to Hop

Hop runs in its own container and does not include the **Hive standalone JDBC driver JAR** by default.  
You have already seen permission issues when trying to copy into `/opt/hop`. The recommended workaround is to use the existing `./hop/files` volume.

- On the **host**, place your `hive-jdbc-standalone.jar` (or equivalent) in the `hop/files` directory:

```bash
cp /path/to/hive-jdbc-standalone.jar "./hop/files/"
```

- Inside the **`hop`** container this JAR will be available as:

```text
/files/hive-jdbc-standalone.jar
```

In the Hop Web UI, when configuring the Hive JDBC driver, point to this path instead of `/opt/hop/lib/...`.  
This avoids having to run the container as root or modify the image.

---

### 7. Using Apache Hop to move data from Hive to Neo4j

1. **Open Hop Web** at `http://localhost:8080`.
2. In the Hop project, open the pipeline(s) stored under the mounted `./hop/files` directory  
   (they are in `hop/files/` in the repo; inside the container they appear under `/files`).
3. Configure / verify:
   - **Hive connection** using the JDBC URL `jdbc:hive2://hive-server:10000/default` and the driver JAR at `/files/hive-jdbc-standalone.jar`.
   - **Neo4j connection** using the Bolt URL `bolt://neo4j:7687` and the Neo4j credentials you set in the browser.
4. Run the Hop pipeline that:
   - Reads from the Hive tables (`movies`, `persons`, `roles`).
   - Writes nodes and relationships into Neo4j.

Once the pipeline finishes successfully, you can inspect the graph in **Neo4j Browser**.
![Alt text](https://drive.google.com/file/d/1G0D7F6VT9co9V-HHRM3JeKI4shtNLAU6/view?usp=sharing)

---

### 8. Troubleshooting notes

- **Permission denied when writing under `/opt` in the `hop` container**  
  The container runs as a non-root user. Instead of changing that, prefer using the existing `/files` mount as described above.

- **JAR not found inside container**  
  Double-check that the JAR is in `./hop/files` on the host and that the container is running:
  ```bash
  ls ./hop/files
  docker ps | grep hop
  docker exec -it hop ls /files
  ```

- **Services not starting / connection refused**  
  Use `docker compose logs <service-name>` (e.g. `hive-server`, `neo4j`, `hop`) to see detailed logs.

---

### 9. Clean up

To stop and remove the containers (data volumes are persisted in the local folders such as `hdfs/`, `hive/`, `neo4j/`):

```bash
docker compose down
```

If you also want to remove all persisted data, delete the data directories:

```bash
rm -rf hdfs hive neo4j hop/files/*
```

> **Warning**: This is destructive – only run it if you are sure you want to reset the environment.


