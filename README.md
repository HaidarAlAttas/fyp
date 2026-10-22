# MoveKL

**Weather-aware public transport trip planning for Kuala Lumpur, built on an open data engineering pipeline.**

MoveKL shows commuters in the Klang Valley how to get from A to B by bus, LRT, MRT, Monorail or KTM, and tells them whether rain or an official weather warning will affect *that* journey at *that* time. Behind the app is an automated pipeline that collects, cleans and validates Malaysian open transport data and weather data, and builds a history that can be used to measure and predict bus delays.

> 🚧 **Status: early development.** Final Year Project (FYP) at Universiti Tenaga Nasional (UNITEN), Malaysia, 2026–2027. The design is complete and the repository is being built up component by component — see [Roadmap](#roadmap).

---

## Why

Existing apps (Google Maps, Moovit, MyRapid PULSE) show routes and live buses, but the weather they show, if any, is for the map area right now — not for the commuter's specific route and travel time. Kuala Lumpur's heavy afternoon storms and flash floods regularly disrupt commuting, and there is no journey-specific view of that risk.

MoveKL links three things together:

1. **Routes** across all Klang Valley rail and Rapid KL bus services
2. **Weather along the journey** — hourly rain forecast for the time you'll actually be travelling, plus official MET Malaysia warnings
3. **Bus reliability** — live delay status, calculated from real-time bus positions against the timetable

## Features

**Mobile app (Android)**
- Map of your location, nearby stops and live bus positions (OpenStreetMap)
- Journey search, e.g. *UNITEN → KLCC*, with route options drawn on the map
- Travel time, live bus delay status, rain expected during the trip, active MET warnings
- Guest-first: plan trips without an account
- Optional account (email or Google Sign-In) to save places (Home, Work, custom) and see trip history

**Data pipeline (the core of the project)**
- Continuous collector for GTFS-Realtime bus positions (every 60 s)
- Scheduled ingestion of GTFS static timetables, hourly weather forecasts and MET warnings
- Cleaning, validation and data quality reporting
- Bus delay calculation by projecting live positions onto GTFS route shapes
- Raw data kept unchanged so everything downstream can be rebuilt

**Optional: machine learning**
- Predict journey delay / route reliability from weather and historical delays, evaluated against OpenTripPlanner's scheduled times

## Architecture

```mermaid
flowchart LR
    subgraph Sources
        GTFS[data.gov.my<br/>GTFS static]
        RT[data.gov.my<br/>GTFS-Realtime]
        OM[Open-Meteo<br/>forecast]
        MET[MET Malaysia<br/>warnings]
    end

    subgraph Server["Server (Docker)"]
        COL[Collector<br/>every 60 s]
        RAW[(Raw files)]
        AF[Apache Airflow<br/>batch jobs]
        PG[(PostgreSQL<br/>+ PostGIS)]
        OTP[OpenTripPlanner]
        API[FastAPI backend]
    end

    subgraph Client
        APP[Flutter app<br/>Android]
        FB[Firebase<br/>Authentication]
        OSM[OpenStreetMap<br/>tiles + Photon]
    end

    RT --> COL --> RAW
    RAW --> AF
    GTFS --> AF
    OM --> AF
    MET --> AF
    AF --> PG
    GTFS --> OTP
    PG --> API
    OTP --> API
    API --> APP
    APP <--> FB
    APP --> OSM
```

| Job | Frequency |
|---|---|
| Collect GTFS-Realtime bus positions → raw files | every 60 s (continuous collector, not Airflow) |
| Process raw positions: decode, clean, validate, load | every 15 min |
| MET Malaysia warnings (deduplicated) | every 15 min |
| Open-Meteo hourly forecast | hourly |
| Bus delay calculation (positions vs timetable) | hourly |
| GTFS static timetables → PostgreSQL | daily |
| Data quality report | daily |
| Rebuild OpenTripPlanner graph | weekly / when GTFS changes |

The mobile app never calls data.gov.my directly — everything goes through the backend, which keeps within the data.gov.my rate limit (4 requests/minute per API).

## Tech stack

| Layer | Tools |
|---|---|
| Data pipeline | Python, Apache Airflow, PostgreSQL, PostGIS |
| Routing | OpenTripPlanner |
| Backend | FastAPI, Firebase Admin (token verification) |
| Mobile app | Flutter (Android), flutter_bloc (Cubit), flutter_map |
| Auth | Firebase Authentication (email/password, Google) |
| Maps & search | OpenStreetMap tiles, Photon, PostgreSQL `pg_trgm` for stop search |
| Machine learning (optional) | scikit-learn, XGBoost / LightGBM |
| Deployment | Docker, Docker Compose |

## Data sources

| Source | Data |
|---|---|
| [data.gov.my](https://developer.data.gov.my/) GTFS static | Prasarana (LRT, MRT, Monorail, Rapid KL bus) and KTMB timetables, stops and route shapes |
| [data.gov.my](https://developer.data.gov.my/) GTFS-Realtime | Vehicle positions for Rapid KL bus, MRT feeder bus and KTMB (no real-time feed for LRT/MRT/Monorail) |
| [Open-Meteo](https://open-meteo.com/) | Hourly weather forecast and historical archive |
| MET Malaysia via data.gov.my | Official weather warnings |
| [OpenStreetMap](https://www.openstreetmap.org/) | Street network for walking routes, map tiles, place search |

## Repository structure

```
movekl/
├── app/            # Flutter mobile app (Android)
├── pipeline/
│   ├── collector/  # Continuous GTFS-Realtime collector
│   ├── dags/       # Airflow DAGs
│   └── sql/        # Database schema and migrations
├── backend/        # FastAPI service
├── otp/            # OpenTripPlanner config and graph build scripts
├── ml/             # Optional delay / reliability models
├── docs/           # Architecture, data dictionary, s390x notes
└── docker-compose.yml
```

*Folders are added as each component is built.*

## Getting started

### Mobile app

Requirements: Flutter SDK, Android Studio or an Android device.

```bash
git clone https://github.com/<your-username>/movekl.git
cd movekl/app
flutter pub get
flutter run
```

### Server stack

*Coming soon* — the pipeline, backend and OpenTripPlanner will run with:

```bash
cp .env.example .env      # add your API keys / Firebase credentials
docker compose up -d
```

## Running on IBM LinuxONE (s390x)

MoveKL is being developed and tested on IBM LinuxONE (s390x architecture). Not every component ships s390x container images, so this project documents what works and how:

- Python, PostgreSQL and Java (Eclipse Temurin) — official s390x builds
- Apache Airflow — installed with pip on an s390x Python base image
- PostGIS — installed from distribution packages
- OpenTripPlanner — shaded JAR on an s390x JDK
- ML libraries — tested for s390x availability; built from source where needed

Notes will be published in [`docs/s390x.md`](docs/s390x.md).

## Roadmap

- [x] Requirements, use cases and system architecture
- [x] UI design (Figma)
- [x] Flutter project set up
- [ ] GTFS-Realtime collector running 24/7
- [ ] PostgreSQL schema and GTFS static ingestion
- [ ] Airflow DAGs: weather, warnings, cleaning, validation, data quality report
- [ ] Bus delay calculation
- [ ] OpenTripPlanner deployment for the Klang Valley
- [ ] FastAPI backend (journeys, weather, warnings, saved places)
- [ ] Flutter app: map, journey search, route details, weather, accounts
- [ ] s390x deployment notes
- [ ] (Optional) ML delay / reliability prediction

## Out of scope

Ticketing and payments · real-time LRT/MRT/Monorail delays · turn-by-turn navigation · iOS release.

## Privacy

Accounts are optional. For registered users MoveKL stores only email, name and saved places, and users can delete their account at any time, in line with Malaysia's Personal Data Protection Act 2010 (PDPA).

## Attribution

- Map data © [OpenStreetMap contributors](https://www.openstreetmap.org/copyright)
- Transport data: [data.gov.my](https://data.gov.my/), Prasarana Malaysia Berhad and KTMB
- Weather data by [Open-Meteo.com](https://open-meteo.com/)
- Weather warnings: Malaysian Meteorological Department (MET Malaysia)
- Routing by [OpenTripPlanner](https://www.opentripplanner.org/)

## Author

**Syed Haidar Alattas** — Software Engineering, Universiti Tenaga Nasional (UNITEN)
Final Year Project, 2026–2027

## License

Released under the [MIT License](LICENSE).
