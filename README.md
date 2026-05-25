# SRT Live Server

## Introduction

srt-live-server (SLS) is an open source live streaming server for low latency based on Secure Reliable Tranport (SRT).
Normally, the latency of transport by SLS is less than 1 second in internet.

## Web Console Image

This branch adds an embedded web console to the SRTLA-capable server. It is served by the existing HTTP listener, with no separate application process:

- connection URL builder for direct SRT, SRTLA and playback roles
- active publisher monitoring using the existing `/stats` API
- latency, RTT, receive bitrate, loss, drops and receive buffer display
- rolling connection metric charts for a selected stream
- HLS-based browser preview with video, audio playback and video-frame snapshots
- built-in operational help at `/help`
- explicit guidance on the scope of SRT timestamp-based packet delivery (TSBPD)

### Run with Docker Compose

The GitHub workflow publishes branch images to GitHub Container Registry. For this feature branch, use:

```bash
docker pull ghcr.io/themusicnerd/irl-srt-server:gui-dashboard
docker run --rm -p 8181:8181/tcp -p 4000-4002:4000-4002/udp ghcr.io/themusicnerd/irl-srt-server:gui-dashboard
```

For a local build with the included config mounted for editing:

```bash
mkdir -p logs
docker compose -f docker-compose.gui.yml up --build -d
```

Open `http://localhost:8181/` after the image starts.

The default SRT ports exposed by `docker-compose.gui.yml` are:

| Port | Role |
| --- | --- |
| `4000/udp` | Player output for mobile/IRL streams |
| `4001/udp` | Direct SRT publisher input |
| `4002/udp` | SRTLA/bonded publisher input |
| `30002/udp` | Studio direct publisher input |
| `30003/udp` | Studio player output |

The GUI image enables the active-stream table with a bundled local-preview API key:

```conf
api_keys sls-gui-dashboard-local;
```

The dashboard fills this key into Stats Access automatically and sends it as the `Authorization` header for `/stats` requests. Because the image and web console expose this convenience key, it is not a security boundary. Replace it with a random secret in `src/sls.conf` before exposing the dashboard outside a trusted network.

### Media Preview

SRT media is not directly playable in a web browser. The GUI image enables HLS recording for publishers, maintains a live playlist beneath `/tmp/mov/sls`, and serves preview media through `/media/`. Select the publisher role and stream name in the dashboard Feed Monitor to play video and audio or capture the displayed frame.

The default HLS segment length is 2 seconds, so preview is deliberately delayed by several seconds. The initial implementation retains preview segments for the lifetime of the container; production deployments should mount appropriate storage and establish cleanup or retention policy.

Safari can use native HLS playback. Other supported browsers load `hls.js` from jsDelivr in the dashboard, so those browser clients require access to that CDN.

### Timing And Multiple Feeds

TSBPD is active in normal SRT live-mode operation and restores the interval between packets delivered within an SRT connection. The configured/negotiated latency therefore provides a consistent buffering target for each stream.

TSBPD does **not** synchronize separate camera feeds that were generated from independent clocks. For production feeds that must be frame-aligned, use a shared clock/timecode or genlock at source, or perform measured alignment in the receiving mixer. The console reports negotiated latency and receive timing health so mismatched buffering can be found, but it is not a frame synchronizer.

## Requirements

