# uiautomatorviewer_docker_novnc

## what it is

a Docker image that runs Android's UI Automator Viewer inside a virtual desktop and shows that desktop in your web browser through noVNC. you can inspect the elements of an Android screen and copy locators for Appium without installing Android Studio.

this is a fork of the noVNC display container https://github.com/theasp/docker-novnc (MIT license, copyright 2017 Andrew Phillips; his license is kept in `app/LICENSE`). the `app/` folder is that upstream project with its history. the Android parts were added on top of it:

| file | what it is |
| --- | --- |
| `novnc.dockerfile` | the added image: Ubuntu 20.04 (`linux/amd64`), OpenJDK 8, Node.js 20, Android SDK tools `sdk-tools-linux-4333796` with `platform-tools`, `build-tools;33.0.0` and `platforms;android-28`, plus `fluxbox`, `x11vnc`, `xvfb`, `novnc`, `supervisor`, `xterm`, `scrcpy` and `tilix`. it copies `app/` to `/app` and starts `/app/entrypoint.sh` |
| `uiautomator-novnc.yaml` | the compose file to start the published image `ghcr.io/clashergeek77/uiautomator-novnc:latest` |
| `.github/workflows/build_image.yml` | builds `novnc.dockerfile` on every push to `master` and pushes the image to ghcr.io with the tags `latest` and a build timestamp |
| `app/` | the upstream container: supervisor config, `entrypoint.sh`, and its own README and example compose file (these are upstream's, they do not describe this image) |

how the pieces connect: UI Automator Viewer (a Java app) draws on a virtual X display (`xvfb`); `x11vnc` serves that display on port 5900; `websockify` serves noVNC and bridges the browser to it on port 8080.

## when to use it

- you need to inspect an Android app's elements to write locators, and you do not want a full Android Studio install.
- you want the viewer on a remote or headless Linux machine and use it from a browser.

when not to use it:

- you want to run mobile tests, not inspect elements: use https://github.com/ClasherGeek77/pytest_android_boilerplate or https://github.com/ClasherGeek77/katalon_android_boilerplate
- you want Appium Inspector on a Linux desktop without Docker: https://github.com/ClasherGeek77/linux_qa_environment_setup_script installs it
- you only want the upstream noVNC container without Android tools: use https://github.com/theasp/docker-novnc

## what you need

- Docker Engine and Docker Compose, from https://docs.docker.com/get-docker/ . the compose file uses `version: '3.7'`. on Linux, `network_mode: "host"` works as expected; on Docker Desktop for macOS and Windows host networking behaves differently (not verified here).
- an x86-64 machine (the image is pinned to `linux/amd64`).
- an Android device or emulator reachable by `adb` (see "if it breaks").
- internet access to pull the image (or to build it: the Dockerfile downloads the Android tools and packages).

## quick start

```bash
git clone https://github.com/ClasherGeek77/uiautomatorviewer_docker_novnc.git
cd uiautomatorviewer_docker_novnc
docker-compose -f uiautomator-novnc.yaml up -d
```

on a Docker with the compose plugin, use `docker compose -f uiautomator-novnc.yaml up -d`.

open http://localhost:8080/vnc.html and click `Connect`.

you should see a small virtual desktop (1600 x 900) with an `xterm` window. in the terminal type:

```bash
uiautomatorviewer
```

then click the device screenshot icon in the viewer to load the screen and its element tree from the connected device. not run here (the image build downloads the Android SDK, and no Android device was connected). what was checked: the compose file, the Dockerfile and the workflow were read; the upstream `app/` scripts expose noVNC on port 8080.

the compose file mounts `./uix` (next to the compose file) to `/root/uix` in the container, so files you save from the viewer to `/root/uix` appear in `uix/` on your machine.

to build the image yourself instead of pulling it:

```bash
docker build -f novnc.dockerfile -t uiautomator-novnc .
```

and change `image:` in `uiautomator-novnc.yaml` to `uiautomator-novnc`.

stop it with `docker-compose -f uiautomator-novnc.yaml down`.

## how to read the code

- start with `uiautomator-novnc.yaml`: one service, port 8080, three environment variables (`DISPLAY_WIDTH`, `DISPLAY_HEIGHT`, `RUN_XTERM`), host networking.
- then `novnc.dockerfile`, top to bottom: tools, Android SDK, display packages, environment variables.
- then `app/entrypoint.sh` and `app/conf.d/*.conf` to see what supervisor starts: `xvfb`, `x11vnc`, `websockify`, `fluxbox`, `xterm`.
- variables you can set (from upstream): `DISPLAY_WIDTH` (default 1024), `DISPLAY_HEIGHT` (768), `RUN_XTERM` (yes), `RUN_FLUXBOX` (yes).

## if it breaks

| symptom | cause | fix |
| --- | --- | --- |
| nothing at `http://localhost:8080` | wrong port, or the container did not start | the port is 8080; run `docker-compose -f uiautomator-novnc.yaml logs` |
| `docker-compose: command not found` | only the compose plugin is installed | use `docker compose` (with a space) |
| `pull access denied` or `not found` for `ghcr.io/clashergeek77/uiautomator-novnc` | the image is not public or not published | build it locally (see quick start) |
| the viewer shows no device | the compose file does not map `/dev/bus/usb`, so a USB device is not passed in. the container shares the host network, so it can reach adb over the network | start adb on the host and connect with `adb connect <device-ip>:5555`, or add a USB device mapping (not verified here) |
| port 8080 already used | host networking uses the host port directly | stop the other service, or change `websockify` port in `app/conf.d/websockify.conf` and rebuild |
| the image does not run on Apple Silicon or ARM | the image is `linux/amd64` | use an x86-64 host, or run under emulation (slow) |
| `uiautomatorviewer` not found in the terminal | the SDK tools folder is not on the PATH of that shell | run `/android-sdk/tools/bin/uiautomatorviewer` (not verified) |
| the Java environment looks unset | the `JAVA_HOME` line in `novnc.dockerfile` is a `RUN` command, so the variable is lost after that step | use the `java` on the PATH from `openjdk-8-jdk` |

## next

- run mobile tests: https://github.com/ClasherGeek77/pytest_android_boilerplate and https://github.com/ClasherGeek77/katalon_android_boilerplate
- a test app to inspect: https://github.com/ClasherGeek77/mobile_app_mock
- Linux workstation setup scripts: https://github.com/ClasherGeek77/linux_qa_environment_setup_script
- upstream project: https://github.com/theasp/docker-novnc
