```bash
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker system prune --all
docker compose pull
docker compose up -d
docker compose up -d --force-recreate
docker exec -it container-name sh
```
