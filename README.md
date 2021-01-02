# rtsp2srt
Secure Reliable Transport (SRT) ist eine Open-Source-Transporttechnologie, die die Streaming-Leistung in Netzwerken wie z.B. dem Internet optimiert.
Wir wollen uns die RTSP-Streams der IP-Kameras auf den Raspberry Pi holen und dort mittels ffmpeg ins SRT-Protokoll umwandeln und dann zur StreamBox senden.
Dazu muss sowohl auf dem Raspberry Pi und dem großen Computer, also unserer StreamBox auf der OBS läuft, ffmpeg mit integriertem SRT-Protokoll installiert werden.

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
und VLC-Symbol aus Startmenu entfernen: `sudo rm /usr/share/raspi-ui-overrides/applications/vlc.desktop`  

Die Installation der Quellpakete für FFmpeg inklusive SRT setzt die folgenden Schritte voraus:  
- Bereitstellung aller notwendigen Tools und Bibliotheken, also Download der Abhängigkeiten (Dependencies)
- Konfiguration (mit einem Konfigurationsskript, z.B. `./configure`)  
- Zusammenstellung der Pakete/Bibliotheken/Binärdateien (`make`)  
- Installation (`sudo make install`)  

### Download und Installation der Abhängigkeiten ###
```
sudo apt update  
sudo apt install autoconf automake build-essential cmake pkg-config texinfo wget git yasm nasm tcl tclsh libtool libva-dev libass-dev libfreetype6-dev libgnutls28-dev libsdl2-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev zlib1g-dev libssl-dev libx264-dev libx265-dev libnuma-dev libx265-doc libvpx-dev libmp3lame-dev libopus-dev  
``` 
(Nicht alle der obigen Pakete werden für unser kleines Projekt benötigt. Ich weiß allerdings nicht, welche entbehrlich sind ;-)  
Beim Raspberry Pi sind z.B. das Audio-Paket libfdk-aac noch nicht mit `apt` installierbar und muss extra kompiliert werden.  
[Infos hier im FFmpeg CompilitionGuide](https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu "FFmpeg CompilitionGuide")  
```
cd /home/pi/ffmpeg_sources  
git -C fdk-aac pull 2> /dev/null || git clone --depth 1 https://github.com/mstorsjo/fdk-aac  
PATH="$HOME/bin:$PATH"  
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
sudo git clone https://github.com/Haivision/srt  
cd srt  
PATH="$HOME/bin:$PATH"
sudo ./configure  
sudo make  
sudo make install 
```  
### FFmpeg downloaden, konfigurieren (inkl. SRT-Bibliothek einbinden), kompilieren und installieren: ###
```
cd /home/pi/ffmpeg_sources  
sudo git clone https://github.com/FFmpeg/FFmpeg.git  
cd FFmpeg  
PATH="$HOME/bin:$PATH"  
# sudo export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig"  
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
```  

Aufruf auf dem Raspi mit z.B.:
```
ffmpeg -fflags nobuffer -i 'rtsp://admin:admin@192.168.95.52:554/1/h264major' -c copy -f mpegts 'srt://172.16.95.6:40052?mode=caller&transtype=live&latency=1000000'
```

### Integration im Skript GatewaySet.sh ###
**In der Konfigurationsdatei GatewaySet.conf können Anpassungen zur Optimierung vorgenommen werden:**

```
# StreamVariablen für rtsp2srt  
# Kamerabesonderheiten (müssten, bei der Verwendung verschiedener Kameras jeweils pro Kamera angepasst werden):
userandpassword='admin:admin'  
rtspcamspec=':554/1/h264major'  
# FFmpeg-Einstellungen; Quelle:
encodflags='-loglevel panic -stats -flags low_delay -fflags discardcorrupt'  
# FFmpeg-Einstellungen (SRT); Ziel:
decodflags='copy -f mpegts'  
srtflags='?mode=caller&transtype=live&latency=100000'  
# Die folgenden Variablen werden hier erstmals mit einem Wert belegt und    
# im Schell-Skript GatewaySet.sh zur Laufzeit u.a. in der Funktion start_rtsp2srt() überschrieben.
# Änderungen der Variablennamen müssten demzufolge jeweils auch im Quelltext GatewaySet.sh durchgeführt werden! 
# Deswegen: Finger hier weg!
streamsource=rtsp://${userandpassword}@${SourceCamIP}${rtspcamspec}
streamtargetport=400${SourceCamIP##*.}
streamtarget=srt://${StreamTargetIP}:${streamtargetport}
```

