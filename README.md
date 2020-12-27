# rtsp2srt
Wir wollen uns die RTSP-Streams der IP-Kameras auf den Raspberry Pi holen und dort mittels ffmpeg ins SRT-Protokoll umwandeln und dann zur StreamBox senden.
Dazu muss sowohl auf dem Raspberry Pi und dem großen Computer, also unserer StreamBox auf der OBS läuft, ffmpeg mit integriertem SRT-Protokoll installiert werden.
## Installation auf dem Raspberry Pi (Raspian OS Buster)
```
sudo apt-get update  
sudo apt-get upgrade  
sudo apt-get install tclsh pkg-config cmake libssl-dev build-essential  
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
### auf der StreamBox
- torichter@VMBox-Ubuntu-StreamBox6:~$ srt-live-transmit 'srt://172.16.95.6:40052?mode=listener&latency=1000' udp://localhost:50052
- in OBS eine Medienquelle einfügen mit der URL: udp://localhost:50052
