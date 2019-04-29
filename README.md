# docker-wine

![docker-wine logo](https://raw.githubusercontent.com/scottyhardy/docker-wine/master/logo.png)

Included in the [scottyhardy/docker-wine GitHub repository](https://github.com/scottyhardy/docker-wine) are scripts to enable you to build a Docker container that runs [Wine](https://www.winehq.org). The  container is based on Ubuntu 16.04 and includes the latest version of [Winetricks](https://wiki.winehq.org/Winetricks). Included below are instructions for running the `docker-wine` container to allow you to use the host's X11 session to display graphics and the host's PulseAudio server for sound without needing to compromise security.

## Running from Docker Hub image

### Create a Docker volume container for user data

Create a volume container so user data is kept separate and can persist after the `docker-wine` container is removed:

```bash
docker volume create winehome
```

### Run without sound

The recommended commands for running `docker-wine` securely are:

```bash
docker run -it \
    --rm \
    --env="DISPLAY" \
    --volume="${XAUTHORITY}:/root/.Xauthority:ro" \
    --volume="winehome:/home/wine" \
    --net="host" \
    --name="wine" \
    scottyhardy/docker-wine <Additional arguments e.g. wine notepad.exe>
```

This assumes the `$XAUTHORITY` environment variable is set to the location of the MIT magic cookie.  If not set, the default location is in the user's home so you can replace `${XAUTHORITY}` with `${HOME}/.Xauthority`. This file is required to allow the container to write to the current user's X session. For this to work you also need to include the `--net=host` argument when executing `docker run` to use the host's network stack which includes the X11 socket.

### Run using PulseAudio for sound

As a one-off, you will need to create the file `.config/pulse/default.pa` in your home folder, to enable you to create a shared UNIX socket `/tmp/pulse-socket` to allow other users on the same host to access the user's PulseAudio server:

```bash
mkdir -p "${HOME}/.config/pulse"
echo -e ".include /etc/pulse/default.pa\nload-module module-native-protocol-unix auth-anonymous=1 socket=/tmp/pulse-socket" > ${HOME}/.config/pulse/default.pa
```

Restart your PulseAudio server to create the new socket:

```bash
pulseaudio -k
pulseaudio --start
```

Now you're ready to run the container using PulseAudio for sound:

```bash
docker run -it \
    --rm \
    --env="DISPLAY" \
    --volume="${XAUTHORITY}:/root/.Xauthority:ro" \
    --volume="/tmp/pulse-socket:/tmp/pulse-socket" \
    --volume="winehome:/home/wine" \
    --net="host" \
    --name="wine" \
    scottyhardy/docker-wine <Additional arguments e.g. winetricks vlc>
```

## Build and run locally on your PC

First, clone the repository from GitHub:

```bash
git clone https://github.com/scottyhardy/docker-wine.git
```

To build the container, simply run:

```bash
make
```

To run the container and start an interactive session with `/bin/bash` run either:

```bash
make run
```

or use the `docker-wine` script as described below.

## Running the `docker-wine` script

When the container is run with the `docker-wine` script, you can override the default interactive bash session by adding `wine`, `winetricks`, `winecfg` or any other valid commands with their associated arguments:

```bash
./docker-wine wine notepad.exe
```

```bash
./docker-wine winecfg
```

```bash
./docker-wine winetricks msxml3 dotnet40 win7
```

## Volume container `winehome`

When the docker-wine image is instantiated with `./docker-wine` script or with the recommended `docker volume create` and `docker run` commands, the contents of the `/home/wine` folder is copied to the `winehome` volume container on instantiation of the `wine` container.

Using a volume container allows the `wine` container to remain unchanged and safely removed after every execution with `docker run --rm ...`.  Any user environments created with `wine` will be stored separately and user data will persist as long as the `winehome` volume is not removed.  This effectively allows the `docker-wine` image to be swapped out for a newer version at anytime.

You can manually create the `winehome` volume container by running:

```bash
docker volume create winehome
```

If you don't want the volume container to persist after running `./docker-wine`, just add `--rm` as your first argument.
e.g.

```bash
./docker-wine --rm wine notepad.exe

```

Alternatively you can manually delete the volume container by using:

```bash
docker volume rm winehome
```

## `ENTRYPOINT` script explained

The `ENTRYPOINT` set for the docker-wine image is simply `/usr/bin/entrypoint`. This script is key to ensuring the user's `.Xauthority` file is copied from `/root/.Xauthority` to `/home/wine/.Xauthority` and ownership of the file is set to the `wine` user each time the container is instantiated.

Arguments specified after `./docker-wine` or after the `docker run ... docker-wine` command are also passed to this script to ensure it is executed as the `wine` user.

For example:

```bash
./docker-wine wine notepad.exe
```

The arguments `wine notepad.exe` are interpreted by the wine container to override the `CMD` directive, which otherwise simply runs `/bin/bash` to give you an interactive bash session as the `wine` user in the container.

## Using docker-wine in your own Dockerfile

If you plan to use `scottyhardy/docker-wine` as a base for another Docker image, you should set up the same entrypoint to ensure you run as the `wine` user and X11 graphics continue to function by adding the following to your `Dockerfile`:

```dockerfile
FROM scottyhardy/docker-wine:latest
... <your code here>
ENTRYPOINT ["/usr/bin/entrypoint"]
```

Or if you prefer to run a program by default you could use:

```dockerfile
ENTRYPOINT ["/usr/bin/entrypoint", "wine", "notepad.exe"]
```

Or if you want to be able to run a program by default but still be able to override it easily you could use:

```dockerfile
ENTRYPOINT ["/usr/bin/entrypoint"]
CMD ["wine", "notepad.exe"]
```
