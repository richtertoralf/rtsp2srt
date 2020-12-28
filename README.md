# rtsp2srt
Wir wollen uns die RTSP-Streams der IP-Kameras auf den Raspberry Pi holen und dort mittels ffmpeg ins SRT-Protokoll umwandeln und dann zur StreamBox senden.
Dazu muss sowohl auf dem Raspberry Pi und dem großen Computer, also unserer StreamBox auf der OBS läuft, ffmpeg mit integriertem SRT-Protokoll installiert werden.

![rtsp2srt](ffmpeg-srt.png "Streamtransport") 
Streamumwandlung und Streamtransport 

## Installation auf dem Raspberry Pi (Raspian OS Buster)
FFmpeg unterstützt inzwischen standartgemäß das SRT-Protokoll. 
In der Standartinstallation über `apt install ffmpeg` fehlt aktuell aber noch die srt-Bibliothek **libsrt**. Deshalb muss (Stand 12/2020) FFmpeg selbst kompiliert werden und dabei das Konfigurationsflag `--enable-libsrt` gesetzt werden.  
Die Installation der Quellpakete setzt die folgenden Schritte voraus:  
- Konfiguration (mit einem Konfigurationsskript, z.B. `./configure`)  
- Zusammenstellung der Pakete/Bibliotheken/Binärdateien (`make`)  
- Installation (`sudo make install`)  

```
sudo apt update  
sudo apt install \  
  autoconf \  
  automake \  
  build-essential \  
  cmake \  
  git-core \  
  libass-dev \  
  libfreetype6-dev \  
  libgnutls28-dev \  
  libsdl2-dev \  
  libtool \  
  libva-dev \  
  libvdpau-dev \  
  libvorbis-dev \  
  libxcb1-dev \  
  libxcb-shm0-dev \  
  libxcb-xfixes0-dev \  
  pkg-config \  
  texinfo \  
  wget \  
  git \  
  yasm \  
  zlib1g-dev \  
  tclsh \  
  libssl-dev \  
  nasm \  
  libx264-dev \  
  libx265-dev  \
  libnuma-dev \  
  libx265.doc \  
  libvpx-dev \  
  libmp3lame \  
  libopus-dev \  
```  
Damit wurden alle Pakete, die zum Kompilieren notwendig sind, geholt.
Beim Raspberry Pi sind z.B. das Audio-Paket libfdk-aac und die AV1 Encoder/Decoder noch nicht mit `apt` installierbar und müssten, wenn benötigt, extra kompiliert werden. [Infos hier im FFmpeg CompilitionGuide](https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu "FFmpeg CompilitionGuide")  
```
cd /home/pi  
sudo git clone https://github.com/Haivision/srt  
cd srt  
sudo ./configure  
sudo make  
sudo make install  
cd /home/pi  
git clone https://github.com/FFmpeg/FFmpeg.git  
cd FFmpeg  
sudo -s  
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig"
./configure --arch=armel --target-os=linux --enable-gpl --enable-omx --enable-omx-rpi --enable-nonfree --enable-mmal --enable-libsrt
# eigentlich müsste auch Folgendes eingeschaltet werden: --enable-libx264 --enable-libx265 --enable-libx265  
# funktioniert aber nicht ???
make

```

ffmpeg ist jetzt nicht installiert, aber kompiliert und kann im aktuellen Arbeitsverzeichnis ausgeführt werden. 
Das müsste jetzt /home/pi/ffmpeg sein.

Aufruf mit:
```
ffmpeg -fflags nobuffer -i 'rtsp://admin:admin@192.168.95.52:554/1/h264major' -c copy -f mpegts 'srt://172.16.95.6:40052?mode=caller&transtype=live&latency=1000000'
```

## Installation auf der StreamBox (Ubuntu 20.04 LTS)

## Testen
### Stream auf dem Raspi empfangen und senden
- `ffmpeg -fflags nobuffer -i 'rtsp://admin:admin@192.168.95.52:554/1/h264major' -c copy -f mpegts 'srt://172.16.95.6:40052?mode=caller&transtype=live&latency=1000000'
`   
### auf der StreamBox
- `srt-live-transmit 'srt://172.16.95.6:40052?mode=listener&latency=1000' udp://localhost:50052`  
- in OBS eine Medienquelle einfügen mit der URL: udp://localhost:50052  
