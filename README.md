# Submission of Homework #2 for Data Engineering Zoomcamp 2026 cohort

## Question 1

For this task, I have created a simple flow that downloads the dataset directly from Github, decompresses it using Kestra built-in component, and logs the size of the files using Kestra's `io.kestra.plugin.core.storage.Size` plugin.

```yaml
id: task01
namespace: homework.solutions
tasks:
  - id: download_gz
    type: io.kestra.plugin.core.http.Download
    uri: https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2020-12.csv.gz
  - id: decompress
    type: io.kestra.plugin.compress.FileDecompress
    compression: GZIP
    from: "{{ outputs.download_gz.uri }}"
  - id: get_sizeof
    type: io.kestra.plugin.core.storage.Size
    uri: "{{ outputs.decompress.uri }}"

```

<img width="2474" height="1714" alt="image" src="https://github.com/user-attachments/assets/7395939f-ccc0-4f08-8e34-a459b151f910" />

### Answer: 134481400 bytes ~ 134.5 MiB

---

## Question 2

Not sure how to approach this, maybe I am missing something. But...

```yaml
id: task02
namespace: homework.solutions

inputs:
  - id: taxi_type
    type: STRING
    defaults: green
  - id: date_year
    type: STRING
    defaults: "2020"
  - id: date_month
    type: STRING
    defaults: "04"

tasks:
  - id: download_gz
    type: io.kestra.plugin.core.http.Download
    uri: https://github.com/DataTalksClub/nyc-tlc-data/releases/download/{{inputs.taxi_type}}/{{inputs.taxi_type}}_tripdata_{{inputs.date_year}}-{{inputs.date_month}}.csv.gz
  - id: decompress
    type: io.kestra.plugin.compress.FileDecompress
    compression: GZIP
    from: "{{ outputs.download_gz.uri }}"
  - id: show_filename
    type: io.kestra.plugin.core.debug.Return
    format: "{{ inputs.taxi_type }}_tripdata_{{ inputs.date_year }}-{{ inputs.date_month }}.csv"

```

<img width="2474" height="1714" alt="image" src="https://github.com/user-attachments/assets/634a8201-7852-44f7-9574-81dd812350a4" />

<img width="2474" height="1714" alt="image" src="https://github.com/user-attachments/assets/5b6b69ab-cf5e-4fbd-8754-2d38e01c674f" />


### Answer: green_tripdata_2020-04.csv

---

## Question 3

For this task I have download all of the files into local directory for ease. However, it turned out to be opposite. This is the reason I am submitting late. I had to modify my `docker-compose.yml` in order to fix Kestra's access to the local file-system and then couldn't really make the looping through these files work properly. Therefore, I just gave up and directly downloaded and loaded the files from Github to Postgres.

Firstly, I created a subflow that (1) downloads, (2) decompresses and (3) upload to postgres table:
```yaml
id: load_taxi_month
namespace: homework.solutions

inputs:
  - id: taxi_type
    type: STRING
    required: true
  - id: date_year
    type: STRING
    required: true
  - id: month
    type: STRING
    required: true
  - id: postgres_database
    type: STRING
    required: true
  - id: postgres_username
    type: STRING
    required: true
  - id: postgres_password
    type: STRING
    required: true
  - id: postgres_address
    type: STRING
    required: true

tasks:
  - id: download_data
    type: io.kestra.plugin.fs.http.Download
    uri: "https://github.com/DataTalksClub/nyc-tlc-data/releases/download/{{inputs.taxi_type}}/{{inputs.taxi_type}}_tripdata_{{inputs.date_year}}-{{inputs.month}}.csv.gz"
  
  - id: decompress_data
    type: io.kestra.plugin.compress.FileDecompress
    compression: GZIP
    from: "{{ outputs.download_data.uri }}"
  
  - id: load_data
    type: io.kestra.plugin.jdbc.postgresql.CopyIn
    url: jdbc:postgresql://{{inputs.postgres_address}}:5432/{{inputs.postgres_database}}
    username: "{{inputs.postgres_username}}"
    password: "{{inputs.postgres_password}}"
    table: "{{inputs.taxi_type}}_taxi_data"
    from: "{{ outputs.decompress_data.uri }}"
    format: CSV
    header: true
```

Then, my main flow would infer this subflow and manually load all 12 month within 2020 of yellow taxi dataset:

