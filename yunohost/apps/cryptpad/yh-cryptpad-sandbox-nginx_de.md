# CryptPad auf YunoHost: Sandbox-Domain Fix

**Version:** Mai 2026  
**YunoHost:** 12.1.39 (stable)  
**CryptPad:** 2026.2.2 ~ynh1  
**Symptome:** Zertifikat-Renewal schlägt fehl, Dokumente laden nicht (endloses "Loading"), Backups schlagen fehl

---

## Inhaltsverzeichnis

1. [Problembeschreibung](#problembeschreibung)
2. [Ursache](#ursache)
3. [Anleitung für User](#anleitung-für-user)
4. [Technische Erklärung für Entwickler](#technische-erklärung-für-entwickler)

---

## Problembeschreibung

Nach der Installation von CryptPad auf YunoHost treten folgende Probleme auf:

### Symptom 1: Zertifikat-Renewal schlägt fehl

Beim manuellen oder automatischen Erneuern des Let's Encrypt-Zertifikats erscheint folgender Fehler:

```
Verifying sandbox.cryptpad.beispiel.de...
Wrote file to /var/www/.well-known/acme-challenge-public/..., 
but couldn't download http://sandbox.cryptpad.beispiel.de/.well-known/acme-challenge/...: 
Error: Response Code: None
Certificate renewing for cryptpad.beispiel.de failed!
```

Oder mit SSL-Fehler:

```
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: 
unable to get local issuer certificate
```

### Symptom 2: Dokumente laden nicht

In der CryptPad-Oberfläche erscheint beim Öffnen von Dokumenten (Tabellen, Präsentationen, Code-Editor, Umfragen etc.) ein endloser Ladebalken. Alle Dokumenttypen hängen bei "Loading".

### Symptom 3: Backups schlagen fehl

YunoHost-Backups für die CryptPad-App schlagen fehl, weil CryptPad-Dienste nicht vollständig verfügbar sind.

---

## Ursache

CryptPad benötigt zwingend eine Sandbox-Domain (`sandbox.cryptpad.beispiel.de`), über die alle Dokumente in einem isolierten iframe geladen werden. Diese Isolation ist ein Sicherheitsmerkmal von CryptPad.

Das YunoHost-Installationspaket erstellt zwar eine nginx-Template-Datei für diese Sandbox-Domain (`nginx-sandbox.conf`), deployed sie jedoch **nicht automatisch** als aktive nginx-Konfiguration. Dadurch:

- Hat `sandbox.cryptpad.beispiel.de` **keinen HTTPS-Server-Block** → Dokumente können nicht geladen werden
- Hat `sandbox.cryptpad.beispiel.de` **keinen Port-80-Server-Block** mit ACME-Challenge → Let's Encrypt kann die Domain nicht verifizieren → Zertifikat-Renewal schlägt fehl

---

## Anleitung für User

### Voraussetzungen

- Root-Zugang zum Server (SSH)
- CryptPad ist auf YunoHost installiert
- DNS-Eintrag für `sandbox.cryptpad.beispiel.de` existiert (CNAME auf Hauptdomain)

---

### Schritt 1: DNS-Eintrag prüfen

Im DNS-Verwaltungspanel sicherstellen, dass folgender Eintrag existiert:

| Host | Typ | Ziel |
|------|-----|------|
| `sandbox.cryptpad` | `CNAME` | `cryptpad.beispiel.de` |

> **Wichtig:** Ein CNAME-Eintrag (kein A-Record!) ist korrekt und reicht aus, da die IP-Adresse dann automatisch vom Hauptdomain-Eintrag übernommen wird.

DNS-Propagation prüfen – abwarten bis eine IP-Adresse zurückgegeben wird:

```bash
dig sandbox.cryptpad.beispiel.de
```

Erwartete Ausgabe (unter `ANSWER SECTION`):

```
sandbox.cryptpad.beispiel.de.  300  IN  A  123.456.789.0
```

Wenn dort eine IP-Adresse steht, ist der DNS-Eintrag aktiv.

---

### Schritt 2: Eigene Variablen ermitteln

Die folgenden Werte werden für die Konfiguration benötigt. Sie aus der YunoHost-App-Konfiguration auslesen:

```bash
cat /etc/yunohost/apps/cryptpad/settings.yml | grep -E "domain|port"
```

Beispiel-Ausgabe:

```
domain: cryptpad.beispiel.de
port: 3000
port_socket: 3004
```

Diese drei Werte notieren – sie werden in den nächsten Schritten benötigt.

---

### Schritt 3: nginx-Konfiguration für Sandbox erstellen

Die App-interne Template-Datei mit den echten Werten befüllen und als aktive nginx-Konfiguration speichern:

```bash
sed -e 's/__DOMAIN__/cryptpad.beispiel.de/g' \
    -e 's/__APP__/cryptpad/g' \
    -e 's/__PORT__/3000/g' \
    -e 's/__PORT_SOCKET__/3004/g' \
    /etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf \
    > /etc/nginx/conf.d/sandbox.cryptpad.beispiel.de.conf
```

> **Hinweis:** `cryptpad.beispiel.de`, `3000` und `3004` durch die eigenen Werte aus Schritt 2 ersetzen, falls sie abweichen.

---

### Schritt 4: nginx-Konfiguration testen und neu laden

Syntax der nginx-Konfiguration prüfen:

```bash
nginx -t
```

Erwartete Ausgabe:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

> Die Warnungen bezüglich `ssl_stapling` und `variables_hash_bucket_size` können ignoriert werden – sie beeinflussen die Funktion nicht.

nginx neu laden:

```bash
systemctl reload nginx
```

---

### Schritt 5: HTTPS-Erreichbarkeit der Sandbox testen

```bash
curl -I https://sandbox.cryptpad.beispiel.de
```

Erwartete Ausgabe (Auszug):

```
HTTP/2 200
server: nginx
content-type: text/html; charset=UTF-8
```

Ein `HTTP/2 200` bestätigt, dass die Sandbox-Domain korrekt erreichbar ist.

---

### Schritt 6: Zertifikat erneuern

Da das Zertifikat noch nicht abgelaufen ist, muss die Erneuerung erzwungen werden:

```bash
yunohost domain cert renew cryptpad.beispiel.de --force
```

Erwartete Ausgabe am Ende:

```
Success! Let's Encrypt certificate renewed for the domain 'cryptpad.beispiel.de'
```

---

### Schritt 7: Hook erstellen – Konfiguration bei YunoHost-Updates erhalten

YunoHost überschreibt bei Updates und `regen-conf`-Aufrufen die nginx-Konfiguration und würde die in Schritt 3 erstellte Datei löschen. Um das zu verhindern, einen Hook erstellen der die Datei nach jedem `regen-conf` automatisch neu erstellt:

```bash
nano /usr/share/yunohost/hooks/conf_regen/99-nginx_sandbox_cryptpad
```

Folgenden Inhalt einfügen:

```bash
#!/usr/bin/env bash
set -e

do_pre_regen() { true; }
do_init_regen() { true; }

do_post_regen() {
    sed -e 's/__DOMAIN__/cryptpad.beispiel.de/g' \
        -e 's/__APP__/cryptpad/g' \
        -e 's/__PORT__/3000/g' \
        -e 's/__PORT_SOCKET__/3004/g' \
        /etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf \
        > /etc/nginx/conf.d/sandbox.cryptpad.beispiel.de.conf
    nginx -t
}

"do_$1_regen" "$(echo "${*:2}" | xargs)"
```

> **Wichtig:** `cryptpad.beispiel.de`, `3000` und `3004` durch die eigenen Werte ersetzen.  
> **Hinweis:** Der Hook prüft die nginx-Konfiguration mit `nginx -t`, lädt nginx aber bewusst nicht neu – das übernimmt YunoHost nach dem `regen-conf` selbst.

Speichern: `Ctrl+O` → `Enter` → `Ctrl+X`

Hook ausführbar machen:

```bash
chmod +x /usr/share/yunohost/hooks/conf_regen/99-nginx_sandbox_cryptpad
```

Hook testen:

```bash
yunohost tools regen-conf nginx
curl -I https://sandbox.cryptpad.beispiel.de
```

Wenn `HTTP/2 200` zurückkommt, ist der Hook korrekt konfiguriert. ✅

Die Meldung `expected to be deleted but was kept back` ist dabei normal und kann ignoriert werden.

---

### Schritt 8: Systemd-Service erstellen – Konfiguration nach jedem Reboot wiederherstellen

Der Hook läuft nur bei `regen-conf`-Aufrufen, nicht beim Systemstart. Nach einem Reboot fehlt die nginx-Konfiguration für die Sandbox-Domain und CryptPad funktioniert nicht. Um das zu verhindern, einen systemd-Service erstellen der die Konfiguration beim Booten automatisch wiederherstellt, bevor nginx startet:

```bash
nano /etc/systemd/system/cryptpad-sandbox-nginx.service
```

Folgenden Inhalt einfügen:

```ini
[Unit]
Description=Restore CryptPad sandbox nginx config
Before=nginx.service
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/share/yunohost/hooks/conf_regen/99-nginx_sandbox_cryptpad post nginx
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Speichern: `Ctrl+O` → `Enter` → `Ctrl+X`

Service aktivieren und starten:

```bash
systemctl daemon-reload
systemctl enable cryptpad-sandbox-nginx
systemctl start cryptpad-sandbox-nginx
```

Service-Status prüfen:

```bash
systemctl status cryptpad-sandbox-nginx
```

Erwartete Ausgabe:

```
Active: active (exited)
```

`active (exited)` ist korrekt – der Service läuft einmalig beim Booten, erledigt seine Aufgabe und beendet sich dann.

---

### Schritt 9: Reboot-Test

```bash
reboot
```

Nach dem Neustart prüfen:

```bash
systemctl status cryptpad-sandbox-nginx
curl -I https://sandbox.cryptpad.beispiel.de
```

Wenn `HTTP/2 200` zurückkommt, ist alles dauerhaft korrekt konfiguriert. ✅

---

### Schritt 10: CryptPad testen

Im Browser `https://cryptpad.beispiel.de` öffnen und folgendes prüfen:

- [ ] Startseite lädt korrekt
- [ ] Neues Dokument erstellen (Tabelle, Präsentation, Code) – kein endloses Loading mehr
- [ ] Backup über YunoHost-Weboberfläche ausführen

---

## Technische Erklärung für Entwickler

### Hintergrund

CryptPad verwendet eine Sandbox-Domain als Sicherheitsmechanismus. Alle Dokument-Editoren werden in einem `<iframe>` geladen, dessen Origin die Sandbox-Domain ist. Dadurch ist der Zugriff auf Cookies, LocalStorage und andere Ressourcen der Hauptdomain von innerhalb des Editors unmöglich – ein gezielter XSS-Angriff kann keine Nutzerdaten der Hauptdomain stehlen.

### Das Problem im YunoHost-Paket

Das YunoHost-Paket für CryptPad enthält die Datei:

```
/etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf
```

Diese enthält Platzhalter (`__DOMAIN__`, `__APP__`, `__PORT__`, `__PORT_SOCKET__`) und definiert sowohl einen Port-80-Block (mit ACME-Challenge) als auch einen Port-443-Block (HTTPS-Proxy zur CryptPad-Instanz).

**Das Problem:** Diese Template-Datei wird beim Installationsprozess nicht in `/etc/nginx/conf.d/` deployed. Das Installationsskript ruft zwar `ynh_add_nginx_config` auf, aber offenbar nur für die Hauptdomain-Konfiguration – nicht für `nginx-sandbox.conf`.

### Konsequenzen

1. **Kein Port-80-Block für `sandbox.*`:** Let's Encrypt schreibt die ACME-Challenge-Datei nach `/var/www/.well-known/acme-challenge-public/`, versucht sie dann über HTTP von `sandbox.cryptpad.beispiel.de` abzurufen – aber nginx hat keinen Server-Block für diese Domain auf Port 80 und leitet stattdessen auf HTTPS um. Da das HTTPS-Zertifikat die Sandbox-Domain noch nicht enthält, schlägt die SSL-Verbindung fehl (`CERTIFICATE_VERIFY_FAILED`), und Let's Encrypt erhält `Response Code: None`.

2. **Kein Port-443-Block für `sandbox.*`:** CryptPad lädt alle Editoren über `https://sandbox.cryptpad.beispiel.de`. Ohne nginx-Konfiguration für diese Domain auf Port 443 gibt es keine Verbindung → endloses Loading in der UI.

### Workaround

Der Workaround besteht aus drei Teilen:

**Teil 1:** Manuelles Deployen der Template-Datei (einmalig):

```bash
sed -e 's/__DOMAIN__/cryptpad.beispiel.de/g' \
    -e 's/__APP__/cryptpad/g' \
    -e 's/__PORT__/3000/g' \
    -e 's/__PORT_SOCKET__/3004/g' \
    /etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf \
    > /etc/nginx/conf.d/sandbox.cryptpad.beispiel.de.conf
```

**Teil 2:** Ein `conf_regen`-Hook unter `/usr/share/yunohost/hooks/conf_regen/99-nginx_sandbox_cryptpad` stellt sicher, dass die Datei nach jedem `yunohost tools regen-conf nginx`-Aufruf automatisch neu erstellt wird. Der Hook verwendet bewusst nur `nginx -t` ohne `systemctl reload nginx`, da YunoHost den Reload nach dem `regen-conf` selbst übernimmt.

**Teil 3:** Ein systemd-Service unter `/etc/systemd/system/cryptpad-sandbox-nginx.service` stellt sicher, dass die Konfiguration nach jedem Reboot wiederhergestellt wird, bevor nginx startet (`Before=nginx.service`). Der Service ruft denselben Hook auf und beendet sich nach erfolgreicher Ausführung (`Type=oneshot`).

### Empfehlung an Paket-Maintainer

Das Installationsskript der CryptPad-App sollte `nginx-sandbox.conf` analog zur Hauptdomain-Konfiguration deployen. Konkret sollte im `install`-Script nach dem `ynh_add_nginx_config`-Aufruf ein weiterer Aufruf für die Sandbox-Konfiguration erfolgen, oder die Sandbox-Domain als vollwertige YunoHost-Domain registriert werden, damit `regen-conf` sie automatisch verwaltet.

Alternativ könnte ein `conf_regen`-Hook direkt im App-Paket mitgeliefert werden.

**Relevante Dateien im Paket:**
- `/etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf` – Template vorhanden, wird aber nicht deployed
- `/etc/yunohost/apps/cryptpad/scripts/install` – Installationsskript, hier fehlt das Deployen der Sandbox-Config
