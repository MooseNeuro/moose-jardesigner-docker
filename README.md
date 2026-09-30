# Running MOOSE + JARDesigner with Docker

This image bundles MOOSE (the simulator), JupyterLab (to write and run MOOSE
scripts), and JARDesigner (the model-building web GUI) into one self-contained
package. You do not need to install Python, MOOSE, or any of their
dependencies yourself — Docker runs all of it for you, identically on
Windows, macOS, and Linux.

## Step 1: Install Docker

Docker itself needs to be installed once per machine. Pick your OS below.

### Windows

1. Go to <https://www.docker.com/products/docker-desktop/> and download
   **Docker Desktop for Windows**.
2. Run the installer. When prompted, make sure **"Use WSL 2 instead of
   Hyper-V"** is checked (this is the default on modern Windows 10/11 and is
   the recommended option — WSL 2 is a lightweight Linux environment that
   Docker runs inside of, since Docker containers are fundamentally Linux
   containers).
3. Restart your computer if the installer asks you to.
4. Launch **Docker Desktop** from the Start menu and wait for it to say
   "Docker Desktop is running" (a whale icon appears in the system tray).

### macOS

1. Go to <https://www.docker.com/products/docker-desktop/> and download
   **Docker Desktop for Mac** — pick the **Apple Silicon** version if you
   have an M1/M2/M3 Mac, or **Intel chip** if it's an older Mac.
2. Open the downloaded `.dmg` and drag Docker into Applications, as
   instructed.
3. Launch Docker from Applications and wait for the whale icon in the menu
   bar to show it's running.

### Linux

Docker Desktop exists for Linux too, but most Linux users install just the
Docker Engine directly (no GUI wrapper needed). On Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Log out and back in after the last command (it adds your user to the
`docker` group so you don't need `sudo` for every Docker command). For other
distributions, follow the official instructions at
<https://docs.docker.com/engine/install/>.

**How to check the install worked (any OS):** open a terminal (Command
Prompt/PowerShell on Windows, Terminal on macOS/Linux) and run:

```bash
docker --version
```

If it prints a version number, Docker is installed correctly.

## Step 2: Run MOOSE + JARDesigner

Open a terminal and run:

```bash
docker run -d \
  --name moose-jardesigner \
  -p 8888:8888 \
  -p 5000:5000 \
  -v moose_workspace:/workspace \
  -v jardesigner_data:/root/.local/share/jardesigner \
  mooseneuro/moose-jardesigner:latest
```

**What each part of this command does:**

| Part | Meaning |
|---|---|
| `docker run` | Start a new container (a running copy of the image). |
| `-d` | "Detached" — runs in the background, so your terminal stays free to use for other things instead of being taken over by the container's output. |
| `--name moose-jardesigner` | Gives the running container a friendly name, so you can refer to it later (e.g. `docker stop moose-jardesigner`) instead of a random ID. |
| `-p 8888:8888` | Maps port 8888 on your computer to port 8888 inside the container — this is JupyterLab. Format is `host:container`. |
| `-p 5000:5000` | Same idea, for port 5000 — this is JARDesigner. |
| `-v moose_workspace:/workspace` | Creates (or reuses) a named storage volume called `moose_workspace` and connects it to the `/workspace` folder inside the container. Any notebooks/scripts you save there survive even if the container is later removed. |
| `-v jardesigner_data:/root/.local/share/jardesigner` | Same idea, for JARDesigner's own saved projects and uploads. |
| `mooseneuro/moose-jardesigner:latest` | The image to run — this gets automatically downloaded from Docker Hub the first time you run this command, and reused after that. |

The first time you run this, it will take a minute or two to download the
image (a few hundred MB). After that, starting/stopping is instant.

## Step 3: Open it in your browser

Once the container is running, open:

- **JupyterLab** (for running MOOSE Python scripts): <http://localhost:8888>
- **JARDesigner** (the model-building GUI): <http://localhost:5000>

## Running only one of the two

By default the container starts both MOOSE/JupyterLab and JARDesigner
together. If you only need one of them, add the mode as the last word on the
command:

```bash
# Only MOOSE + JupyterLab
docker run -d --name moose-jardesigner -p 8888:8888 \
  -v moose_workspace:/workspace \
  mooseneuro/moose-jardesigner:latest moose

# Only JARDesigner
docker run -d --name moose-jardesigner -p 5000:5000 \
  -v jardesigner_data:/root/.local/share/jardesigner \
  mooseneuro/moose-jardesigner:latest jardesigner
```

The word at the very end (`moose`, `jardesigner`, or `both`) tells the
container's startup script which service(s) to launch.

## Stopping and restarting

```bash
docker stop moose-jardesigner    # stops the container, keeps your data
docker start moose-jardesigner   # starts it again later
```

Your notebooks and JARDesigner projects are safe across `stop`/`start`
because they live in the named volumes (`moose_workspace`,
`jardesigner_data`), not inside the container itself.

## Removing the container entirely

```bash
docker rm -f moose-jardesigner
```

This deletes the running container, but **not your data** — the volumes
persist separately. To start fresh again later, just re-run the `docker run`
command from Step 2 with the same volume names; your notebooks and projects
will still be there.

## Troubleshooting

**"port is already allocated"** — something else on your computer is
already using port 8888 or 5000 (maybe another copy of this container is
still running). Fix: `docker rm -f moose-jardesigner`, then run the command
from Step 2 again. If you genuinely need to run two copies at once, change
the *first* number in `-p 8888:8888` to something free, e.g. `-p 8889:8888`,
and open `http://localhost:8889` instead.

**"container name already in use"** — a container with that name already
exists (even if it's stopped, Docker keeps the name reserved until you
delete it). Fix: `docker rm -f moose-jardesigner`, then re-run.

**Nothing loads in the browser** — give it 10-15 seconds after starting;
JupyterLab and JARDesigner take a moment to boot up inside the container.
Check `docker logs moose-jardesigner` for any startup errors.
