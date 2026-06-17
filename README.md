# VoteSecure — Online Voting System

A full-stack **Online Voting System** built with **Python Flask + SQLite**, containerised with **Docker**, and deployed via a **Jenkins CI/CD pipeline** — designed as a DevOps mini project.

---

## Project Structure

```
online-voting-system/
├── app/
│   ├── __init__.py          # Flask app factory
│   ├── database.py          # SQLite DB init & helpers
│   ├── routes.py            # All URL routes & logic
│   └── templates/
│       ├── base.html        # Shared layout + navbar
│       ├── index.html       # Home / landing page
│       ├── register.html    # Voter registration
│       ├── login.html       # Voter login
│       ├── vote.html        # Ballot page
│       └── results.html     # Live results page
├── tests/
│   └── test_voting.py       # Pytest unit tests
├── Dockerfile               # Multi-stage Docker build
├── docker-compose.yml       # Docker Compose config
├── Jenkinsfile              # Jenkins CI/CD pipeline
├── requirements.txt         # Python dependencies
├── run.py                   # App entry point
├── .gitignore
└── .dockerignore
```

---

## Features

- Voter registration with unique Voter ID (e.g. `VTR-AB12CD34`)
- Secure login using Voter ID + email
- One-vote-per-voter enforcement
- Live results with percentage bar charts
- REST API endpoint `/api/results`
- Fully Dockerised (multi-stage build)
- Jenkins CI/CD pipeline (9 stages)
- 20+ unit tests with pytest

---

## Tech Stack

| Layer      | Technology          |
|------------|---------------------|
| Backend    | Python 3.11, Flask 3 |
| Database   | SQLite (via Python stdlib) |
| Frontend   | Jinja2 templates, CSS |
| Container  | Docker (multi-stage) |
| Orchestration | Docker Compose   |
| CI/CD      | Jenkins (Declarative Pipeline) |
| Testing    | pytest, pytest-flask |

---

## Quick Start (Local)

### 1. Clone & Run with Python

```bash
git clone https://github.com/yourname/online-voting-system.git
cd online-voting-system

python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

python run.py
# Visit http://localhost:5000
```

### 2. Run with Docker

```bash
# Build the image
docker build -t votesecure:latest .

# Run the container
docker run -d \
  --name votesecure_app \
  -p 5000:5000 \
  -v votesecure_data:/app/data \
  votesecure:latest

# Visit http://localhost:5000
```

### 3. Run with Docker Compose

```bash
docker-compose up -d
# Visit http://localhost:5000

# Stop
docker-compose down
```

### 4. Run Tests

```bash
source venv/bin/activate
pytest tests/ -v
```

---

## Database Schema

### `voters`
| Column     | Type    | Description                  |
|------------|---------|------------------------------|
| id         | INTEGER | Primary key                  |
| name       | TEXT    | Full name                    |
| email      | TEXT    | Unique email                 |
| voter_id   | TEXT    | Unique ID (VTR-XXXXXXXX)     |
| has_voted  | INTEGER | 0 = not voted, 1 = voted     |
| created_at | TIMESTAMP | Registration time          |

### `candidates`
| Column     | Type    | Description                  |
|------------|---------|------------------------------|
| id         | INTEGER | Primary key                  |
| name       | TEXT    | Candidate name               |
| party      | TEXT    | Party name                   |
| symbol     | TEXT    | Emoji symbol                 |
| vote_count | INTEGER | Number of votes received     |

### `votes`
| Column       | Type    | Description                |
|--------------|---------|----------------------------|
| id           | INTEGER | Primary key                |
| voter_id     | TEXT    | FK → voters.voter_id       |
| candidate_id | INTEGER | FK → candidates.id         |
| voted_at     | TIMESTAMP | Time of vote             |

### `election_settings`
| Column        | Type    | Description               |
|---------------|---------|---------------------------|
| election_name | TEXT    | Name of the election      |
| is_active     | INTEGER | 1 = active, 0 = closed    |

---

## API Endpoints

| Method | URL           | Description              |
|--------|---------------|--------------------------|
| GET    | `/`           | Home page                |
| GET    | `/register`   | Registration form        |
| POST   | `/register`   | Submit registration      |
| GET    | `/login`      | Login form               |
| POST   | `/login`      | Submit login             |
| GET    | `/vote`       | Ballot page (auth req.)  |
| POST   | `/vote`       | Submit vote (auth req.)  |
| GET    | `/results`    | Live results page        |
| GET    | `/api/results`| JSON results API         |
| GET    | `/logout`     | Logout & clear session   |

---

## Jenkins CI/CD Pipeline

The `Jenkinsfile` defines **9 stages**:

```
Checkout → Install Dependencies → Run Tests → Code Quality
    → Build Docker Image → Test Docker Image
        → Push to Registry → Deploy → Cleanup
```

### Stage Breakdown

| Stage | What it does |
|-------|-------------|
| **Checkout** | Pulls latest code from Git |
| **Install Dependencies** | Creates venv, installs pip packages |
| **Run Tests** | Runs pytest, publishes JUnit XML report |
| **Code Quality** | Runs flake8 linting |
| **Build Docker Image** | Builds multi-stage Docker image, tags with build number |
| **Test Docker Image** | Spins up a test container, runs HTTP smoke test |
| **Push to Registry** | Pushes image to Docker Hub (main branch only) |
| **Deploy** | Stops old container, starts new one on port 5000 |
| **Cleanup** | Removes dangling images, cleans workspace |

### Setting Up Jenkins

1. Install Jenkins and the following plugins:
   - **Git plugin**
   - **Docker plugin**
   - **Pipeline plugin**
   - **JUnit plugin**

2. Create a new **Pipeline** job in Jenkins.

3. Under **Pipeline**, choose **Pipeline script from SCM**:
   - SCM: Git
   - Repository URL: `https://github.com/yourname/online-voting-system`
   - Script Path: `Jenkinsfile`

4. (Optional) Add Docker Hub credentials:
   - Go to **Manage Jenkins → Credentials**
   - Add credentials with ID: `docker-hub-credentials`
   - Set `DOCKER_REGISTRY` in `Jenkinsfile` to your Docker Hub username

5. Click **Build Now** — the full pipeline runs automatically.

---

## Docker Details

### Multi-Stage Build
The `Dockerfile` uses **two stages**:
- **Stage 1 (builder):** Installs all pip dependencies into `/install`
- **Stage 2 (production):** Copies only the installed packages and app source — resulting in a smaller, cleaner image

### Security
- Runs as a **non-root user** (`appuser`) inside the container
- SQLite database stored in a **named Docker volume** (persists across restarts)
- Built-in Docker `HEALTHCHECK` polls the app every 30 seconds

---

## Environment Variables

| Variable        | Default         | Description              |
|-----------------|-----------------|--------------------------|
| `DATABASE_PATH` | `voting.db`     | Path to SQLite database  |
| `FLASK_ENV`     | `production`    | Flask environment mode   |
| `FLASK_APP`     | `run.py`        | Flask entry point        |

---

## Screenshots (Pages)

| Page | URL | Description |
|------|-----|-------------|
| Home | `/` | Hero banner, stats, candidates list |
| Register | `/register` | Voter sign-up form |
| Login | `/login` | Voter ID + email login |
| Vote | `/vote` | Radio-button ballot |
| Results | `/results` | Live bar chart results |

---

## Author

Built as a DevOps Mini Project demonstrating Git, Jenkins, and Docker for CI/CD workflows.
