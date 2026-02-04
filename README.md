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

For this task I have download all of the files into local directory for ease. However, it turned out to be opposite. This is the reason I am submitting late.

So, firstly I had to modify my `docker-compose.yml` in order to fix Kestra's access to the local file-system by adding entries like:

```yaml

...

volumes:
- /tmp/kestra-data/data:/taxi-data:ro

...

kestra:
  plugins:
    configurations:
      - type: io.kestra.plugin.fs.local.List # Or io.kestra.plugin.fs.local.Downloads, io.kestra.plugin.fs.local.Download
        values:
          allowed-paths:
            - /taxi-data


```