```yaml
id: task03
namespace: homework.solutions

inputs:
  - id: taxi_type
    type: STRING
    defaults: yellow
  - id: date_year
    type: STRING
    defaults: "2020"
  - id: postgres_database
    type: STRING
    defaults: <REDACTED>
  - id: postgres_username
    type: STRING
    defaults: <REDACTED>
  - id: postgres_password
    type: STRING
    defaults: <REDACTED>
  - id: postgres_address
    type: STRING
    defaults: <REDACTED>

tasks:
  # Create table
  - id: create_table
    type: io.kestra.plugin.jdbc.postgresql.Query
    url: jdbc:postgresql://{{inputs.postgres_address}}:5432/{{inputs.postgres_database}}
    username: "{{inputs.postgres_username}}"
    password: "{{inputs.postgres_password}}"
    sql: |
      CREATE TABLE IF NOT EXISTS {{inputs.taxi_type}}_taxi_data (
        VendorID INTEGER,
        lpep_pickup_datetime TIMESTAMP,
        lpep_dropoff_datetime TIMESTAMP,
        store_and_fwd_flag VARCHAR(1),
        RatecodeID FLOAT,
        PULocationID INTEGER,
        DOLocationID INTEGER,
        passenger_count INTEGER,
        trip_distance DECIMAL(10,2),
        fare_amount DECIMAL(10,2),
        extra DECIMAL(10,2),
        mta_tax DECIMAL(10,2),
        tip_amount DECIMAL(10,2),
        tolls_amount DECIMAL(10,2),
        ehail_fee DECIMAL(10,2),
        improvement_surcharge DECIMAL(10,2),
        total_amount DECIMAL(10,2),
        payment_type INTEGER,
        trip_type INTEGER,
        congestion_surcharge DECIMAL(10,2)
      );

  # Truncate table
  - id: truncate_table
    type: io.kestra.plugin.jdbc.postgresql.Query
    url: jdbc:postgresql://{{inputs.postgres_address}}:5432/{{inputs.postgres_database}}
    username: "{{inputs.postgres_username}}"
    password: "{{inputs.postgres_password}}"
    sql: "TRUNCATE TABLE {{inputs.taxi_type}}_taxi_data;"

  # January
  - id: load_january
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "01"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"
    # February
  - id: load_february
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "02"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # March
  - id: load_march
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "03"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # April
  - id: load_april
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "04"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # May
  - id: load_may
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "05"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # June
  - id: load_june
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "06"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # July
  - id: load_july
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "07"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # August
  - id: load_august
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "08"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # September
  - id: load_september
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "09"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # October
  - id: load_october
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "10"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # November
  - id: load_november
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "11"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"

  # December
  - id: load_december
    type: io.kestra.plugin.core.flow.Subflow
    namespace: homework.solutions
    flowId: load_taxi_month
    inputs:
      taxi_type: "{{inputs.taxi_type}}"
      date_year: "{{inputs.date_year}}"
      month: "12"
      postgres_database: "{{inputs.postgres_database}}"
      postgres_username: "{{inputs.postgres_username}}"
      postgres_password: "{{inputs.postgres_password}}"
      postgres_address: "{{inputs.postgres_address}}"
  
    # Count total loaded rows
  - id: count_loaded
    type: io.kestra.plugin.jdbc.postgresql.Query
    url: jdbc:postgresql://{{inputs.postgres_address}}:5432/{{inputs.postgres_database}}
    username: "{{inputs.postgres_username}}"
    password: "{{inputs.postgres_password}}"
    sql: "SELECT COUNT(*) as total_rows FROM {{inputs.taxi_type}}_taxi_data;"
    store: true
    fetch: true
```

So, finally here are the screenshots showing my results:

<img width="2400" height="1732" alt="Screenshot From 2026-02-04 18-25-58" src="https://github.com/user-attachments/assets/f7788f4b-d802-46b0-a8ae-83f43305488e" />

<img width="2392" height="1722" alt="image" src="https://github.com/user-attachments/assets/9ef7d3d5-0a33-4e86-96e2-dd6115c36508" />

<img width="2392" height="1722" alt="image" src="https://github.com/user-attachments/assets/3f139a4b-ccc7-42b3-b25c-5dbab16eb0c6" />

<img width="1778" height="1546" alt="image" src="https://github.com/user-attachments/assets/fa27316a-5302-4c09-9a28-2794fd98609a" />


### Answer: 24,648,499

## Question 4
Similarly, but changing the parameters:

<img width="2400" height="1732" alt="Screenshot From 2026-02-04 18-17-45" src="https://github.com/user-attachments/assets/059505a1-313b-472a-82d3-e0cd94116ffb" />


<img width="2400" height="1732" alt="image" src="https://github.com/user-attachments/assets/9c8c115b-9182-4618-bd2b-7a2a81d51818" />


<img width="2400" height="1732" alt="image" src="https://github.com/user-attachments/assets/a29071c8-bd99-4f7d-9a79-8ee8308e4fb1" />


<img width="1892" height="1416" alt="image" src="https://github.com/user-attachments/assets/39c9e03f-b3fa-4a95-9175-6e96f9793054" />

### Answer: 1,734,051

## Question 5

For this task, I simply just limited the parameters to 2021, yellow and March.

<img width="1836" height="1554" alt="image" src="https://github.com/user-attachments/assets/1d4489bc-d985-45b0-b382-4dc80369542b" />

### Answer: 1,925,152

## Question 6

```yaml
triggers:
  - id: schedule
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 9 * * *"
    timezone: America/New_York
```

### Answer: Add a location property set to New_York in the Schedule trigger configuration

