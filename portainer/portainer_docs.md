### Instalation:
creér un volume pour portainer:
~ docker volume create portainer_data
## creer le conteneur portainer
~ docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce

# Pour voir si le conteneur est lancer:
~ docker ps
ip:9000

~ docker stop portainer
reset password:
➜  ~ docker run --rm -v portainer_data:/data portainer/helper-reset-password
Unable to find image 'portainer/helper-reset-password:latest' locally
latest: Pulling from portainer/helper-reset-password
ddb28a3a02c2: Pull complete 
02b4906c8588: Pull complete 
Digest: sha256:327a5580faf6a67e1eb98f4fac41bab14af7425ea88437b680945c995f1529e9
Status: Downloaded newer image for portainer/helper-reset-password:latest
{"level":"info","filename":"portainer.db","time":"2026-01-28T09:33:44Z","message":"loading PortainerDB"}
2026/01/28 09:33:45 Password successfully updated for user: admin
2026/01/28 09:33:45 Use the following password to login: 98woR7]{+h04Y<^qmyx6dPJ5/%1Hg:Wi
~ docker start portainer

## grafana loki
https://grafana.com/docs/alloy/latest/set-up/install/docker/
installer le driver loki
~ docker plugin install grafana/loki-docker-driver:3.6.0-arm64 --alias loki --grant-all-permissions

@check installation:
~ docker plugin ls

Uninstall the Docker driver client
To cleanly uninstall the plugin, disable and remove it:

docker plugin disable loki --force
docker plugin rm loki

Étape 2 : Configurer le daemon Docker
Créez/modifiez /etc/docker/daemon.json :
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "debug": true,
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "http://localhost:3100/loki/api/v1/push",
    "loki-batch-size": "400"
    "loki-retries": "2",
    "loki-max-backoff": "800ms",
    "loki-timeout": "1s",
    "mode": "non-blocking",
    "max-buffer-size": "4m",
  }
}
EOF

sudo systemctl restart docker

collect log with
docker run --log-driver=loki \
    --log-opt loki-url="https://<user_id>:<password>@logs-us-west1.grafana.net/loki/api/v1/push" \
    --log-opt loki-retries=5 \
    --log-opt loki-batch-size=400 \
    grafana/grafana

 logging:
driver: loki
options:
loki-url: http://loki:3100/loki/api/v1/push
loki-retries: "2"
loki-max-backoff: "800ms"
loki-timeout: "1s"
mode: "non-blocking"