<!-- Improved compatibility of back to top link: See: https://github.com/othneildrew/Best-README-Template/pull/73 -->
<a id="readme-top"></a>



<!-- PROJECT SHIELDS -->
<!--
*** I'm using markdown "reference style" links for readability.
*** Reference links are enclosed in brackets [ ] instead of parentheses ( ).
*** See the bottom of this document for the declaration of the reference variables
*** for contributors-url, forks-url, etc. This is an optional, concise syntax you may use.
*** https://www.markdownguide.org/basic-syntax/#reference-style-links
-->
[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]



<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow">
    <h3 align="center">Excel CSV Data Extractor</h3>
  </a>

  <p align="center">
    Upload a spreadsheet, land it in PostgreSQL as a table, and inspect it through a React UI —
    with an Airflow 3 stack available for scheduled reprocessing.
    <br />
    <a href="https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/tree/main/backend">Explore the source »</a>
    <br />
    <br />
    <a href="https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/tree/main/backend">Backend</a>
    &middot;
    <a href="https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/tree/main/frontend">Frontend</a>
    &middot;
    <a href="https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/issues">Report Bug</a>
    &middot;
    <a href="https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/issues/new?labels=enhancement">Request Feature</a>
  </p>
</div>



<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#architecture">Architecture</a></li>
        <li><a href="#how-a-file-becomes-a-table">How a File Becomes a Table</a></li>
        <li><a href="#api-reference">API Reference</a></li>
        <li><a href="#repository-layout">Repository Layout</a></li>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li>
      <a href="#usage">Usage</a>
      <ul>
        <li><a href="#through-the-web-ui">Through the Web UI</a></li>
        <li><a href="#through-the-api">Through the API</a></li>
        <li><a href="#through-the-terminal-cli">Through the Terminal CLI</a></li>
        <li><a href="#through-apache-airflow">Through Apache Airflow</a></li>
      </ul>
    </li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li>
      <a href="#contributing">Contributing</a>
      <ul>
        <li><a href="#top-contributors">Top contributors:</a></li>
      </ul>
    </li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>



<!-- ABOUT THE PROJECT -->
## About The Project

This repository is a two-part application for getting a flat file into a database and looking at it
without opening a SQL client.

* **`backend/`** is a FastAPI service that accepts uploads, writes them into PostgreSQL, and exposes
  read and search endpoints. It also carries an Apache Airflow 3 Compose stack and the DAGs that can
  do the same work on a schedule.
* **`frontend/`** is a Create React App in TypeScript. It talks to the FastAPI service over HTTP and
  renders upload, table-list, schema, statistics, search, and raw-data views.

Both halves are meant to run against the same PostgreSQL instance — the one the Compose stack starts
— so a file uploaded through the UI is immediately visible to the Airflow DAGs and vice versa.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### Architecture

```text
Browser  ──HTTP──▶  FastAPI  ──psycopg2 / SQLAlchemy──▶  PostgreSQL 17
:3000              :8000                                  :5432
                          ▲                                   ▲
                          │  ./data (shared volume)           │
                    ┌─────┴─────┐                             │
                    │ Airflow   │─────────────────────────────┘
                    │ :8080     │   CeleryExecutor + Redis broker
                    └───────────┘
```

| Service | Port | Notes |
| --- | --- | --- |
| React dev server | 3000 | CRA default; the only origin allowed by the API's CORS policy |
| FastAPI | 8000 | Started with `uvicorn`; not containerised |
| Airflow API server | 8080 | Part of `backend/docker-compose.yaml` |
| PostgreSQL 17 | 5432 | Metadata DB for Airflow *and* storage for uploaded data |
| Flower | 5555 | Behind the `flower` Compose profile |
| Redis | internal | Celery broker only, not published to the host |

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### How a File Becomes a Table

The file name is the table name. `Monthly Sales 2024-Q1.xlsx` becomes `data_monthly_sales_2024_q1`:
lowercased, with hyphens and spaces turned into underscores, prefixed with `data_`, and loaded with
`DataFrame.to_sql(..., if_exists='replace')`. Because the prefix is `data_`, every ingested table is
discoverable with a single query — `SELECT tablename FROM pg_tables WHERE tablename LIKE 'data_%'` —
which is exactly what `GET /tables` and the CLI use.

The `replace` strategy means re-uploading the same filename overwrites the previous table rather than
appending to it.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### API Reference