**Im Linux Shell Skript GatewaySet.sh wird folgende Funktion zum Start von FFmpeg verwendet:**
```
start_rtsp2srt() {  
    streamsource=rtsp://${userandpassword}@${SourceCamIP}${rtspcamspec}  
    streamtargetport=400${SourceCamIP##*.}  
    streamtarget=srt://${StreamTargetIP}:${streamtargetport}  
    echo "Ich starte FFmpeg mit folgendem String:"  
    flag_stream=1  
    echo ffmpeg $encodflags -i $streamsource -c $decodflags $streamtarget $srtflags  
    ffmpeg $encodflags -i $streamsource -c $decodflags $streamtarget$srtflags   
    # Achtung, innerhalb eines Shell-Scriptes sind Variablen in einer Funktion NICHT lokal gekapselt, sondern sind global im gesamten Skript verfügbar.   
}
```

## Installation auf der StreamBox (Ubuntu 20.04 LTS) ##

## Testen ##
### rtsp-Stream auf dem Raspi empfangen und als SRT-Stream senden ###
- `ffmpeg -fflags nobuffer -i 'rtsp://admin:admin@192.168.95.52:554/1/h264major' -c copy -f mpegts 'srt://172.16.95.6:40052?mode=caller&transtype=live&latency=1000000'`   

### auf der StreamBox ###
**Beispiele:**  

- `srt-live-transmit 'srt://192.168.95.6:40052?mode=listener&latency=1000' udp://localhost:50052`  
oder  
- `srt-live-transmit 'srt://172.16.95.6:40052?mode=listener&latency=1000' udp://localhost:50052`  
- in OBS eine Medienquelle einfügen mit der URL: udp://localhost:50052  

## Portadressierung ##

Die Raspberry Pi (192.168.95.41 bis .49) in den AkkuBoxen adressieren ihren Kamerastream an die StreamBox (192.168.95.6) über folgende Ports:  

|    Kamera-IP  | StreamBox Eingangsport vom Raspi | StreamBox localhost Port für OBS |  
| :-------------| ----------------------:| --------------------------------:|  
| 192.168.95.51 | 40051 | 50051 |  
| 192.168.95.52 | 40052 | 50052 |
|  -"- | -"- | -"- |
| 192.168.95.59 | 40059 | 50051 |
Die Ports haben einen "Basisteil" 400 oder 500 und einen "Kamerateil" 51 bis 59.

Die StreamBox hat die Adressen 192.168.95.6 im LAN und 172.16.95.6 im VPN. Sie kann über beide Adressen erreicht werden.
Die App "srt-live-transmit" horcht auf beiden Adressen auf allen möglichen Kameraports und leitete eingehende Streams auf die entsprechenden localhost-Ports für OBS weiter.
Die jeweiligen srt-live-stream Instanzen laufen entweder ständig oder werden nur gestartet, wenn die jeweiligen Kameras (per ping) erreichbar sind.

Beispiel für den Start zweier Instanzen für eine Kamera:
```
#!/usr/bin/bash  

quelle51LAN='srt://192.168.95.6:40052?mode=listener&latency=1000'  
quelle51VPN='srt://172.16.95.6:40052?mode=listener&latency=1000'  
ziel51='udp://localhost:50052'  

echo "Quellen51: " $quelle51LAN und $quelle51VPN  
echo "Ziel51:    " $ziel51  

exec /usr/local/bin/srt-live-transmit ${quelle51LAN} ${ziel51} &>/dev/null &  
exec /usr/local/bin/srt-live-transmit ${quelle51VPN} ${ziel51} &>/dev/null  
```  


