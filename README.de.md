# Intel-MIPI-Kamera unter Fedora: zwei unterschiedliche Fehler

Gerät: Dell Latitude 9430 mit Intel IPU6 und OV02C10. Untersuchung vom 29. September 2026. [Ausführliche Diagnose und Versionen auf Englisch](README.md).

## Nach Kernelupdate fehlt die Kamera

Unter Kernel `7.2.7-200.fc44.x86_64` wurde der Sensor erkannt. Der Dienst `v4l2-relayd@icamerasrc.service` startete aber, bevor `akmods` die benötigten Zusatzmodule fertiggestellt hatte. Nach mehreren Fehlversuchen blieb der Dienst gestoppt.

Zuerst prüfen, ob die Modulinstallation erfolgreich beendet wurde:

```bash
systemctl status akmods.service v4l2-relayd@icamerasrc.service --no-pager
modinfo -n intel_ipu6_psys
modinfo -n v4l2loopback
```

Wenn beide Module für den laufenden Kernel vorhanden sind und dieser Relay-Dienst bereits zur eigenen Kamerakonfiguration gehört, half hier:

```bash
sudo modprobe intel_ipu6_psys &&
sudo systemctl reset-failed v4l2-relayd@icamerasrc.service &&
sudo systemctl restart v4l2-relayd@icamerasrc.service
```

Danach funktionierten Kamera-App und Zoom mit **Intel MIPI Camera**. Eine dauerhafte Korrektur der Startreihenfolge ist damit noch nicht umgesetzt.

## Firefox flackert, Zoom funktioniert

In den Protokollen gab es einen libcamera-Absturz und PipeWire-Formatkonflikte. Für Firefox wurde folgender Workaround eingerichtet:

Die Testseite war https://de.webcamtests.com/ . Der Benutzer bestätigte nachträglich, dass Zoom und die Webseite kurz gleichzeitig geöffnet waren und dabei nur eine Anwendung auf die Kamera zugreifen konnte. Das passt zu den Meldungen „Gerät belegt“; diese allein belegen keinen Firefox-Fehler. Ob auch das Flackern und der Absturz damit zusammenhingen, ist nicht nachgewiesen.

1. Andere Kameravorschauen schließen.
2. In Firefox `about:config` öffnen.
3. **`media.webrtc.camera.allow-pipewire`** auf **`false`** setzen.
4. Firefox vollständig beenden und neu starten.
5. Auf der Webseite **Intel MIPI Camera** auswählen.

Wichtig: **Bindestrich** in `allow-pipewire`, kein Unterstrich. Auf dem untersuchten Gerät war zuvor der wirkungslose Schlüssel `allow_pipewire` gesetzt.

Die entsprechende Profilkonfiguration lautet:

```javascript
user_pref("media.webrtc.camera.allow-pipewire", false);
```

Der Benutzer bestätigte anschließend, dass alles funktioniert. Welche Fehlerkomponente das Flackern genau auslöste, wurde nicht durch einen kontrollierten Vergleich isoliert. Der Workaround umgeht einen Kameraweg; er repariert keinen Treiberquellcode.

Rücknahme: Den Override gegebenenfalls aus `user.js` entfernen und den korrekten Schlüssel in `about:config` zurücksetzen.

Der ältere Fehler unter Kernel 7.2.5, bei dem der Sensor überhaupt nicht eingebunden wurde, ist separat zu betrachten. Hierfür ist kein erneuter Wechsel auf einen alten Kernel erforderlich gewesen.

[Vorbereitete Fehlerberichte und Veröffentlichungsstatus](PUBLISHING.md).