Please install the SRT library first, refer to [SRT](https://github.com/Haivision/srt) for system enviroment setup.
SLS can only run on Unix-based operating systems.

## Compilation

```bash
git submodule update --init
mkdir build && cd build
cmake ../ -DCMAKE_BUILD_TYPE=Release
make -j
```

Binaries are created in `build/bin/` directory.

## Usage

`cd build`

### Help information

```bash
./srt_server -h
```

### Run with default configuration file

```bash
./srt_server -c ../sls.conf
```

## Configuration

Configuration directives are documented on the [wiki page](https://github.com/rstular/srt-live-server/wiki/Directives).

### SRTLA / Bonded Connection Support

SRT Live Server supports both SRTLA (bonded cellular) and direct SRT connections on the same server using separate publisher ports:

```
server {
    listen_player 4000;               # All streams playable here
    listen_publisher 4001;            # Direct SRT (OBS, FFmpeg)
    listen_publisher_srtla 4002;      # SRTLA/bonded (via srtla_rec)
    ...
}
```

- `listen_publisher` - For direct SRT connections (standard behavior)
- `listen_publisher_srtla` - For SRTLA/bonded connections (enables SRTLA patches automatically)
- `listen_player` - Single port for all playback (streams from both publisher types)

**Why separate ports?**
SRTLA bonded connections require special SRT patches that disable dynamic reorder tolerance and periodic NAK reports. Using the wrong setting causes glitching:
- Direct SRT with SRTLA patches = dropped packets
- SRTLA without patches = spurious retransmissions

## Testing

srt-live-server only supports the MPEG-TS format streaming.

### Test with FFmpeg

You can push camera live stream using FFmpeg. FFmpeg must be compiled with `--enable-libsrt` flag - to obtain appropriate binaries, download FFmpeg sourcecode from https://github.com/FFmpeg/FFmpeg, then compile FFmpeg with `--enable-libsrt`.

`srt` library is installed in folder `/usr/local/lib64`.

If `ERROR: srt >= 1.3.0 not found using pkg-config` occurs during the compilation of FFmpeg, please check the `ffbuild/config.log` file and follow its instruction to resolve this issue. In most cases it can be resolved by executing the following command:

```bash
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/lib64/pkgconfig
```

If `error while loading shared libraries: libsrt.so.1` occurs, please add `srt` library path to the runtime linker configuration file, `/etc/ld.so.conf`, then refresh the cache by running the comand `/sbin/ldconfig` as root.

#### Push stream from webcam to SRT

```bash
./ffmpeg -f avfoundation -framerate 30 -i "0:0" -vcodec libx264  -preset ultrafast -tune zerolatency -flags2 local_header  -acodec libmp3lame -g  30 -pkt_size 1316 -flush_packets 0 -f mpegts "srt://[your.sls.ip]:8080?streamid=uplive.sls/live/test"
```

#### Play a SRT stream using FFplay

```bash
./ffplay -fflags nobuffer -i "srt://[your.sls.ip]:8080?streamid=live.sls/live/test"
```

### Test with OBS

OBS supports SRT protocol to publish streams from version `v25.0` onwards. To publish SRT stream from OBS to SRT Live Server you can use the following url:

```
srt://[your.sls.ip]:8080?streamid=uplive.sls/live/test
```

You can also add a SRT stream as an input source. To do this, add a `Media source` to OBS, enter `mpegts` as input format and set the following input URL:

```
srt://[your.sls.ip]:8080?streamid=live.sls/live/test
```

### Test with SRT Live Client

There is a test tool in SLS which can be used as a performance test - it has no codec overhead, only network overhead. The SRT Live Client can play a SRT stream to a TS file, or push a TS file to a SRT stream.

#### Push a TS file via SRT

```bash
./srt_client -r srt://[your.sls.ip]:8080?streamid=uplive.sls/live/test -i [the full file name of exist ts file]
```

#### Play a SRT stream

```bash
./srt_client -r srt://[your.sls.ip]:8080?streamid=live.sls/live/test -o [the full file name of ts file to save]
```

## Use SLS with docker

Please refer to: https://hub.docker.com/r/ravenium/srt-live-server

## Development

To build a debug build of the SRT Live Server, run the following commands:

```bash
git submodule update --init
mkdir build && cd build
cmake ../ -DCMAKE_BUILD_TYPE=Debug
make -j
```

## Note:

- SLS refers to the RTMP url format (domain/app/stream_name), example: www.sls.com/live/test. The URL must be set in streamid parameter of SRT, which will be the unique identification a stream.

- How to distinguish the publisher and player of the same stream? In the configuration file file, you can set parameters of domain_player/domain_publisher and app_player/app_publisher to resolve it. Importantly, the two combination strings of domain_publisher/app_publisher and domain_player/app_player must not be equal in the same server block.

- I supplied a simple android app for testing SLS, which can be downloaded from https://github.com/Edward-Wu/liteplayer-srt
