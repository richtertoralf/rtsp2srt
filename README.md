# rtsp2srt (für Raspberry Pi)
>Secure Reliable Transport (SRT) ist eine Open-Source-Transporttechnologie, die die Streaming-Leistung in Netzwerken wie z.B. dem Internet optimiert.  

In unseren Anwendungsfällen ist es z.B. notwendig, HD RTSP Streams mit 30 bis 60 fps von simplen Überwachungskameras in guter Qualität über Wifi-Netzwerke, 4G/LTE und das Internet weiterzuleiten. Außerdem gibt es Anwendungsfälle, bei dem wir Videostreams über das Internet versenden und und diese dann auf Bildschirmen die an RaspberryPis angeschlossen sind, anzeigen lassen wollen.

Wir wollen uns zuerst den RTSP-Stream der IP-Kameras auf den RaspberryPi holen und dort mittels FFmpeg ins SRT-Protokoll umwandeln und dann zu einem entfernten Computer über ein Netzwerk senden. Wir verwenden dafür FFmpeg, da es sehr gut dokumentiert und OpenSource ist.  

Dazu muss sowohl auf dem RaspberryPi und dem entfernten Computer, auf dem wir z.B. OBS-Studio laufen haben, FFmpeg mit der SRT-Bibliothek installiert werden. Alternativ kann auf dem entfernten Computer auch die [Haivision App srt-live-transmit](https://github.com/Haivision/srt/blob/master/docs/srt-live-transmit.md) als Empfänger genutzt werden. Es gibt inzwischen auch Fälle, bei denen OBS-Studio den SRT-Stream selbst empfangen kann.  
Um das zu verstehen, sollte man sich die die "caller" und "listner" Konzepte von SRT anschauen.  

**Die Funktion rtsp2srt läuft auf den Raspberry Pi´s in unserer Livestreamanwendung. Zusätzlich werden in unserer Anwendung Gateways verwaltet und WireGuard als VPN genutzt. In unserer Anwendung werden automatisch angeschlossene Kameras erkannt, es wird geprüft, auf welchem Weg (LAN, Wifi oder 4G/LTE) der enfernten Computer, auf dem OBS-Studio läuft, erreichbar ist. Je nach Qualität der Verbindung (Ping und Bandbreite) wird z.B. die Bitrate der IP-Kameras eingestellt.**  

Du kannst diese Anleitung zum Installieren von FFmpeg und SRT aber auch nutzen, um z.B. den RaspberryPi als SRT-Player zu nutzen. FFmpeg bringt die Anwendung FFplay mit. Damit kannst du dann SRT-Streams auf dem RaspberryPi abspielen und auf einem Bildschirm ansehen.

![rtsp2srt](ffmpeg-srt.png "Streamtransport") 
Beuispiel Streamumwandlung und Streamtransport 


## Installation auf dem Raspberry Pi (Raspian OS Buster)
FFmpeg unterstützt inzwischen standartgemäß das SRT-Protokoll. 
In der Standartinstallation über `apt install ffmpeg` fehlt aktuell aber noch die srt-Bibliothek **libsrt**. Deshalb muss (Stand 12/2020) FFmpeg selbst kompiliert werden und dabei z.B. das Konfigurationsflag `--enable-libsrt` gesetzt werden.  
> Wenn **Raspberry Pi OS with desktop** als Betriebssystem installiert wurde, ist FFmpeg (ohne SRT), als Basis z.B. für den VLC-Player bereits in der Grundinstallation enthalten und sollte deinstalliert werden. Besser ist deshalb die Installation von **Raspberry Pi OS Lite ohne Anwendungsprogramme und ohne GUI** und die nachträgliche Installation aller benötigten Programmpakete!  

Wenn du RaspberryPi mit einem Desktop installiert hast, solltest du zuerst den VLC-Player inklusive FFmpeg entfernen:  
```
sudo apt purge vlc* && sudo apt purge ffmpeg*  
sudo apt autoremove  
```
und VLC-Symbol aus Startmenu entfernen:  
`sudo rm /usr/share/raspi-ui-overrides/applications/vlc.desktop`  

### die schnelle Variante mit static build von Johan van Sickle ###
Infos: https://www.johnvansickle.com/ffmpeg/faq/  
Achtung: FFplay fehlt (Stand 20.08.2021) und bei der folgenden Methode wird die man-page (Handbuchseite) nicht mit installiert.  

Beispiel:  
``` 
# als normaler User **pi** ausführen:  
cd ~  
mkdir ~/ffmpegsource/  
cd ~/ffmpegsource/  
wget https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-armhf-static.tar.xz  
tar xvf ffmpeg*.tar.xz  
cd ffmpeg*static  
sudo mv ffmpeg ffprobe /usr/local/bin/  
cd ~  
rm -r ffmpegsource/  
```

## Installation ##
**und jetzt ausführlich**

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
### Installation auf NanoPi Fire3, NanoPi Neo V1.4 ###
- Bei der Installation als root, kann `sudo` entfallen. (Gilt für jedes Gerät ;-)   
- Bei **FFmpeg ./configure** müssen RaspberryPi - spezifische Einstellungen weggelassen werden: `--enable-omx-rpi`  
- Um die Hardwarebeschleunigung in FFmpeg zu nutzen, müssen gerätespezifische Einstellungen vorgenommen werden.
- Für die NanoPi´s habe ich folgende Konfiguration, ohne Hardwarebeschleunigung verwendet: 
`sudo ./configure --extra-ldflags="-latomic" --arch=armel --target-os=linux --enable-gpl --enable-libmp3lame --enable-libfdk-aac --enable-libfreetype --enable-libx264 --enable-libx265 --enable-nonfree --enable-libsrt`   
Der NanoPi Fire3 hat acht CPU-Kerne und benötigt deshalb keine extra Hardwarebeschleunigung. Bei den RaspberryPi zero, 1, 2 und 3 macht die extra Beschleunigung jedoch Sinn. 
#### Hintergrundinfos zur Hardwarebeschleunigung auf dem RaspberryPi ####
Auch der Raspberry Pi bringt auf seinem SoC Hardware-Unterstützung für einige Algorithmen mit. Fehlen die, wird das Abspielen von Videos schwierig. Auch hier muss man FFmpeg selber übersetzen und die Optionen (»–enable-omx«, »–enable-omx-rpi«) aktivieren. Der Pi dekodiert auch einige Codecs in Hardware, braucht dafür bei Mpeg 2 aber eine Lizenz, die extra zu beschaffen ist. Beispielkonfiguration siehe Listing 1. Raspbian verlangte noch nach den Paketen »libmp3lame-dev« und »libx264-dev«.  
**Listing 1**  
Konfiguration für den Raspberry Pi  
```
01 ./configure --arch=armel \  
02             --target-os=linux \  
03             --enable-gpl \  
04             --enable-omx \  
05             --enable-omx-rpi \  
06             --enable-mmal \  
07             --enable-nonfree \  
08             --enable-libmp3lame \  
09             --enable-libx264  
```  
Der Aufruf zum Transkodieren eines Testvideos mit Hardwaresupport lautet dann z.B.:
`./ffmpeg  -i ~/Testvideo.MPG -c:v h264_omx  Testvideo.MKV`  
Willst du (bei vorhandener Lizenz) auch die Hardwaredekodierung verwenden, nutzt der folgenden Aufruf:
`./ffmpeg -c:v mpeg2_mmal -i ~/Testvideo.MPG-c:v h264_omx Testvideo.MKV`  
Hier muss manuell der richtigen Hardwaredekodierer passend zur vorliegenden Datei ausgewählt werden, wobei »h264_mmal« auch ohne Lizenz funktioniert.
mit #> `ffmpeg -hwaccels` werden diverse Varianten angezeigt.  
In unseren Test auf dem RaspberryPi 4 habe ich zwar die Optionen zur Hardware-Unterstützung mit aktiviert, bei der Streamanpassung von RTSP zu SRT aber nicht weiter beachtet. Der RaspberryPi 4 hat eine Quad-Core CPU. Damit ist keine Hardwareunterstützung beim Dekodieren mehr notwendig. Die Qualität der Softwaredekodierung soll besser sein, als die einer Dekodierung mit Hardwareunterstützung.  
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
## weitere Infos: ##  
- [SRT-Kochbuch: FFmpeg](https://srtlab.github.io/srt-cookbook/apps/ffmpeg/)
- [SRT-Kochbuch: OBS](https://srtlab.github.io/srt-cookbook/apps/obs-studio/)
