# Jenkins + Python Flask + Docker CI Pipeline

A CI pipeline that builds, tests, and containerizes a simple Flask app using Jenkins running inside Docker.

> **Note:** This is currently a CI/test pipeline, not a full production-style deployment pipeline (see [Lessons Learned](#lessons-learned)).

## Architecture

```
GitHub Repository
       |
       | Git Clone
       v
    Jenkins
       |
       | Python Build
       v
   Python Virtual Env
       |
       | Python Test
       v
   Docker Build
       |
       v
 Docker Image
       |
       | Docker Run
       v
Flask Container
       |
       | Port 5000
       v
Flask Application
```

The application returns: `Hello from Balaji Flask App!`

## Project Structure

```
code-base/
├── app.py
├── Dockerfile
├── Jenkinsfile
├── requirements.txt
├── README.md
└── .gitignore
```

## Prerequisites

- Jenkins running inside Docker, with the host Docker socket mounted:
  ```bash
  docker run -d \
    --name jenkins-lab \
    -p 8080:8080 \
    -p 50000:50000 \
    -v jenkins-data:/var/jenkins_home \
    -v /var/run/docker.sock:/var/run/docker.sock \
    myjenkins:lts-jdk21
  ```
- The Jenkins container's Docker group **GID must match the host's `docker` group GID**, and the Jenkins user must belong to that group — otherwise Jenkins can see the socket but gets a permission-denied error when calling Docker. Check the host GID with:
  ```bash
  getent group docker
  ```
- `python3-venv` installed inside the Jenkins container (needed for the Python build stage).

## Flask Application

**`app.py`**
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from Balaji Flask App!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

> **Important:** Flask must listen on `0.0.0.0`, not `127.0.0.1`. Inside a container, `127.0.0.1` refers only to the container itself; `0.0.0.0` allows connections through the container's network interface.

## Dockerfile

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

## Pipeline Stages

1. **Git Clone** — pull the repo
2. **Python Build** — create a virtual environment and install dependencies
3. **Python Test** — run tests inside the venv
4. **Docker Build** — build the Flask image
5. **Docker Run** — run the container and verify it responds
6. **Post: Cleanup** — remove the test container

## Lessons Learned

A running log of the real issues hit while building this pipeline, and how they were fixed.

<details>
<summary><strong>1. Jenkins initially used Maven</strong></summary>

Jenkins expected a `pom.xml`, but this is a Python project. The build process must match the technology stack — switched from a Maven build to Python Build → Python Test → Docker Build → Docker Run.
</details>

<details>
<summary><strong>2. PEP 668 / "externally-managed-environment" error</strong></summary>

Modern Debian/Ubuntu Python installations block global `pip install`. Fixed by creating a virtual environment and installing into it:
```bash
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
```
</details>

<details>
<summary><strong>3. Virtual environment creation failed</strong></summary>

`python3 -m venv venv` failed because `python3-venv` wasn't installed in the Jenkins container. Fixed with:
```bash
apt update
apt install -y python3.13-venv
```
</details>

<details>
<summary><strong>4. Jenkins couldn't run Docker</strong></summary>

The Jenkins container had no access to `/var/run/docker.sock`. Fixed by mounting the host socket:
```bash
-v /var/run/docker.sock:/var/run/docker.sock
```
</details>

<details>
<summary><strong>5. Docker permission denied</strong></summary>

Mounting the socket wasn't enough — the Jenkins user also needs permission to use it. Docker socket access depends on the host's `docker` group GID (`getent group docker`). Fixed by matching that GID inside the Jenkins container and adding the Jenkins user to the group. (Deliberately avoided the insecure `chmod 666 /var/run/docker.sock` workaround.)
</details>

<details>
<summary><strong>6. Jenkinsfile Markdown formatting error</strong></summary>

Pasting the Jenkinsfile with Markdown code fences (```` ```groovy ````) caused `unexpected char: '`'`. The Pipeline editor expects raw Groovy — paste only the `pipeline { ... }` block, no fences.
</details>

<details>
<summary><strong>7. Flask app unreachable from Jenkins</strong></summary>

`curl http://localhost:5000` from Jenkins failed because Jenkins itself runs inside its own container — `localhost` there means the Jenkins container, not the Flask container. Fixed by testing from inside the Flask container instead:
```bash
docker exec balaji-flask-container \
  python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:5000').read().decode())"
```
</details>

<details>
<summary><strong>8. App disappeared after the pipeline finished</strong></summary>

The `post { always { docker rm -f balaji-flask-container } }` block removed the container after every run — including successful ones — so nothing was left running afterward. This is expected for a CI/test pipeline; a real deployment stage (e.g. deploy to a persistent environment) would be needed to keep it running.
</details>

## License

MIT (or your preferred license)
