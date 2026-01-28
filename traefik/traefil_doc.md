
https://doc.traefik.io/

## Create a self‑signed certificate
https://doc.traefik.io/traefik/setup/docker/

mkdir -p certs
~ openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/local.key -out certs/local.crt \
  -subj "/CN=*.docker.localhost"

## Create the Traefik Dashboard Credentials
~ htpasswd -nb admin "P@ssw0rd" | sed -e 's/\$/\$\$/g'