All endpoints are served by `backend/api.py`. Table names are normalised (lowercase, `(` and `)`
removed, spaces to underscores) before they reach SQL.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/tables` | List ingested tables (`data_%`) |
| `GET` | `/tables/{table}/columns` | Column names, for the search column picker |
| `GET` | `/tables/{table}/schema` | `column_name`, `data_type`, `is_nullable` |
| `GET` | `/tables/{table}/data?limit=10` | Row sample plus total row count |
| `GET` | `/tables/{table}/stats` | Per-column null counts and null %, plus mean/median/min/max for numeric columns |
| `POST` | `/search-table` | Search; `application/x-www-form-urlencoded` |
| `POST` | `/upload` | Multipart upload of `.xlsx`, `.xls`, `.csv` into `./data` |
| `POST` | `/process-file` | Load every file in `./data` into PostgreSQL |

`POST /search-table` accepts `table_name`, `search_term`, and optionally `column`, `case_sensitive`,
`regex`, `exact_match`, and `limit`. When `column` is a specific column the filter is pushed into SQL
(`LIKE`, `=`, or `~` depending on the flags); when it is `all`, the whole table is pulled into a
DataFrame and matched with `str.contains`. The response echoes the filters that were applied alongside
the matching rows.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### Repository Layout

```text
.
├── backend/
│   ├── api.py                     # FastAPI app — upload, process, browse, search
│   ├── cli.py                     # interactive terminal client for the same tables
│   ├── requirements.txt           # pandas, openpyxl, xlrd, psycopg2-binary
│   ├── docker-compose.yaml        # Airflow 3.0.2 + Postgres 17 + Redis 7.2
│   └── dags/
│       ├── excel_csv_processor.py # excel_csv_processor DAG — 5-minute schedule
│       └── other_dags.py          # dashboard_update_dag — hourly, placeholder task
└── frontend/
    ├── package.json               # React 19, react-scripts 5.0.1, TypeScript 4.9
    └── src/
        ├── App.tsx                # all views and the API calls
        ├── components/            # Header, Footer
        └── pages/                 # HelpPage, PrivacyPage, TermsPage (placeholder content)
```

Two things worth knowing before you build on this:

* `backend/requirements.txt` lists the data-processing dependencies only. `fastapi`, `uvicorn`, and
  `sqlalchemy` are imported by `api.py` but not pinned there, so the install step below adds them
  explicitly.
* The database credentials in `DB_CONFIG` (`api.py`) and in the `db_url` strings are hardcoded to
  `airflow` / `airflow`. There is no environment variable indirection yet.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### Built With

* [![React][React.js]][React-url]
* [![TypeScript][TypeScript]][TypeScript-url]
* [![FastAPI][FastAPI]][FastAPI-url]
* [![Apache Airflow][Apache Airflow]][Apache-Airflow-url]
* [![Python][Python]][Python-url]
* [![Pandas][Pandas]][Pandas-url]
* [![PostgreSQL][PostgreSQL]][PostgreSQL-url]
* [![Redis][Redis]][Redis-url]
* [![Celery][Celery]][Celery-url]
* [![Docker][Docker]][Docker-url]
* [![Uvicorn][Uvicorn]][Uvicorn-url]

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- GETTING STARTED -->
## Getting Started

You need three things running for the full experience: PostgreSQL, the FastAPI service, and the CRA
dev server. Airflow is optional but is what gives you the scheduled reprocessing path.

### Prerequisites

* Docker Engine with the Compose plugin
  ```sh
  # macOS
  brew install --cask docker
  # Debian / Ubuntu
  sudo apt-get install -y docker.io docker-compose-v2
  ```
* Python 3.9+
  ```sh
  python3 --version
  ```
* Node.js 14+ and npm
  ```sh
  node --version
  npm --version
  ```

### Installation

1. Clone the repo
   ```sh
   git clone https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow.git
   cd Excel_csv_data_extractor_Airflow
   ```

2. Start PostgreSQL (and Airflow, if you want it) from the Compose file
   ```sh
   cd backend
   docker compose up -d
   docker compose ps          # wait for postgres to report healthy
   cd ..
   ```
   `docker compose up -d` also mounts `./backend/dags`, `./backend/data`, `./backend/logs`,
   `./backend/config`, and `./backend/plugins` into the Airflow containers. The Airflow UI is then at
   <http://localhost:8080> with `airflow` / `airflow`.

   If you only need the database, bring just Postgres up:
   ```sh
   docker compose up -d postgres
   ```

3. Set up the API
   ```sh
   cd backend
   python3 -m venv venv
   source venv/bin/activate           # Windows: venv\Scripts\activate
   pip install -r requirements.txt
   pip install fastapi uvicorn sqlalchemy
   mkdir -p data
   uvicorn api:app --reload --host 0.0.0.0 --port 8000
   ```
   Run it **from `backend/`**. `api.py` resolves the upload directory as `./data` relative to the
   working directory, and the Compose file mounts `backend/data` — starting the API from anywhere
   else splits the two.

4. Set up the frontend in a second terminal
   ```sh
   cd frontend
   npm install
   npm start
   ```
   CRA serves on <http://localhost:3000>, which is the origin the API's CORS policy allows.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- USAGE EXAMPLES -->
## Usage

### Through the Web UI

1. Open <http://localhost:3000>.
2. **Upload** — choose an `.xlsx`, `.xls`, or `.csv` file. It is written to `backend/data/`.
3. Click **Process File** once the upload succeeds. The API reads every spreadsheet in
   `backend/data/`, loads it, and reports `filename -> table (N rows)` for each.
4. Click **Data Tables** and pick the table you just loaded. The detail view has four tabs:
   * **Schema** — column, Postgres type, nullable
   * **Statistics** — total rows and columns, per-column null counts, and min / max / mean / median
     for numeric columns
   * **Search** — search across all columns or one, with toggles for case sensitivity, exact match,
     and regex; matches are highlighted in the results
   * **View Data** — paged sample with an adjustable row limit

### Through the API

```sh
# List ingested tables
curl http://localhost:8000/tables

