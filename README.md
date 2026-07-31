# 🎬 IMDb — Azure Data Pipeline & Analytics

A production-style, cloud-based data pipeline that ingests, cleans, models, and visualizes **50M+ IMDb records** using **Azure Data Factory**, **Snowflake**, and **Power BI**, built on **Medallion Architecture**.

---

## Objective

Extract, clean, and model IMDb's public datasets to answer real business/analytics questions:

- Movie genre and rating trends
- Cast & crew profession breakdowns
- Region and language-based release patterns
- Series vs. non-series popularity
- Top-rated titles and character-level insights

---

## Tech Stack

| Category | Tools |
|---|---|
| Ingestion / ETL | Azure Data Factory (ADF), Alteryx |
| Data Warehouse | Snowflake |
| Data Modeling | ER Studio (Star Schema) |
| Profiling | ydata-profiling, Alteryx |
| BI / Reporting | Power BI |
| Architecture | Medallion (Bronze → Silver → Gold) |
| Version Control | Git & GitHub |

---

## Medallion Architecture

```mermaid
flowchart LR
    subgraph Source["IMDb Public Datasets (.tsv.gz)"]
        S1[name.basics – 14M+]
        S2[title.akas – 51M+]
        S3[title.crew – 11M+]
        S4[title.basics – 11M+]
        S5[title.episode – 8.8M+]
        S6[title.principals – 90M+]
        S7[title.ratings – 1.5M+]
    end

    subgraph Bronze[" Bronze - Raw Ingestion"]
        B1[(ADLS / Blob Storage
        Raw Landing Zone)]
    end

    subgraph Silver[" Silver - Cleaned & Staged"]
        SV1[Alteryx: Profiling & Cleansing]
        SV2[ADF Pipelines: Flatten, Stage,
        Standardize Schemas]
        SV3[(Snowflake Staging Tables)]
    end

    subgraph Gold[" Gold - Modeled for BI"]
        G1[(Star Schema:
        Dimensions + Fact + Bridges)]
    end

    R1[Power BI Dashboards]

    Source --> B1
    B1 --> SV1 --> SV2 --> SV3
    SV3 --> G1
    G1 --> R1
```

---

## Azure Data Factory Pipeline Flow

```mermaid
flowchart TD
    A[Raw IMDb Files in ADLS] --> B[Staging Pipelines]
    B --> B1[Title_Basics_Stg]
    B --> B2[Title_Ratings_Stg]
    B --> B3[Title_Crew_Stg]
    B --> B4[Title_Principal_Stg]
    B --> B5[Title_Akas_Stg]
    B --> B6[Title_Episode_Stg]
    B --> B7[Name_Basics_Stg]
    B --> B8[Region / Language Codes_Stg]

    B1 --> C1[Dataflow: Flatten & Clean]
    B3 --> C2[Dataflow: Flatten Title Crew]
    B5 --> C3[Dataflow: Flatten Region/Lang/Title]
    B8 --> C4[Dataflow: Flatten Language Codes]

    C1 --> D[Load Dimensions]
    C2 --> D
    C3 --> D
    C4 --> D

    D --> D1[DIM_TITLE]
    D --> D2[DIM_DATE]
    D --> D3[DIM_GENRE]
    D --> D4[DIM_LANGUAGE]
    D --> D5[DIM_REGION]
    D --> D6[DIM_PERSONNEL]

    D1 --> E[Load Bridge Tables]
    D6 --> E
    E --> E1[BRIDGE_PERSON_TITLE]
    E --> E2[BRIDGE_PERSON_PROFESSION]
    E --> E3[BRIDGE_REGION_LANG_TITLE]

    D1 --> F[Load Fact Table]
    D2 --> F
    D3 --> F
    E1 --> F
    F --> F1[(FACT_MOVIE_DETAILS)]

    F1 --> G[Power BI Reporting Layer]
```

---

##  Star Schema (Snowflake Data Model)

