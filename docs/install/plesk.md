# Plesk

Plesk serves its control panel and every hosted site from its own nginx (and Apache) on **80, 443
and 8443**. The stock `compose.yaml` publishes 80/tcp, 443/tcp and 443/udp on the host, so on a
Plesk machine `docker compose up` dies at the first container with *port is already allocated* — or
binds a port Plesk is about to use and takes the panel or a customer's site down with it.

`compose.plesk.yaml` moves plMail off those ports. FrankenPHP serves **plain HTTP inside the
container**, one host port is published on **127.0.0.1**, and Plesk — which already holds the
certificate — proxies the domain to it. Nothing in the container issues a certificate, so the
first-open warnings a normal install shows do not exist here.

## Starting it

```bash
docker compose -f compose.yaml -f compose.plesk.yaml up -d --wait
```

Keep **both** `-f` flags on every later `up`, `pull` and `down`. Drop the second one and compose is
back to the stock file, which tries for 80 and 443 again.

Plesk's Docker extension can run the same two files (**Docker → Compose**, or paste
`compose.plesk.yaml` into its editor beside the base file) instead of the CLI. Either way the
result is the same stack.

It publishes `127.0.0.1:${PLESK_HTTP_PORT:-8080}`. Set `PLESK_HTTP_PORT` to move that host port. It
binds to `127.0.0.1` on purpose — only Plesk's proxy, on the same machine, has any business
reaching the app, and that is also what the extension's own manual port mapping does. If Plesk's
nginx runs in a container or on another host, edit the `host_ip:` line in the overlay and widen it.

## Pointing a domain at it

Give the domain a certificate in Plesk first (Let's Encrypt under **Websites & Domains → SSL/TLS
Certificates**); Plesk terminates TLS and forwards plain HTTP into the container.

Then either:

- **Docker Proxy Rules** — **Websites & Domains → the domain → Docker Proxy Rules → Add Rule**,
  with the URL `/`, the plMail `php` container, and the port mapping the overlay created
  (container `80` → host `8080`). Plesk's Docker extension, its own manual port mapping, and these
  rules are the documented route; see the Plesk Docker documentation.
- **A hand-written nginx directive** on the domain (**Apache & nginx Settings → Additional nginx
  directives**), which works whether or not Plesk's extension manages the containers:

  ```nginx
  location / {
      proxy_pass http://127.0.0.1:8080;
      proxy_set_header Host              $host;
      proxy_set_header X-Real-IP         $remote_addr;
      proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
  }
  ```

  `SERVER_NAME: ":80"` in the overlay makes Caddy answer plain HTTP for **whatever Host arrives**,
  so no hostname needs configuring on the container side.

## Telling the app who it is

The address plMail is reached at from outside is what it hands to Google and Microsoft as a
callback, and what the Mercure hub URL is derived from. It must be the **https** address Plesk
answers, not the container's own port:

```bash
APP_PUBLIC_URL=https://mail.example.com \
  docker compose -f compose.yaml -f compose.plesk.yaml up -d --wait
```

The overlay passes `APP_PUBLIC_URL` through to both the `php` and the `worker` service, so a value
set this way reaches the workers that build push subscriptions — which have no request to read a
hostname from. Leave it unset and the setup screen asks for it on first boot and stores it instead,
exactly as on any other install.

## The proxying details that matter

Two of plMail's responses are **long-lived and stream**, and Plesk's default proxy buffers and
times out both:

- The **Mercure stream** is Server-Sent Events; a buffering proxy holds the events, and the mail
  list stops updating by itself while the rest of the app works. Set `proxy_buffering off` and a
  long `proxy_read_timeout` for `/.well-known/mercure`.
- The **assistant's** summary and compose-assist endpoints say nothing while the model reads; a
  minute-long read timeout — nginx's default — closes them mid-answer.

See [Behind a reverse proxy](reverse-proxy.md#proxy-configuration-itself) for the full directive
set, which applies unchanged here.

**Check that `X-Forwarded-Proto` reaches the container.** Plesk's Docker extension has added it to
generated proxy rules since extension 1.4.3, but a hand-written rule that omits it makes Symfony
believe it is serving plain HTTP: OAuth redirect URIs come out `http://`, cookies lose their
`secure` flag, and share links get the wrong scheme. `TRUSTED_PROXIES` already trusts the private
range Plesk's nginx connects from, so no change is needed there.

## What the overlay changes

- `SERVER_NAME: ":80"` on `php`, so Caddy serves plain HTTP instead of redirecting to https.
- `php`'s published ports **replaced** (not added to) with one `127.0.0.1:8080 → 80/tcp` mapping.
  This uses Compose's `!override` tag because compose otherwise *concatenates* two files' `ports`,
  which would leave the stock file's 443 mappings in place. It needs Compose v2.24 or newer.
- `APP_ENV: prod` hardcoded, and `APP_PUBLIC_URL` passed through on `php` and `worker`.

## Things that bite

**Forgetting the second `-f`.** Compose silently falls back to the stock file and tries to take 80
and 443, which is the collision this page exists to avoid.

**Leaving `SERVER_NAME` at its default.** Caddy answers Plesk's plain-HTTP request with a redirect
to https and the browser loops between Plesk and the container.

**A hub stream that never updates.** Plesk's proxy buffering the `/.well-known/mercure` response,
almost always. The topbar's connection indicator is where it becomes visible.