# Schema and a row sample
curl http://localhost:8000/tables/data_monthly_sales_2024_q1/schema
curl "http://localhost:8000/tables/data_monthly_sales_2024_q1/data?limit=25"

# Per-column statistics
curl http://localhost:8000/tables/data_monthly_sales_2024_q1/stats

# Upload, then load
curl -F "file=@./report.xlsx" http://localhost:8000/upload
curl -X POST http://localhost:8000/process-file

# Search: substring across every column
curl -X POST http://localhost:8000/search-table \
  -d "table_name=data_monthly_sales_2024_q1" \
  -d "search_term=contoso" \
  -d "column=all" \
  -d "regex=false" \
  -d "exact_match=false" \
  -d "case_sensitive=false" \
  -d "limit=100"

# Search: case-insensitive regex on one column
curl -X POST http://localhost:8000/search-table \
  -d "table_name=data_monthly_sales_2024_q1" \
  -d "search_term=^North.*(Ltd|Inc)$" \
  -d "column=customer_name" \
  -d "regex=true"
```

### Through the Terminal CLI

`backend/cli.py` talks to the same tables without a browser:

```sh
cd backend
source venv/bin/activate
python cli.py
```

```text
=== Excel Data CLI ===
1. List tables
2. Describe table
3. Query table
4. Exit
```

### Through Apache Airflow

`excel_csv_processor` is defined with `schedule=timedelta(minutes=5)`, `start_date=2024-01-01`, and
`catchup=False`. It has two `PythonOperator` tasks:

* `process_files` — globs `/opt/airflow/data/*.{xlsx,xls,csv}` and loads each one to `data_<name>`
  with `if_exists='replace'`, the same convention as `/process-file`.
* `search_table` — takes `table_name` and `search_term` from the DAG run config and writes matches to
  `search_results_<table_name>`.

It is created paused. Unpause it in the UI, or trigger it manually:

```sh
docker compose exec airflow-apiserver airflow dags unpause excel_csv_processor
docker compose exec airflow-apiserver airflow dags trigger excel_csv_processor

# search_table needs a run config
docker compose exec airflow-apiserver airflow dags trigger excel_csv_processor \
  --conf '{"table_name": "data_monthly_sales_2024_q1", "search_term": "contoso"}'

docker compose exec airflow-apiserver airflow dags list-import-errors
```

`other_dags.py` defines `dashboard_update_dag` on an hourly schedule. Its task is a stub — the
callable prints a message and returns a dict; the dashboard update itself is a comment.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- ROADMAP -->
## Roadmap

- [x] FastAPI upload, processing, browse, statistics, and search endpoints
- [x] React + TypeScript UI covering all of the above
- [x] Interactive terminal CLI over the same tables
- [x] Airflow 3 Compose stack with an `excel_csv_processor` DAG on a 5-minute schedule
- [ ] Move database credentials and the API base URL out of hardcoded literals into environment
      variables. `DB_CONFIG` in `api.py`, the `db_url` strings in `api.py` and
      `excel_csv_processor.py`, and the `http://localhost:8000` URLs in `App.tsx` are all fixed.
- [ ] Parameterise identifiers in the SQL built with f-strings. Search *values* are already bound as
      query parameters; table and column names are interpolated directly, and the
      `.lower().replace(...)` cleanup is the only guard. A proper identifier quoter would close this
      off.
- [ ] Give the API a Dockerfile so the FastAPI service can join the Compose network instead of
      needing `localhost:5432` to be reachable from the host.
- [ ] Replace the placeholder `HelpPage`, `PrivacyPage`, `TermsPage`, and `Footer` content. They
      currently ship with the stock enterprise-support scaffold, including an `expleo.com` support
      address that is not wired to anything. `CompliancePage` and `SecurityPage` are still in the
      tree but no longer imported by `App.tsx`.
- [ ] Fix the test suite. `src/App.test.tsx` is the unmodified Create React App test and still
      asserts on a "learn react" link that the app no longer renders, so `npm test` fails.
- [ ] Implement `dashboard_update_dag`'s task, currently a stub.
- [ ] Decide on a single ingestion path. The load logic is duplicated between `/process-file` in
      `api.py` and `process_files` in `excel_csv_processor.py`; having the API trigger the DAG through
      the Airflow REST API would keep one implementation.

See the [open issues](https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/issues) for a full list of proposed features (and known issues).

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

If you have a suggestion that would make this better, please fork the repo and create a pull request. You can also simply open an issue with the tag "enhancement".
Don't forget to give the project a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

If you touch the backend or the DAGs, start `backend/docker-compose.yaml` and confirm the DAGs still
import with `airflow dags list-import-errors`. If you touch the frontend, `npm run build` is the
fastest way to catch a type error.

### Top contributors:

<a href="https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow" alt="contrib.rocks image" />
</a>

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- LICENSE -->
## License

No license file is currently bundled with this repository. Until one is added, the default GitHub
terms apply and the repository should be treated as unlicensed — do not redistribute it without
asking.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- CONTACT -->
## Contact

Naoufal Elouahabi - [LinkedIn](https://www.linkedin.com/in/elouahabi-naoufal/) - elouahabi-naoufal1@gmail.com

Project Link: [https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow](https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow)

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- ACKNOWLEDGMENTS -->
## Acknowledgments

* [Apache Airflow](https://airflow.apache.org) — `backend/docker-compose.yaml` is derived from the
  official `docker-compose.yaml` shipped with Airflow 3.
* [FastAPI](https://fastapi.tiangolo.com)
* [Create React App](https://create-react-app.dev) — the `frontend/` scaffold and its scripts.
* [pandas](https://pandas.pydata.org) — all parsing, statistics, and `to_sql` loading.
* [Img Shields](https://shields.io)
* [Choose an Open Source License](https://choosealicense.com)

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow.svg?style=for-the-badge
[contributors-url]: https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow.svg?style=for-the-badge
[forks-url]: https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/network/members
[stars-shield]: https://img.shields.io/github/stars/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow.svg?style=for-the-badge
[stars-url]: https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/stargazers
[issues-shield]: https://img.shields.io/github/issues/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow.svg?style=for-the-badge
[issues-url]: https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow/issues
[license-shield]: https://img.shields.io/github/license/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow.svg?style=for-the-badge
[license-url]: https://github.com/Elouahabi-Naoufal/Excel_csv_data_extractor_Airflow
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com/in/elouahabi-naoufal/
[React.js]: https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB
[React-url]: https://react.dev/
[TypeScript]: https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white
[TypeScript-url]: https://www.typescriptlang.org/
[FastAPI]: https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white
[FastAPI-url]: https://fastapi.tiangolo.com/
[Apache Airflow]: https://img.shields.io/badge/Apache%20Airflow-017CEE?style=for-the-badge&logo=apacheairflow&logoColor=white
[Apache-Airflow-url]: https://airflow.apache.org/
[Python]: https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white
[Python-url]: https://www.python.org/
[Pandas]: https://img.shields.io/badge/Pandas-150458?style=for-the-badge&logo=pandas&logoColor=white
[Pandas-url]: https://pandas.pydata.org/
[PostgreSQL]: https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white
[PostgreSQL-url]: https://www.postgresql.org/
[Redis]: https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white
[Redis-url]: https://redis.io/
[Celery]: https://img.shields.io/badge/Celery-37814A?style=for-the-badge&logo=celery&logoColor=white
[Celery-url]: https://docs.celeryq.dev/
[Docker]: https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white
[Docker-url]: https://www.docker.com/
[Uvicorn]: https://img.shields.io/badge/Uvicorn-499848?style=for-the-badge&logo=uvicorn&logoColor=white
[Uvicorn-url]: https://www.uvicorn.org/
