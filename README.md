changedetection.io: fields and data disappear
Problem
Fields and watches added in changedetection.io disappear later, so the data is not saved permanently.

Cause (most likely)
changedetection.io saves all watches, fields and settings in the folder /datastore inside the Docker container. If that folder is not mounted to a folder on the host, it lives only in the container's temporary disk. When the container is deleted and recreated (image update, auto-updater such as Watchtower, docker compose up after a config change, a new docker run), the folder is deleted with it and the new container starts empty.

The fix
Mount /datastore to a folder on the host, so the data lives outside the container:

volumes:
  - ./datastore:/datastore
The full working file is docker-compose.yml in this repo.

How I found it
Ran the app without a volume, added a watch, then removed and recreated the container: the watch was gone.
docker inspect <container> --format '{{json .Mounts}}' returned [], which means nothing is stored outside the container.
Added the volume above, added a watch, ran docker compose down and docker compose up -d: the watch was still there, and the data files were visible in ./datastore on the host.
How to apply the fix on your own Docker server
Important: if your instance already has data, copy it out of the container FIRST (step 2). If you add the volume and recreate the container without doing that, the new container will use the empty host folder and the existing data will be lost.

1. Check whether you have the problem
docker ps --format '{{.Names}}'
docker inspect <container-name> --format '{{json .Mounts}}'
[] or no entry for /datastore: confirmed, the data is not persisted.
An entry with "Destination":"/datastore" and a host path: the volume exists, so the cause is something else (see "If it still happens").
2. Back up the current data out of the container
mkdir -p /opt/changedetection
docker cp <container-name>:/datastore /opt/changedetection/datastore
ls -la /opt/changedetection/datastore
You should see changedetection.json, secret.txt and one folder per watch. Keep a second copy somewhere safe before continuing:

cp -a /opt/changedetection/datastore /opt/changedetection/datastore-backup
3. Add the volume
If you use docker compose, edit your compose file and add the volumes lines under the changedetection service:

services:
  changedetection:
    image: ghcr.io/dgtlmoon/changedetection.io
    container_name: changedetection
    restart: unless-stopped
    stop_grace_period: 30s
    volumes:
      - /opt/changedetection/datastore:/datastore
    ports:
      - "5000:5000"
(Keep your own existing settings, such as environment variables, the port, a browser/playwright service or a reverse proxy. Only the volumes lines are new.)

If you use docker run, add -v:

docker run -d --name changedetection --restart unless-stopped \
  -p 5000:5000 \
  -v /opt/changedetection/datastore:/datastore \
  ghcr.io/dgtlmoon/changedetection.io
4. Recreate the container
# compose:
docker compose down
docker compose up -d

# docker run:
docker rm -f changedetection
# ...then run the docker run command from step 3
5. Check that it worked
docker inspect changedetection --format '{{json .Mounts}}'
It must now show /datastore with a host path. Then:

Open the web interface and check that your watches are still there.
Add a test field or watch.
Recreate the container once more (docker compose down && docker compose up -d).
The test field must still be there.
Check that the file on the host is updated:
ls -la --time-style=full-iso /opt/changedetection/datastore/changedetection.json
If it still happens
Check	Command
Save or permission errors	docker logs --tail 200 changedetection
Disk full	df -h
An auto-updater recreating the container	docker ps | grep -i watchtower
Two containers using the same datastore	docker ps and compare the mounts
Container killed before it saved	keep stop_grace_period: 30s and use docker stop, not docker kill
Settings coming from a tag or group	Check the watch's tags; a tag can override watch settings
Also note that your instance shows version v0.55.8, while I tested on v0.60.10. After the volume is in place and backed up, updating to the newest image is worth trying (docker compose pull && docker compose up -d).

Test it in your own Docker (2 minutes, nothing to install except Docker)
Each copy keeps its own data. Nothing is shared between users, so this runs a fresh, empty copy on your machine.

git clone https://github.com/Leyaaaan1/test-01
cd test-01
docker compose up -d
Open http://localhost:5000 and add a watch.
Recreate the container:
docker compose down
docker compose up -d
Reload the page. The watch is still there.
To see the bug for comparison, run a copy without a volume on another port:

docker run -d --name cd-broken -p 5001:5000 ghcr.io/dgtlmoon/changedetection.io
Add a watch at http://localhost:5001, then run docker rm -f cd-broken and the same docker run again. The watch is gone.

Clean up when finished:

docker compose down
docker rm -f cd-broken
Notes
I did not access your server at ersatzteil-store.ch, so this is the most likely cause based on reproducing it locally, not a confirmed one. The checks above show whether it is the cause on your server.
Do not publish the datastore folder or secret.txt.
Time taken: 1 hours
