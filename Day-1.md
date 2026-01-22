## Docker – Quick Refresher

- Docker lets you package your app and its dependencies into a portable unit that runs the same way everywhere.  
- An **image** is a blueprint/template: it has your code, dependencies, and a minimal OS layer.  
- A **container** is a running instance of an image. You can start, stop, and remove containers without changing the image.  

Basic workflow:

1. Write a `Dockerfile` that describes how to build the image.  
2. Build the image:  
   ```bash
   docker build -t myimage .
   ```  
3. Run a container from the image:  
   ```bash
   docker run -p 8000:80 myimage
   ```  

Useful commands:

```bash
# List running containers
docker ps

# List local images
docker images

# Stop and remove a container
docker stop <container_id>
docker rm <container_id>
```

***

## Minimal FastAPI API – Notes

### Basic structure

- Create a file `main.py`.  
- Define a FastAPI app and at least one route:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello from FastAPI"}
```

### Running locally (no Docker yet)

Install dependencies (for example, in a virtual environment):

```bash
pip install fastapi "uvicorn[standard]"
```

Run the app:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

Check:

- `http://127.0.0.1:8000` → should return the JSON response.  
- `http://127.0.0.1:8000/docs` → interactive Swagger UI.  

***

## Project Layout for Dockerized FastAPI

A simple layout:

```text
.
├── main.py
├── requirements.txt
└── Dockerfile
```

`requirements.txt`:

```text
fastapi
uvicorn[standard]
```

***

## Dockerfile for FastAPI – Example

```dockerfile
# Use a lightweight Python base image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose the port FastAPI will run on inside the container
EXPOSE 80

# Command to run the app with Uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
```

Key points:

- `WORKDIR /app` sets the default directory for subsequent commands.  
- Dependencies are installed first so Docker can reuse the layer when only code changes.  
- `EXPOSE 80` documents the container’s internal port (FastAPI will listen on 80).  
- `CMD` starts Uvicorn and points it at `main:app`.  

***

## Building and Running the Container

From the folder containing the `Dockerfile`:

### Build the image

```bash
docker build -t fastapi-demo .
```

### Run the container

```bash
docker run -p 8000:80 fastapi-demo
```

What this does:

- Maps host port `8000` → container port `80`.  
- Inside the container, FastAPI listens on port `80`.  
- On your machine, you access it via port `8000`.  

### Test the API in Docker

- Open in browser:  
  - `http://localhost:8000` → should show the JSON from `read_root`.  
  - `http://localhost:8000/docs` → interactive docs, same as local non‑Docker run.  

***

## Quick Troubleshooting Notes 

- If the container exits immediately, check the logs:  
  ```bash
  docker logs <container_id>
  ```  
- If the browser cannot connect, verify:  
  - The container is running (`docker ps`).  
  - Ports are mapped correctly (`-p 8000:80`).  
  - Uvicorn is listening on `0.0.0.0` and the same port you exposed in Docker.  


