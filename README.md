# rtsp2srt (für Raspberry Pi)
Secure Reliable Transport (SRT) ist eine Open-Source-Transporttechnologie, die die Streaming-Leistung in Netzwerken wie z.B. dem Internet optimiert.
Wir wollen uns die RTSP-Streams der IP-Kameras auf den Raspberry Pi holen und dort mittels ffmpeg ins SRT-Protokoll umwandeln und dann zu einem Computer senden.
Dazu muss sowohl auf dem Raspberry Pi und dem großen Computer, auf der OBS läuft, ffmpeg mit integriertem SRT-Protokoll installiert werden. Alternativ kann auf dem Computer mit OBS auch die [Haivision App srt-live-transmit](https://github.com/Haivision/srt/blob/master/docs/srt-live-transmit.md) genutzt werden.  
**Die Funktion start_rtsp2srt läuft auf den Raspberry Pi innerhalb des Scriptes GatewaySet.sh und wird in der Datei GatewaySet.conf konfiguriert.**

![rtsp2srt](ffmpeg-srt.png "Streamtransport") 
Streamumwandlung und Streamtransport 

## Installation auf dem Raspberry Pi (Raspian OS Buster)
FFmpeg unterstützt inzwischen standartgemäß das SRT-Protokoll. 
In der Standartinstallation über `apt install ffmpeg` fehlt aktuell aber noch die srt-Bibliothek **libsrt**. Deshalb muss (Stand 12/2020) FFmpeg selbst kompiliert werden und dabei z.B. das Konfigurationsflag `--enable-libsrt` gesetzt werden.  
> Wenn **Raspberry Pi OS with desktop** als Betriebssystem installiert wurde, ist FFmpeg (ohne SRT), als Basis z.B. für den VLC-Player bereits in der Grundinstallation enthalten und sollte deinstalliert werden. Besser ist deshalb die Installation von **Raspberry Pi OS Lite ohne Anwendungsprogramme und ohne GUI** und die nachträgliche Installation aller benötigten Programmpakete!  

VLC-Player inklusive FFmpeg entfernen:  
```
sudo apt purge vlc* && sudo apt purge ffmpeg*  
sudo apt autoremove  
```
und VLC-Symbol aus Startmenu entfernen:  
`sudo rm /usr/share/raspi-ui-overrides/applications/vlc.desktop`  

Die Installation der Quellpakete für FFmpeg inklusive SRT setzt die folgenden Schritte voraus:  
- Bereitstellung aller notwendigen Tools und Bibliotheken, also Download der Abhängigkeiten (Dependencies)
- Konfiguration (mit einem Konfigurationsskript, z.B. `./configure`)  
- Zusammenstellung der Pakete/Bibliotheken/Binärdateien (`make`)  
- Installation (`sudo make install`)  

### Download und Installation der Abhängigkeiten ###
```
sudo apt update && sudo apt upgrade
sudo apt install autoconf dh-autoreconf automake build-essential cmake pkg-config texinfo wget git yasm nasm tcl tclsh libtool libtheora-dev libva-dev libgpac-dev libass-dev libfreetype6-dev libgnutls28-dev libsdl2-dev libsdl1.2-dev libvdpau-dev libvorbis-dev libxcb1-dev libxext-dev libx11-dev libxfixes-dev libxcb-shm0-dev libxcb-xfixes0-dev zlib1g-dev libssl-dev libx264-dev libx265-dev libnuma-dev libx265-doc libvpx-dev libmp3lame-dev libopus-dev  texi2html zlib1g-dev libopus-dev
``` 
Nicht alle der obigen Pakete werden für unser kleines Projekt benötigt. Ich weiß allerdings nicht, welche entbehrlich sind ;-)  
  
Beim Raspberry Pi sind z.B. das Audio-Paket libfdk-aac noch nicht mit `apt` installierbar und muss extra kompiliert werden.  
[Infos hier im FFmpeg CompilitionGuide](https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu "FFmpeg CompilitionGuide")  
```
mkdir /home/pi/ffmpeg_sources
cd /home/pi/ffmpeg_sources  
git -C fdk-aac pull 2> /dev/null || git clone --depth 1 https://github.com/mstorsjo/fdk-aac  
PATH="$HOME/bin:$PATH"  
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig"   
cd fdk-aac  
autoreconf -fiv  
./configure  
make  
sudo make install  
```  

### SRT von Haivision downloaden, kompilieren und installieren (inklusive srt-live-transmit): ###
```
mkdir -p /home/pi/ffmpeg_sources
cd /home/pi/ffmpeg_sources  
sudo git -C srt pull 2> /dev/null || git clone --depth 1  https://github.com/Haivision/srt  
PATH="$HOME/bin:$PATH"
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig" 
cd srt
sudo ./configure  
sudo make  
sudo make install 
```   

### FFmpeg downloaden, konfigurieren (inkl. SRT-Bibliothek einbinden), kompilieren und installieren: ###
```
cd /home/pi/ffmpeg_sources  
sudo git -C FFmpeg pull 2> /dev/null || git clone https://github.com/FFmpeg/FFmpeg.git  
PATH="$HOME/bin:$PATH"  
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig"  
cd FFmpeg
sudo ./configure --extra-ldflags="-latomic" --arch=armel --target-os=linux --enable-gpl --enable-libmp3lame --enable-libfdk-aac --enable-libfreetype --enable-libx264 --enable-libx265  --enable-omx --enable-omx-rpi --enable-nonfree --enable-mmal --enable-libsrt  
sudo make  
sudo make install  
source ~/.profile  
sudo reboot
```  

### testen ###
```
ffmpeg -version	(Version anzeigen)  
ffmpeg -formats	(verfügbare Formate anzeigen)  
ffmpeg -codecs	(verfügbare Codecs anzeigen) 
ffmpeg -protocols (verfügbare Protokolle anzeigen)
ffmpeg -protocols | grep srt
```  

Weiterleitung eines RTSP-Kamerastreams mit niedriger Latenz:  
```
ffmpeg -fflags nobuffer -i 'rtsp://192.168.95.52:554/1/h264major' -c copy -f mpegts 'srt://172.16.95.6:40052?mode=caller&transtype=live&latency=100'  
```
oder mit besserer Qualität (Standarteinstellungen) / Raspberry Pi 4 schafft damit auch 50fps: 
```
ffmpeg -i rtsp://192.168.95.52:554/1/h264major -c copy -f mpegts srt://192.168.95.6:40052
```  
Bildschirmaufnahme [Quelle: https://trac.ffmpeg.org/wiki/Capture/Desktop](https://trac.ffmpeg.org/wiki/Capture/Desktop):  
```
ffmpeg -video_size 1920x1080 -framerate 25 -f x11grab -i :0.0 -f mpegts srt://192.168.95.6:40052
```
