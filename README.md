# Cloud

## Run locally with Docker

Install Docker with Docker Compose, then run from the repository root:

```sh
docker compose up --build -d
```

- Frontend (Vite development server): http://localhost:5173
- Backend health endpoint: http://localhost:5000/api/health
- MongoDB: `mongodb://mongodb:27017/cloud` inside the Compose network.

MongoDB data is stored in the `mongodb_data` volume and survives container restarts.
The backend waits for MongoDB to be healthy before starting.

This setup is for local development. It uses a default development JWT secret
and MongoDB without authentication, accessible only inside the Docker network.
You can override `JWT_SECRET` in a root `.env` file.
The existing login and signup screens do not yet call backend authentication routes.

View logs:

```sh
docker compose logs -f
```

Stop the containers (keep database data):

```sh
docker compose down
```

To also delete the database data, use `docker compose down -v`.

## Run locally with Kubernetes

See [the Kubernetes guide](k8s/README.md) for Linux/Windows prerequisites,
deployment, and a demonstration of scaling the backend from one to three replicas.
