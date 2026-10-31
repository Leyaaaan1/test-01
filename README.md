# changedetection.io: data not saved permanently

## Problem
Fields and watches added in changedetection.io disappear later.

## Cause (most likely)
The container has no persistent volume. changedetection.io stores all
watches and settings in `/datastore` inside the container. When the
container is deleted and recreated (app update, config change,
auto-updater, etc.), that folder is deleted with it, so everything
that was added disappears.

## How I reproduced it
    docker run -d --name cd-broken -p 5000:5000 ghcr.io/dgtlmoon/changedetection.io

1. Added a watch for http://ersatzteil-store.ch/
2. Ran `docker rm -f cd-broken` and the same `docker run` again
3. The watch was gone
4. `docker inspect cd-broken` showed `Mounts: []`, so nothing was
   stored outside the container

## The fix
Mount `/datastore` to a folder on the host (see `docker-compose.yml`):

    volumes:
      - ./datastore:/datastore

Start it with:

    docker compose up -d

## How I verified it
1. Added the watch for http://ersatzteil-store.ch/
2. Ran `docker compose down` and then `docker compose up -d`
3. The watch was still there
4. The data files are visible in `./datastore` on the host

## Live test
Link: https://suspended-illinois-notebooks-motor.trycloudflare.com
Password: sent on WhatsApp

The link runs from my computer, so it is only available while my
computer is online. If it is down, message me and I will restart it
(the link changes after a restart). I can also restart the container
live so you can see the data survive.

## Test it yourself
    git clone https://github.com/YOURNAME/changedetection-fix
    cd changedetection-fix
    docker compose up -d

1. Open http://localhost:5000 and add a watch
2. Run `docker compose down` and then `docker compose up -d`
3. Reload: the watch is still there

## Screenshots
- `screenshots/before-bug.png`: watch lost after recreating the container
- `screenshots/after-fix.png`: watch still there after down/up with the fix

## Note
This is the most likely cause, based on reproducing it locally. I have
not seen your server, so on the live instance I would check:

- `docker inspect <container> --format '{{json .Mounts}}'` (is /datastore mounted?)
- `docker logs <container>` (permission or write errors?)
- `df -h` (is the disk full?)
- Whether Watchtower or another auto-updater recreates the container
- Whether two containers share the same datastore

Time taken: X hours