```mermaid
erDiagram
    FACT_MOVIE_DETAILS }o--|| DIM_TITLE : TITLE_ID
    FACT_MOVIE_DETAILS }o--|| DIM_DATE : DATE_SK
    FACT_MOVIE_DETAILS }o--|| DIM_GENRE : GENRE_SK

    DIM_TITLE ||--o{ BRIDGE_PERSON_TITLE : TITLE_ID
    DIM_PERSONNEL ||--o{ BRIDGE_PERSON_TITLE : PERSON_ID
    DIM_PERSONNEL ||--o{ BRIDGE_PERSON_PROFESSION : PERSON_ID
    DIM_TITLE ||--o{ BRIDGE_REGION_LANG_TITLE : TITLE_ID
    DIM_REGION ||--o{ BRIDGE_REGION_LANG_TITLE : REGION_SK
    DIM_LANGUAGE ||--o{ BRIDGE_REGION_LANG_TITLE : LANGUAGE_SK

    FACT_MOVIE_DETAILS {
        int MOVIE_DETAIL_SK PK
        int TITLE_ID FK
        int DATE_SK FK
        int GENRE_SK FK
        float AVG_RATING
        int NUM_VOTES
        string TITLE_TYPE
    }

    DIM_TITLE {
        int TITLE_ID PK
        string PRIMARY_TITLE
        string ORIGINAL_TITLE
        boolean IS_ADULT
        int RUNTIME_MINUTES
    }

    DIM_DATE {
        int DATE_SK PK
        int START_YEAR
        int END_YEAR
        int DECADE
    }

    DIM_GENRE {
        int GENRE_SK PK
        string GENRE_NAME
    }

    DIM_PERSONNEL {
        int PERSON_ID PK
        string PRIMARY_NAME
        int BIRTH_YEAR
        int DEATH_YEAR
    }

    DIM_REGION {
        int REGION_SK PK
        string REGION_CODE
        string REGION_NAME
    }

    DIM_LANGUAGE {
        int LANGUAGE_SK PK
        string LANGUAGE_CODE
        string LANGUAGE_NAME
    }

    BRIDGE_PERSON_TITLE {
        int PERSON_ID FK
        int TITLE_ID FK
        string ROLE_CATEGORY
    }

    BRIDGE_PERSON_PROFESSION {
        int PERSON_ID FK
        string PROFESSION
    }

    BRIDGE_REGION_LANG_TITLE {
        int TITLE_ID FK
        int REGION_SK FK
        int LANGUAGE_SK FK
    }
```

> **Why bridge tables?** IMDb relationships aren't strictly one-to-many a title can have many cast/crew members and release in many region/language combinations, and a person can have multiple professions. Bridge tables handle these many-to-many relationships cleanly within the star schema.

---

##  Repository Structure

```
IMDB---Azure-Data-Pipeline-and-Analytics/
│
├── IMDB_AMS_DADABI-main/           # Azure Data Factory export
│   ├── factory/                   # ADF factory definition
│   ├── pipeline/                  # Staging & load pipelines (per entity)
│   ├── dataflow/                  # Mapping data flows (flatten, bridge, dim loads)
│   ├── dataset/                   # Source/sink dataset definitions
│   └── linkedService/             # Connections (ADLS, Blob, Snowflake, Key Vault)
│
├── CLEANING/                       # Alteryx cleaning workflows (.yxmd) + cleaning report
├── PROFILING/                      # ydata-profiling reports (HTML/PDF) per entity
├── PowerBI_Dashboards/              # Exported dashboard PDFs
├── images/                         # Dashboard screenshots
├── Mapping_Midterm.xlsx            # Source-to-target mapping document
└── README.md
```

---

##  Pipeline Steps

1. **Ingest** — Raw IMDb `.tsv.gz` files land in Azure Data Lake Storage (Bronze layer).
2. **Profile** — Run `ydata-profiling` and Alteryx profiling on each entity (titles, names, ratings, crew, episodes, akas, region/language codes) to catch nulls, duplicates, and type issues.
3. **Clean (Alteryx)** — Dedicated `.yxmd` workflows clean and standardize each dataset before staging.
4. **Stage & Transform (ADF)** — Per-entity staging pipelines load data into Snowflake staging tables; Mapping Data Flows flatten nested/array fields (e.g. multiple genres, languages, professions per row) and standardize schemas per the source-to-target mapping document.
5. **Model (Gold layer)** — Load dimension tables, bridge tables (for many-to-many relationships), and the central `FACT_MOVIE_DETAILS` table into Snowflake using a star schema.
6. **Report** — Power BI connects to the Gold layer to power interactive dashboards with slicers/filters for business users.

---

##  Sample Dashboards

Dashboards built in **Power BI**, covering genre/rating trends, top-rated titles, cast & crew profession breakdowns, and region/language release patterns.

![DASH1](images/Dash1.png)
![DASH2](images/Dash2.png)
![DASH3](images/Dash3.png)
![DASH4](images/Dash4.png)
![DASH5](images/Dash5.png)

---

##  Future Improvements

- Move Snowflake credentials to Azure Key Vault–backed linked services everywhere (partially in place already)
- Add pipeline-level monitoring/alerting in ADF
- Automate incremental loads instead of full staging reloads
- Add CI/CD for ADF pipeline deployment across environments
