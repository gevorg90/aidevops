dblocal is the local development database server.

Services include:
- PostgreSQL
- Redis
- MongoDB
- Elasticsearch

Applications using these databases run in the local Kubernetes cluster.

If developers report database connectivity problems:
1. SSH to dblocal.
2. Determine which database service is affected.
3. Check both systemd state and actual service connectivity.
4. Inspect relevant logs.
5. Restart the affected service when appropriate.
6. Verify connectivity.
7. If necessary, investigate/restart the affected Kubernetes application.
