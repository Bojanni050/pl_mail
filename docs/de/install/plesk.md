# Plesk

Plesk bedient sein Control Panel und jede gehostete Website aus seinem eigenen nginx (und Apache) auf
**80, 443 und 8443**. Die Standard-`compose.yaml` veröffentlicht 80/tcp, 443/tcp und 443/udp auf dem
Host, auf einer Plesk-Maschine stirbt `docker compose up` also schon am ersten Container mit
*port is already allocated* — oder belegt einen Port, den Plesk gleich braucht, und reißt das Panel
oder die Website einer Kundin mit.

`compose.plesk.yaml` holt plMail von diesen Ports herunter. FrankenPHP liefert **einfaches HTTP im
Container**, ein Host-Port wird auf **127.0.0.1** veröffentlicht, und Plesk — das das Zertifikat
ohnehin besitzt — leitet die Domain dorthin weiter. Nichts im Container stellt ein Zertifikat aus,
die Warnungen beim ersten Öffnen, die eine normale Installation zeigt, gibt es hier also nicht.

## Starten

```bash
docker compose -f compose.yaml -f compose.plesk.yaml up -d --wait
```

Behalte **beide** `-f`-Flags bei jedem späteren `up`, `pull` und `down`. Lass das zweite weg, und
Compose fällt wieder auf die Standard-Datei zurück, die erneut 80 und 443 belegen will.

Die Docker-Erweiterung von Plesk kann dieselben zwei Dateien ausführen (**Docker → Compose**, oder
`compose.plesk.yaml` dort im Editor neben die Basis-Datei legen) statt der CLI. Das Ergebnis ist in
beiden Fällen derselbe Stack.

Veröffentlicht wird `127.0.0.1:${PLESK_HTTP_PORT:-8080}`. Setz `PLESK_HTTP_PORT`, um diesen Host-Port
zu verschieben. Die Bindung an `127.0.0.1` ist Absicht — nur Plesks Proxy auf derselben Maschine hat
irgendein Geschäft damit, die Anwendung zu erreichen, und genau das tut auch die manuelle
Port-Zuordnung der Erweiterung selbst. Läuft Plesks nginx in einem Container oder auf einem anderen
Host, ändere die `host_ip:`-Zeile im Overlay und mach sie weiter.

## Eine Domain darauf zeigen lassen

Gib der Domain zuerst ein Zertifikat in Plesk (**Websites & Domains → SSL/TLS-Zertifikate**); Plesk
beendet TLS und leitet einfaches HTTP in den Container weiter.

Dann entweder:

- **Docker Proxy Rules** — **Websites & Domains → die Domain → Docker Proxy Rules → Add Rule**, mit
  URL `/`, dem plMail-Container `php` und der Port-Zuordnung, die das Overlay angelegt hat
  (Container `80` → Host `8080`). Das ist der dokumentierte Weg der Plesk-Docker-Erweiterung.
- **Eine handgeschriebene nginx-Direktive** auf der Domain (**Apache & nginx Settings → Additional
  nginx directives**), die funktioniert, ob die Erweiterung die Container verwaltet oder nicht:

  ```nginx
  location / {
      proxy_pass http://127.0.0.1:8080;
      proxy_set_header Host              $host;
      proxy_set_header X-Real-IP         $remote_addr;
      proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
  }
  ```

  `SERVER_NAME: ":80"` im Overlay lässt Caddy für **jeden ankommenden Host** einfaches HTTP
  beantworten, es muss also kein Hostname auf der Containerseite konfiguriert werden.

## Der Anwendung sagen, wer sie ist

Die Adresse, unter der plMail von außen erreicht wird, ist die, die es Google und Microsoft als
Callback nennt, und aus der die Mercure-Hub-URL abgeleitet wird. Sie muss die **https**-Adresse sein,
die Plesk beantwortet, nicht der eigene Port des Containers:

```bash
APP_PUBLIC_URL=https://mail.example.com \
  docker compose -f compose.yaml -f compose.plesk.yaml up -d --wait
```

Das Overlay reicht `APP_PUBLIC_URL` an den `php`- **und** den `worker`-Dienst durch, ein so
gesetzter Wert erreicht also auch die Worker, die Push-Abonnements aufbauen — und die haben keinen
Request, aus dem sie einen Hostnamen lesen könnten. Lass es ungesetzt, und der Einrichtungsbildschirm
fragt beim ersten Start danach und speichert sie, genau wie bei jeder anderen Installation.

## Die Proxy-Details, auf die es ankommt

Zwei Antworten von plMail sind **langlebig und streamend**, und Plesks Standard-Proxy puffert und
beendet beide:

- Der **Mercure-Stream** sind Server-Sent Events; ein puffernder Proxy hält die Ereignisse
  zurück, und die Mailliste aktualisiert sich nicht mehr von selbst, während der Rest der Anwendung
  funktioniert. Setz `proxy_buffering off` und ein langes `proxy_read_timeout` für
  `/.well-known/mercure`.
- Die **Assistent**-Endpunkte für Zusammenfassung und Verfassen sagen nichts, während das Modell
  liest; ein Lesetimeout von einer Minute — nginx' Voreinstellung — bricht sie mitten in der Antwort
  ab.

Siehe [Hinter einem Reverse-Proxy](reverse-proxy.md#die-proxy-konfiguration-selbst) für den vollen Satz
an Direktiven, der hier unverändert gilt.

**Prüf, dass `X-Forwarded-Proto` den Container erreicht.** Die Docker-Erweiterung von Plesk fügt es
seit Version 1.4.3 zu erzeugten Proxy-Regeln hinzu, aber eine handgeschriebene Regel ohne es lässt
Symfony glauben, es liefere einfaches HTTP: OAuth-Redirect-URIs kommen als `http://` heraus, Cookies
verlieren ihr `secure`-Flag, und Freigabelinks bekommen das falsche Schema. `TRUSTED_PROXIES`
vertraut dem privaten Bereich, aus dem Plesks nginx sich verbindet, schon, hier ist also nichts zu
ändern.

## Was das Overlay ändert

- `SERVER_NAME: ":80"` auf `php`, damit Caddy einfaches HTTP liefert statt auf https umzuleiten.
- Die veröffentlichten Ports von `php` werden **ersetzt** (nicht ergänzt) durch genau eine Zuordnung
  `127.0.0.1:8080 → 80/tcp`. Das nutzt Compose' `!override`-Tag, weil Compose die `ports` zweier
  Dateien sonst *aneinanderhängt*, was die 443-Zuordnungen der Standard-Datei stehen ließe. Es
  braucht Compose v2.24 oder neuer.
- `APP_ENV: prod` fest verdrahtet, und `APP_PUBLIC_URL` auf `php` und `worker` durchgereicht.

## Was beißt

**Das zweite `-f` vergessen.** Compose fällt still auf die Standard-Datei zurück und will 80 und 443
belegen — genau die Kollision, gegen die diese Seite existiert.

**`SERVER_NAME` auf dem Vorgabewert lassen.** Caddy beantwortet Plesks einfache-HTTP-Anfrage mit einer
Weiterleitung auf https, und der Browser pendelt zwischen Plesk und dem Container hin und her.

**Ein Hub-Stream, der sich nie aktualisiert.** Fast immer puffert Plesks Proxy die Antwort auf
`/.well-known/mercure`. Sichtbar wird das am Verbindungsindikator in der oberen Leiste.
