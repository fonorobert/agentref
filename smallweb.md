<!-- Combined from smallweb-website docs -->


<!-- File: getting-started.md -->
# Getting started

## Why Smallweb ?

Smallweb is a new take on self-hosting. The goal is to make it as easy as possible to host your own apps. To achieve this, it uses a "file-based hosting" approach: every subfolder in your smallweb folder becomes a unique subdomain.

Smallweb is obsessed with scaling down. No dev server or a build is required, and you don't even need to care about deployment. You can just write a single file, hit save, and voila! Your app is live.

## Live Demo

A shared public instance of smallweb is available at [smallweb.club](https://smallweb.club). You can use it to test smallweb without setting up your own instance.

Please ping me either on [bluesky](https://bsky.app/profile/pomdtr.me) or [discord](https://discord.gg/BsgQK42qZe) if the demo is down.

## Local Installation

First, you'll need to [install Deno](https://docs.deno.com/runtime/getting_started/installation/). I'll use the curl command here:

```sh
curl -fsSL https://deno.land/install.sh | sh
```

Then, install smallweb using one of the following methods:

```sh
# use the install script
curl -fsSL https://install.smallweb.run | sh

# brew (macOS/Linux)
brew install pomdtr/tap/smallweb

# scoop (Windows)
scoop bucket add pomdtr https://github.com/pomdtr/scoop-bucket.git
scoop install pomdtr/smallweb
```

Next, create your smallweb directory (I'll use `~/smallweb`), and initialize it:

```sh
mkdir ~/smallweb && cd ~/smallweb
smallweb init smallweb.live
```

> [!INFO]
> `smallweb.live` is a magic domain that will always route request to `127.0.0.1` (localhost). You can also use [traefik.me](https://traefik.me) or [local.gd](https://local.gd).

Now you can start smallweb:

```sh
smallweb up
```

And access your apps at `http://<app>.smallweb.live:7777`.

You'll have two apps available by default: `www` and `example`. You can access them at `http://www.smallweb.live:7777` and `http://example.smallweb.live:7777`.

If you want to be able to omit the port from the URL, you'll need to use the `:80` address:

```sh
# your app are now available at http://<app>.smallweb.live
smallweb up --addr :80
```

On linux, you'll have to allow smallweb to bind to port 80:

```sh
sudo setcap 'cap_net_bind_service=+ep' $(which smallweb)
```

### Generating tls certificates using mkcert

In order to get a proper TLS setup, you'll need to generate certificates for your domain.

The easiest way to do this is using [mkcert](https://github.com/FiloSottile/mkcert).

```sh
brew install mkcert
mkcert -install
mkcert -cert-file cert.pem -key-file key.pem smallweb.live '*.smallweb.live'
```

Then, you can use these certificates when starting smallweb:

```sh
smallweb up --addr :443 --tls-cert cert.pem --tls-key key.pem
```

Your apps will now be available at `https://<app>.smallweb.live`.

On linux, you'll have to allow smallweb to bind to port 443:

```sh
sudo setcap 'cap_net_bind_service=+ep' $(which smallweb)
```

### Using smallweb behind a reverse proxy

Another option is to use a reverse proxy like [caddy](https://caddyserver.com) as a reverse proxy on port 80 and 443.

```sh
brew install caddy && brew services start caddy
```

Here is an example `Caddyfile` routing `smallweb.live` to smallweb:

```txt
smallweb.live, *.smallweb.live {
  tls internal
  reverse_proxy localhost:7777
}
```

### Keep smallweb running in the background

To keep smallweb running in the background, you can use a process manager like [systemd](https://www.freedesktop.org/wiki/Software/systemd/) on Linux or [launchd](https://launchd.info/) on macOS.

#### Systemd

Here is an example systemd service file for smallweb.

```ini
[Unit]
Description=Smallweb
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/smallweb up
WorkingDirectory=/home/:user/smallweb
```

You should store this file as `~/.config/systemd/user/smallweb.service`.

Then, enable and start the service:

```sh
systemctl --user enable smallweb
systemctl --user start smallweb
```

#### Launchd

And here is an example launchd plist file for smallweb on macOS:

```xml
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>KeepAlive</key>
        <true />
        <key>Label</key>
        <string>com.github.pomdtr.smallweb</string>
        <key>LimitLoadToSessionType</key>
        <array>
            <string>Aqua</string>
            <string>Background</string>
            <string>LoginWindow</string>
            <string>StandardIO</string>
            <string>System</string>
        </array>
        <key>ProgramArguments</key>
        <array>
            <string>/opt/homebrew/bin/smallweb</string>
            <string>up</string>
        </array>
        <key>WorkingDirectory</key>
        <string>/Users/:user/smallweb</string>
        <key>RunAtLoad</key>
        <true />
        <key>StandardErrorPath</key>
        <string>/Users/pomdtr/Library/Logs/smallweb.log</string>
        <key>StandardOutPath</key>
        <string>/Users/pomdtr/Library/Logs/smallweb.log</string>
    </dict>
</plist>
```

You should store this file as `~/Library/LaunchAgents/com.github.pomdtr.smallweb.plist`.

Then, load the service:

```sh
launchctl load ~/Library/LaunchAgents/com.github.pomdtr.smallweb.plist
```


<!-- File: guides/authentication.md -->
# Authentication with OpenID Connect

## Using a public OIDC provider

You can instruct smallweb to authenticate users using OpenID connect. The easiest to get started is to use [Lastlogin](https://lastlogin.net), a public OIDC provider that do not require a registration process.

Firt, set the `oidc.issuer` field in your smallweb config:

```json
// $SMALLWEB_DIR/.smallweb/config.json
{
    "domain": "example.com",
    "oidc": {
        "issuer": "https://lastlogin.net"
    },
    "authorizedEmails": [
        "your-email@example.com"
    ]
}
```

Then, set the `private` field to `true` in your app manifest:

```json
// $SMALLWEB_DIR/demo/smallweb.json
{
    "private": true
}
```

The next time you'll access `demo.example.com`, you'll need to confirm your email address using lastlogin. Once you do that, you'll be redirected to your app.

You can access the email of the current user using the `Remote-Email` header:

```ts
// $SMALLWEB_DIR/demo/main.ts
export default {
    fetch(req: Request) {
        const email = req.headers.get("Remote-Email");
        return new Response(`Hello, ${email}`, {
            headers: {
                "Content-Type": "text/plain",
            },
        });
    }
}
```

If you want to only protect specific routes, set the `privateRoutes` and `publicRoutes` array in your app manifest. Smallweb uses [doublestar](https://github.com/bmatcuk/doublestar) to check request paths against the global patterns

## Hosting your own OIDC provider from smallweb

Nothing stops you from using an internal url as the issuer !

```json
{
    "domain": "example.com",
    "oidc": {
        "issuer": "https://auth.example.com"
    },
    "authorizedEmails": [
        "your-email@example.com"
    ]
}
```

In this case, the `auth` app should exposes an OIDC provider. I recommend using [openauth](https://github.com/toolbeam/openauth).

> [!WARNING]
> Openauth does not support beeing used as an OIDC provider yet, so I've built [a wrapper](https://jsr.io/@pomdtr/openauth-oidc) to add support for it.
> Make sure to follow [this issue](https://github.com/toolbeam/openauth/issues/134) for official support.

```ts
// ~/smallweb/auth/main.ts

import { issuer } from "npm:@openauthjs/openauth";
import { oidc } from "jsr:@pomdtr/openauth-oidc";
import { MemoryStorage } from "npm:@openauthjs/openauth/storage/memory";

const storage = new MemoryStorage({
    persist: "./data/db.json",
})

const iss = issuer({
    storage,
    // ...
})

export default oidc(iss, storage);
```

Openauth support a bunch of issuers (Google, Github, etc), refer to [their documentation](https://openauth.js.org/docs/) for more information.

Smallweb use the app origin as the `client_id`, and no `client_secret`. You can read more about the logic behind these choices at <https://lastlogin.net/developers/>.


<!-- File: guides/commands.md -->
# CLI Commands

To add a cli command to your app, you'll need to add a `run` method to your app's default export.

```ts
// File: ~/smallweb/custom-command/main.ts
export default {
    run(args: string[]) {
        console.log("Hello world");
    }
}
```

Use `smallweb run` to execute the command:

```console
$ smallweb run custom-command
Hello world
```

## Using a cli framework

I personally recommend using [commander.js](https://www.npmjs.com/package/commander) to build complex cli commands.

```ts
import { Command } from 'npm:@commander-js/extra-typings';

export default {
    async run(args: string[]) {
        const program = new Command();

        program
            .option('-n, --name <name>', 'name to greet', 'world')
            .action((options) => {
                console.log(`Hello ${options.name}!`);
            });

        await program.parseAsync(args, { from: 'user' });
    }
}
```


<!-- File: guides/cron.md -->
# Cron Jobs

Cron jobs are a way to schedule jobs to run at specific times.

You can define cron jobs in your app manifest file, which is located in the app's directory.

```json
// ~/smallweb/hello/smallweb.json
{
  "crons": [
    {
      "schedule": "0 0 * * *",
      "args": ["pomdtr"]
    }
  ]
}
```

This cron job will trigger the [cli entrypoint](./commands.md) of the `hello` app every day at midnight, with the argument `pomdtr`.

```ts
// ~/smallweb/hello/main.ts
export default {
  run: async (args: string[]) => {
    console.log(`Hello ${args[0]}`);
  },
};
```

You can trigger it manually by just using the `smallweb run hello pomdtr` command.


<!-- File: guides/custom-commands.md -->
# Custom Commands

The smallweb CLI can be extended with custom commands. To create a new command, just create a script in `$SMALLWEB_DIR/.smallweb/commands`. The script name (without the extension) will be mapped to a cli subcommand.

Ex: if i create the following file as `$SMALLWEB_DIR/.smallweb/commands/edit.sh`:

```sh
#!/bin/sh

if [ -z "$1" ]; then
    echo "Usage: smallweb edit <app>"
    exit 1
fi

code "$SMALLWEB_DIR/$1"
```

I will be able to run `smallweb edit my-app` to open the `my-app` folder in VSCode.

## Environment variables available to plugins

- `SMALLWEB_VERSION`: The version of the smallweb CLI.
- `SMALLWEB_DIR`: The directory where the smallweb apps are stored.
- `SMALLWEB_DOMAIN`: The domain where the smallweb apps are served from.


<!-- File: guides/data.md -->
# Storing Data

Each smallweb app has write access to a `data` directory at the root of the app dir. This is a good place to store data that your app needs to persist between requests.

## Storing Files

 You can use the `Deno` standard library to read and write files in the `data` directory.

## Using a json file as a database

The [lowdb](https://github.com/typicode/lowdb) library is a good choice for small apps. It allows you to use a json file as a database.

```ts
import { JSONFilePreset } from "npm:lowdb/node"

const db = await JSONFilePreset('data/db.json', { posts: [] })

// read existing posts
console.log(db.data.posts)

// add new post
const post = { id: 1, title: 'lowdb is awesome', views: 100 }

// In two steps
db.data.posts.push(post)
await db.write()

// Or in one
await db.update(({ posts }) => posts.push(post))
```

## Sqlite

If you need a more robust database, you can use the `node:sqlite` package.

```ts
import { DatabaseSync } from 'node:sqlite';

// Open a database
const db = new DatabaseSync('data/test.db');

// Create table
db.exec(`
  CREATE TABLE IF NOT EXISTS people (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT
  ) STRICT
`);

// Insert data
const insert = db.prepare('INSERT INTO people (name) VALUES (?)');
for (const name of ["Peter Parker", "Clark Kent", "Bruce Wayne"]) {
  insert.run(name);
}

// Query and print data
const query = db.prepare('SELECT name FROM people');
const results = query.all();
for (const row of results) {
  console.log(row.name);
}
```

## Using an external database

Of course, you can also use an external service to store your data. I personally recommend [Turso](https://turso.tech/), which has a generous free tier.


<!-- File: guides/email.md -->
# Receiving Emails

Each app has an unique email address (`<app>@<domain>`). You can hande incoming emails by declaring an `email` method:

```ts
import PostalMime from "npm:postal-mime";

export default {
    email: async (msg: ReadableStream) => {
        const email = await PostalMime.parse(msg);

        console.log('Subject:', email.subject);
        console.log('HTML:', email.html);
        console.log('Text:', email.text);
    }
}
```

The email method receive a readable stream as it's first argument. You can use any library to parse the email content, I recommend [postal-mime](https://www.npmjs.com/package/postal-mime).

Support for sending emails is not implemented yet.

> [!WARNING]
> Smallweb does not check SPF or DKIM records yet, meaning that you cannot not trust the sender address. This will be fixed in a future release.

## Setup

You'll need to set an MX record to start receiving emails (assuming you have already set up the wildcard DNS record for your smallweb instance):

```txt
@  IN  MX  10 mail.smallweb.run
```

Then, specify the smtp address when you start smallweb:

```sh
smallweb up --smtp-addr :25
```


<!-- File: guides/env.md -->
# Env Variables

## Environment variables

You can set environment variables for your app by creating a file called `.env` in the application folder.

Here is an example of a `.env` file:

```txt
BEARER_TOKEN=SECURE_TOKEN
```

Use the `Deno.env.get` method to access the environment variables in your app:

```ts
// File: ~/smallweb/demo/main.ts
export default {
  fetch(req: Request) {
    return new Response(`Hello, ${Deno.env.get("BEARER_TOKEN")}`, {
      headers: {
        "Content-Type": "text/plain",
      },
    });
  },
}
```

## Injected environment variables

Smallweb automatically injects the following environment variables into your app:

- `SMALLWEB_VERSION`: The version of the smallweb CLI.
- `SMALLWEB_DIR`: The directory where the smallweb apps are stored.

In addition to these, the `SMALLWEB_ADMIN` environment variable is also set for admin apps.

## Encrypted Secrets

> [!WARNING] This feature is recommended only for advanced users.

Smallweb delegates encryption to [SOPS](https://github.com/getsops/sops). Encrypted secrets are stored in a `secrets.enc.env` file at the root of your app dir (you can also use `secrets.env`).

Smallweb will automatically decrypt the file using your private ssh key (`~/.ssh/id_ed25519` by default) and inject the secrets as environment variables into your app at runtime.

> [!NOTE] This is a opinionated guide, sops support other encryption methods.
> Check the [SOPS documentation](https://github.com/getsops/sops) for more information.

On MacOS/Linux, you can install SOPS using Homebrew:

```sh
brew install sops # make sure your sops version is 3.10.0 or higher
```

Then we'll create a `.sops.yaml` file at the root of our smallweb dir:

```yaml
# ~/smallweb/.sops.yaml
creation_rules:
  - key_groups:
      - age:
          - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEPR1N/c+B7OPXEraFx2r3UHViHFbZ2Afg8VQLQ59ZKd # your ssh public key
```

From now on, we can generate/edit a `secrets.enc.env` for your app using the following command:

```sh
# edit <app> secrets
sops ./<app>/secrets.enc.env
```

> [!WARNING] Make sure to never edit the encrypted file directly!

If you add a new public key to your `.sops.yaml` file, you'll need to update the keys in your encrypted files:

```sh
# run this command at the root of your smallweb dir
sops updatekeys */secrets.enc.env
```


<!-- File: guides/file-sync.md -->
# Syncing files using mutagen

I recommend using [mutagen](https://mutagen.io) to sync your files between your development machine and your remote server.

## Installation

First, install the required tools on your development machine using Homebrew:

```sh
brew install mutagen-io/mutagen/mutagen
```

# Start the syncing process

```sh
mkdir ~/smallweb && cd ~/smallweb
```

Then, create the `~/smallweb/.mutagen.yml` file with the following content:

```yaml
sync:
  defaults:
    mode: two-way-resolved
    ignore:
      paths:
        - node_modules
        - .DS_Store
  example:
    alpha: <remote>:<remote-path>
    beta: <local-path>
```

For example, to sync the directories storing every apps running under the `smallweb.run` and `pomdtr.me` domains, I use:

```yaml
sync:
  defaults:
    mode: "two-way-resolved"
    ignore:
      paths:
        - node_modules
        - .DS_Store
  pomdtr-me:
    alpha: vps:/home/pomdtr/smallweb/pomdtr.me
    beta: ./pomdtr.me
  smallweb-run:
    alpha: vps:/home/pomdtr/smallweb/smallweb.run
    beta: ./smallweb.run
```

Then, to start the sync session, run the following command:

```sh
mutagen project start
```

From now on, each time you make a change to your files, they will be automatically synced to the server, and vice-versa.

## Resume the sync session on reboot

To ensure that the sync session resumes automatically after a reboot, you can use the following command:

```sh
mutagen daemon register
```

## Diagnosing sync issues

If you encounter any issues with the sync, you can check the errors using the following command:

```sh
mutagen project list --long
```


<!-- File: guides/http.md -->
---
outline: [2, 3]
---

# HTTP Requests

Smallweb maps every subdomains of your root domain to a directory in your root directory.

For example with, the following configuration:

```json
// ~/smallweb/.smallweb/config.json
{
    "domain": "example.com"
}
```

The routing system maps domains to directories as follows:

- Direct subdomain mapping:
  - `api.example.com` → `~/smallweb/api`
  - `blog.example.com` → `~/smallweb/blog`

- Root domain behavior:
  - Requests to `example.com` automatically redirect to `www.example.com` if the `www` directory exists
  - If the `www` directory does not exist, a 404 error is returned

> [!WARNING]
> Subdomains must be alphanumeric, and can contain hyphens. You should also avoid using uppercase letters in your subdomains, as they are usually converted to lowercase. Underscores are allowed, but not recommended.

> [!NOTE]
> Any folder starting with `.` is ignored by the routing system. You can use it to your advantage to create hidden directories that are not accessible from the web (ex: `.github`, or `.data`).

Smallweb detects the type of website you are trying to host based on the files in the directory.

- If the directory contains a `main.[js,ts,jsx,tsx]` file, it is considered a [dynamic website](#dynamic-websites).
- Else, it is considered a [static website](#static-websites).
  - if the directory contains `dist/index.html` or an `index.html` file, it is served at the root of the website.
  - if the directory contains an `index.md` file, it is rendered as html.

You can opt-out of the default behavior by creating a `smallweb.json` file at the root of the directory, and specifying the `root` and `entrypoint` fields. See [Manifest](/docs/reference/manifest.md) for more information.

## Static Websites

You can create a website by just adding an `index.html` file in the folder.

```html
<!-- File: ~/smallweb/example-static/index.html -->
<!DOCTYPE html>
<html>
  <head>
    <title>Example Static Website</title>
  </head>
  <body>
    <h1>Hello, world!</h1>
  </body>
</html>
```

To access the website, open `https://example-static.<smallweb-domain>` in your browser.

If your static website contains a `main.js` file, but you want to serve it as a static website, yoy can create a `smallweb.json` file at the root of the directory, and specify the `entrypoint` field to `jsr:@smallweb/file-server`.

  ```json
  {
    "entrypoint": "jsr:@smallweb/file-server"
  }
  ```

### Rendering markdown

The smallweb static server automatically renders markdown files as html. If you create a file named index.md in a folder (and no `index.html`), it will be served at the root of the website.

### Single Page Applications

To serve a single page application, you need to redirect all requests to the `index.html` file. You can do this by a `_redirects` file at the root of the static directory.

```txt
/* /index.html 200
```

If you're using a tool like [Vite](https://vitejs.dev), this `_redirects` file should be added to the `public` directory.

## Dynamic Websites

Smallweb can also host dynamic websites. To create a dynamic website, you need to create a folder with a `main.[js,ts,jsx,tsx]` file in it.

The file should export a default object with a `fetch` method that takes a `Request` object as argument, and returns a `Response` object.

```ts
// File: ~/smallweb/example-server/main.ts

export default {
  fetch(request: Request) {
    const url = new URL(request.url);
    const name = url.searchParams.get("name") || "world";

    return new Response(`Hello, ${name}!`, {
      headers: {
        "Content-Type": "text/plain",
      },
    });
  },
}
```

To access the server, open `https://example-server.<smallweb-domain>` in your browser.

### Routing Requests using Hono

Smallweb use the [deno](https://deno.com) runtime to evaluate the server code. You get typescript and jsx support out of the box, and you can import any module from the npm and jsr registry by prefixing the module name with `npm:` or `jsr:`.

As an example, the following code snippet use the [hono package](https://hono.dev) to extract params from the request url.

```jsx
// File: ~/smallweb/hono-example/main.ts

import { Hono } from "jsr:@hono/hono";

const app = new Hono();

app.get("/", c => c.text("Hello, world!"));

app.get("/:name", c => c.text(`Hello, ${c.params.name}!`));

// Hono instances have a `fetch`, so they can be used as the default export
export default app;
```

To access the server, open `https://hono-example.<smallweb-domain>` in your browser.


<!-- File: guides/integrations.md -->
# Integrations

Smallweb can easily be integrated to other tools.

Since the whole state of your smallweb instance is stored in a single directory, it is quite easy to scan it to find the existing apps and domains.

## Raycast

A smallweb extension is available [on the Raycast store](https://www.raycast.com/pomdtr/smallweb).

![raycast extension screenshot](./img/raycast_extension.png)

## VS Code

A smallweb extension is available [on the VS Code marketplace](https://marketplace.visualstudio.com/items?itemName=pomdtr.smallweb).

![vscode extension screenshot](./img/vscode_extension.png)


<!-- File: guides/opentelemetry.md -->
# OpenTelemetry

Smallweb supports OpenTelemetry for monitoring and tracing. To activate it, just set the `OTEL_DENO` environment variable to `true` before starting the server.

```sh
OTEL_DENO=true smallweb up
```

By default, it will send traces, metrics and logs to `http://localhost:4317`. You can wire it to  self-hosted grafana instance with the following docker-command:

```sh
docker run --name lgtm -p 3000:3000 -p 4317:4317 -p 4318:4318 --rm -ti \
    -v "$PWD"/lgtm/grafana:/data/grafana \
    -v "$PWD"/lgtm/prometheus:/data/prometheus \
    -v "$PWD"/lgtm/loki:/data/loki \
    -e GF_PATHS_DATA=/data/grafana \
    docker.io/grafana/otel-lgtm:0.8.1
```

You can use a bunch of `OTEL_` prefixed env variables to configure opentelemetry. If you want to use grafana cloud (they have a generous free tier), you can follow [this guide](https://grafana.com/docs/grafana-cloud/send-data/otlp/send-data-otlp/#manual-opentelemetry-setup-for-advanced-users).

Checkout the [Deno Docs](https://docs.deno.com/runtime/fundamentals/open_telemetry) to learn more about opentelemetry support in deno.


<!-- File: guides/permissions.md -->
# Permissions

Smallweb apps have access to:

- read access to their app folder, and the deno cache folder.
- write access to the `data` subfolder in app folder
- access to the network, to make HTTP requests.
- access to the env files defined in the `.env` files from the smallweb root folder and the app folder.

This sandbox protects the host system from malicious code, and ensures that apps can only access the resources they need.

## Admin Apps

If you want to create an app that can access the whole smallweb directory, you can set the `admin` property to `true` in the the app manifest.

```json
// ~/smallweb/app-name/smallweb.json
{
    "admin": true
}
```

Admin apps have read/write access to the whole smallweb dir.

```ts
const { SMALLWEB_ADMIN, SMALLWEB_DIR } = Deno.env.toObject();

if (!SMALLWEB_ADMIN) {
    throw new Error("This app is not an admin app");
}

const apps = await Deno.readDir(SMALLWEB_DIR).filter((dir) => dir.isDirectory && !strings.startsWith(dir.name, "."));
```


<!-- File: guides/ssh.md -->
# SSH Server

Smallweb includes a built-in SSH server. In order to use it, you must provide the `--ssh-addr` flag when starting smallweb:

```sh
smallweb up --ssh-addr :2222
```

## Authentication

To access the SSH server, you'll need to add your public key in the `~/.smallweb/config.json` file:

```json
{
  "domain": "example.com",
  "authorizedKeys": [
    "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC7..."
  ]
}
```

You can also map a public key to a specific app:

```json
{
  "domain": "example.com",
  "apps": {
    "smallblog": {
      "authorizedKeys": [
        "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC7..."
      ]
    }
  }
}
```

### Usage

The ssh server use the username to route commands to the correct app:

- `_` is the username for the smallweb cli (and maps to the `$SMALLWEB_DIR` when using `sftp`)
- `<app-name>` is the username for an app (and maps to the `$SMALLWEB_DIR/<app-name>` when using `sftp`)

```sh
# list smallweb apps using the smallweb cli
ssh -p 2222 _@example.com ls

# run the smallblog app cli
ssh -p 2222 smallblog@example.com ls
```

You can interact with your smallweb dir using the whole range of ssh tools:

- accessing the smallweb cli: `ssh -p 2222 _@example.com`
- accessing an app cli: `ssh -p 2222 <app-name>@example.com`
- editing your smallweb dir using [lftp](https://lftp.yar.ru/): `lftp sftp://_:@example.com:2222`
- mounting an app dir using ssfs:

  ```sh
  mkdir ~/my-smallweb-app && sshfs -p 2222 <app-name>@example.com:/ ~/my-smallweb-app
  ```

It's a good idea to add an alias to your `~/.ssh/config` file:

```sh
Host example.com
  Port 2222
  User _
```

Then you can just run `ssh example.com` to access the smallweb cli.


<!-- File: guides/symlinks.md -->
# Symlinks

Symbolic links have multiple uses in smallweb.

## Sharing files between apps

In smallweb, apps can only access files in their own directory. However, you can use symlinks to share files between apps.

If you to access the `users.json` file from `app1` in `app2`, you can create a symbolic link in the `app2` folder:

```sh
smallweb link app1/users.json app2/users.json
```

You can also use this mechanism to share directories between apps.

## Map multiple subomains to the same app

In smallweb, subdomains are mapped to directories. If you want to map multiple subdomains to the same app, you just need multiple subdirectories with symlinks to the main app directory.

```sh
# Make gh.<domain> point to the github app
smallweb link github gh
```

Of course, you can also use the `customDomains` field in the global config file to achieve the same result, but using symlinks make them visible from the filesystem.


<!-- File: hosting/cloudflare/index.md -->
# Cloudflare Tunnel

Cloudflare Tunnel is a **free** service that allows you to expose your local server to the internet, without having to expose your local IP address.

Additionally, it provides some protection against DDoS attacks, and allows you to use Cloudflare's other services like Access.

## Setup

1. Make sure that you have a domain name that you can manage with Cloudflare.

1. From your cloudflare dashboard, navigate to `Zero Trust > Networks > Tunnels`

1. Click on `Create a tunnel`, and select the `Clouflared` option

1. Follow the intructions to install cloudflared, and create a connector on your device.

1. Add a hostname for your apex domain (ex: `example.com`), and use `http://localhost:7777` as the origin service.

    ![Tunnel Configuration](./tunnel.png)

1. Do the same for your wildcard domain (ex: `*.example.com`). You'll need to then go to the DNS configuration of your domain, and add a `CNAME` record for your wildcard hostname, with a target of `<tunnel-id>.cfargotunnel.com`.

    ![DNS Configuration](./dns.png)

1. Add a new `CNAME` record for your wildcard hostname, with a target of `<tunnel-id>.cfargotunnel.com`.

  ![DNS Configuration](./dns.png)



<!-- File: hosting/local/index.md -->
# Hosting smallweb locally

This section will guide you through the process of setting up a local instance of smallweb.

This setup is useful for developing and testing smallweb apps locally, without having to deploy them to the internet.

If you want to expose your apps to the internet instead, you can follow the [Cloudflare Tunnel setup guide](../cloudflare/index.md).

## Architecture

The following diagram illustrates the architecture of the local setup:

![Localhost architecture](./architecture.excalidraw.png)

The components needed are:

- a dns server to map `localhost` domains to `127.0.0.1` ip address (only needed on macOS)
- a reverse proxy to generate and renew SSL certificates for the domains, and forward requests to the smallweb server
- smallweb itself


<!-- File: hosting/local/linux.md -->
# Ubuntu / Debian setup

## Install Deno

```sh
curl -fsSL https://deno.land/install.sh | sh
# add ~/.deno/bin to PATH
echo "export PATH=\$PATH:\$HOME/.deno/bin" >> ~/.bashrc
```

## Setup Smallweb

```sh
curl -fsSL https://install.smallweb.run | sh
echo "export PATH=\$PATH:\$HOME/.local/bin" >> ~/.bashrc
```

## Setup Caddy

```sh
sudo apt-get update
sudo apt-get install -y caddy

# Write caddy configuration
cat <<EOF | sudo tee /etc/caddy/Caddyfile
smallweb.localhost, *.smallweb.localhost {
  tls internal

  reverse_proxy localhost:7777
}
EOF

sudo systemctl restart caddy
caddy trust
```

There is no need to setup dnsmasq on Ubuntu, as it seems to be already configured to resolve `.smallweb.localhost` domains to `127.0.0.1`.

## Testing the setup

First, let's create a dummy smallweb website:

```sh
mkdir -p ~/smallweb/.smallweb
cat <<EOF > ~/smallweb/.smallweb/config.json
{
  "domain": "smallweb.localhost"
}
EOF

mkdir -p ~/smallweb/example
cat <<EOF > ~/smallweb/example/main.ts
export default {
  fetch() {
    return new Response("Smallweb is running");
  }
}
EOF

cd ~/smallweb && smallweb up
```

If everything went well, you should be able to access `https://example.localhost` in your browser, and see the message `Smallweb is running`.

## Start smallweb at login

```sh
cat <<EOF > ~/.config/systemd/user/smallweb.service
[Unit]
Description=Smallweb
After=network.target

[Service]
Type=simple
ExecStart=$HOME/.local/bin/smallweb up --enable-crons
Restart=always
RestartSec=10
Environment="SMALLWEB_DIR=$HOME/smallweb"

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable smallweb
systemctl --user start smallweb
```


<!-- File: hosting/local/macos.md -->
# MacOS setup

In the future, we might provide a script to automate this process, but for now, it's a manual process.

## Install Brew

```sh
# install homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Install Deno

```sh
brew install deno
```

## Setup Smallweb

```sh
brew install pomdtr/tap/smallweb
```

## Setup Caddy

Caddy’s configuration path depends on whether you're using an Intel-based Mac or an Apple Silicon (M1/M2) Mac.

- **For Apple Silicon Macs**:

  ```sh
  brew install caddy

  # Write caddy configuration
  cat <<EOF > /opt/homebrew/etc/Caddyfile
  smallweb.localhost, *.smallweb.localhost {
    tls internal

    reverse_proxy localhost:7777
  }
  EOF
  ```

- **For Intel-based Macs**:

  ```sh
  brew install caddy

  # Write caddy configuration
  cat <<EOF > /usr/local/etc/Caddyfile
  smallweb.localhost, *.smallweb.localhost {
    tls internal

    reverse_proxy localhost:7777
  }
  EOF
  ```

### Run Caddy

```sh
# Run caddy in the background
brew services start caddy

# Add caddy https certificates to your keychain
caddy trust
```

## Setup dnsmasq

The configuration path for dnsmasq also depends on your Mac's architecture.

- **For Apple Silicon Macs**:

  ```sh
  brew install dnsmasq

  # Write dnsmasq configuration
  echo "address=/.localhost/127.0.0.1" >> /opt/homebrew/etc/dnsmasq.conf
  ```

- **For Intel-based Macs**:

  ```sh
  brew install dnsmasq

  # Write dnsmasq configuration
  echo "address=/.localhost/127.0.0.1" >> /usr/local/etc/dnsmasq.conf
  ```

## Run dnsmasq

```sh
# Run dnsmasq in the background
sudo brew services start dnsmasq

# Indicates to the system to use dnsmasq for .localhost domains
sudo mkdir -p /etc/resolver
cat <<EOF | sudo tee -a /etc/resolver/localhost
nameserver 127.0.0.1
EOF
```

## Testing the setup {#testing-the-setup-macos}

First, let's create a dummy smallweb website:

```sh
mkdir -p ~/smallweb/.smallweb
cat <<EOF > ~/smallweb/.smallweb/config.json
{
  "domain": "smallweb.localhost"
}
EOF

mkdir -p ~/smallweb/example
cat <<EOF > ~/smallweb/example/main.ts
export default {
  fetch() {
    return new Response("Smallweb is running");
  }
}
EOF

# Start smallweb
cd ~/smallweb && smallweb up
```

If everything went well, you should be able to access `https://example.smallweb.localhost` in your browser, and see the message `Smallweb is running`.

## Start smallweb at login

```sh
cat <<EOF > ~/Library/LaunchAgents/com.github.pomdtr.smallweb.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.github.pomdtr.smallweb</string>
        <key>ProgramArguments</key>
        <array>
            <string>$(brew --prefix)/bin/smallweb</string>
            <string>up</string>
            <string>--enable-crons</string>
            <string>--dir=$HOME/smallweb</string>
        </array>
        <key>RunAtLoad</key>
        <true />
        <key>StandardOutPath</key>
        <string>$HOME/Library/Logs/smallweb.log</string>
        <key>StandardErrorPath</key>
        <string>$HOME/Library/Logs/smallweb.log</string>
        <key>WorkingDirectory</key>
        <string>$HOME/</string>
    </dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.github.pomdtr.smallweb.plist
```

Each time you update smallweb, you'll need to restart the service. The easiest way to do this is to unload and load the service:

```sh
launchctl unload ~/Library/LaunchAgents/com.github.pomdtr.smallweb.plist
launchctl load ~/Library/LaunchAgents/com.github.pomdtr.smallweb.plist
```


<!-- File: hosting/local/windows.md -->
# Windows (WSL)

## Install WSL

Follow the instructions on the [official documentation](https://learn.microsoft.com/en-us/windows/wsl/install).

The next commands in this guide should be run in the WSL terminal.

## Install Deno

```sh
curl -fsSL https://deno.land/install.sh | sh
# add ~/.deno/bin to PATH
echo "export PATH=\$PATH:\$HOME/.deno/bin" >> ~/.bashrc
```

## Setup Smallweb

```sh
curl -fsSL https://install.smallweb.run | sh
# add ~/.local/bin to PATH
echo "export PATH=\$PATH:\$HOME/.local/bin" >> ~/.bashrc
```

## Setup Caddy

```sh
sudo apt-get update
sudo apt-get install -y caddy

# Write caddy configuration
cat <<EOF | sudo tee /etc/caddy/Caddyfile
smallweb.localhost, *.smallweb.localhost {
  tls internal

  reverse_proxy localhost:7777
}
EOF

sudo systemctl restart caddy
caddy trust
```

There is no need to setup dnsmasq on Windows, as it seems to be already configured to resolve `.smallweb.localhost` domains to `127.0.0.1`.

## Testing the setup

First, let's create a dummy smallweb website:

```sh
mkdir -p ~/smallweb/.smallweb
cat <<EOF > ~/smallweb/.smallweb/config.json
{
  "domain": "smallweb.localhost"
}
EOF

mkdir -p ~/smallweb/example
cat <<EOF > ~/smallweb/example/main.ts
export default {
  fetch() {
    return new Response("Smallweb is running");
  }
}
EOF
```

If everything went well, you should be able to access `https://example.smallweb.localhost` in your browser, and see the message `Smallweb is running`.

## Start smallweb at login

```sh
cat <<EOF > ~/.config/systemd/user/smallweb.service
[Unit]
Description=Smallweb
After=network.target

[Service]
Type=simple
ExecStart=$HOME/.local/bin/smallweb up --enable-crons
Restart=always
RestartSec=10
Environment="SMALLWEB_DIR=$HOME/smallweb"

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable smallweb
systemctl --user start smallweb
```


<!-- File: hosting/runtipi.md -->
# Runtipi

Smallweb is available on the [Runtipi App Store](https://runtipi.io/docs/apps-available)!

Runtipi lets you install all your favorite self-hosted apps without the hassle of configuring and managing each service. Once you have it setup, you can install smallweb with a single click.

## Installation

Refer to the [runtipi documentation](https://runtipi.io/docs/getting-started/installation) for instructions.


<!-- File: hosting/vps.md -->
---
outline: [2, 3]
---

# VPS

This page will guide you through the process of hosting smallweb on a fresh new VPS running debian 12.

Any VPS provider will work, but if you're looking for a recommendation, I had a good experience with [Hetzner](https://www.hetzner.com/cloud).

Of course, there are an infinite way to hook up smallweb to your own setup, so feel free to adapt the guide to your own needs.

## Guide

In the following guide, I will assume you own the domain `example.com` and want to host smallweb on it.

You can replace `example.com` with your own domain.

### Setup DNS Records

First, you need to set up your DNS records to point to your VPS on your domain registrar.

```txt
example.com. 3600 IN A <your-ipv4>
*.example.com. 3600 IN A <your-ipv4>
example.com. 3600 IN AAAA <your-ipv6>
*.example.com. 3600 IN AAAA <your-ipv6>
example.com. 3600 IN MX 10 mail.example.com.
```

You can find your IPv4 and IPv6 addresses by running the following command on your VPS:

```sh
# IPv4
curl -4 https://icanhazip.com

# IPv6
curl -6 https://icanhazip.com
```

If you do not own a domain yet, you can use a free [sslip.io](https://sslip.io/) domain. Ex: `37.27.85.244` -> `37-27-85-244.sslip.io`.

### Install Docker

```sh
apt update && apt install -y curl
curl -fsSL https://get.docker.com | sh
```

### Setup the `smallweb` user

```sh
# Create a system user for smallweb
useradd --user-group --create-home --shell "$(which bash)" --uid 1000 smallweb

# Create a SSH key for the smallweb user
mkdir -p /home/smallweb/.ssh
cp /root/.ssh/authorized_keys /home/smallweb/.ssh/authorized_keys
ssh-keygen -t ed25519 -N "" -f /home/smallweb/.ssh/id_ed25519
chown -R smallweb:smallweb /home/smallweb/.ssh
```

### Setup Compose project

Create the following directory structure:

```yaml
# /opt/docker/smallweb/compose.yaml
services:
  smallweb:
    image: ghcr.io/pomdtr/smallweb:latest
    restart: unless-stopped
    command: up --enable-crons --ssh-addr :2222 --smtp-addr :25 --ssh-private-key /run/secrets/ssh_private_key --on-demand-tls
    secrets:
      - ssh_private_key
    ports:
      - "80:80"
      - "443:443"
      - "2222:2222"
      - "25:25"
    volumes:
      - ./data:/smallweb
      - deno_cache:/home/smallweb/.cache/deno
      - certmagic_cache:/home/smallweb/.cache/certmagic

secrets:
  ssh_private_key:
    file: "/home/smallweb/.ssh/id_ed25519"

volumes:
  deno_cache:
  certmagic_cache:
```

Then init the smallweb workspace and start the service:

```sh
cd /opt/docker/smallweb

mkdir data && chown smallweb:smallweb data
docker compose run smallweb init --domain "example.com"
docker compose up -d
```

### Edit your local ssh config

Add the following to your `~/.ssh/config` file:

```txt
Host example.com
  User _
  Port 2222
  RequestTTY yes
  LogLevel ERROR
```

Now you can use `ssh example.com` to access the smallweb cli, and `ssh <app>@example.com` to access an app cli.

### Sync your smallweb dir locally

First, install [Mutagen](https://mutagen.io/documentation/introduction/installation/). Then, run the following command to sync your local smallweb directory with the remote smallweb directory:

```sh
# run mutagen daemon on login
mutagen daemon register

# This command should be run on your local machine
mutagen sync create \
  --name=smallweb \
  --ignore=node_modules,.DS_Store \
  --ignore-vcs \
  --mode=two-way-resolved \
  smallweb@<vps-ip>:smallweb ~/smallweb
```


<!-- File: reference/cli.md -->
# CLI Reference

## smallweb

Host websites from your internet folder

```
smallweb [flags]
```

### Options

```
      --dir string      The root directory for smallweb
      --domain string   The domain for smallweb
  -h, --help            help for smallweb
```

## smallweb config

Open Smallweb configuration

```
smallweb config [flags]
```

### Options

```
  -h, --help   help for config
      --json   Output the configuration in JSON format
```

### Options inherited from parent commands

```
      --dir string      The root directory for smallweb
      --domain string   The domain for smallweb
```

## smallweb crons

List cron jobs

```
smallweb crons [app] [flags]
```

### Options

```
  -a, --app string   filter by app name
  -h, --help         help for crons
      --json         output as json
```

### Options inherited from parent commands

```
      --dir string      The root directory for smallweb
      --domain string   The domain for smallweb
```

## smallweb doctor

Check the system for potential problems

```
smallweb doctor [flags]
```

### Options

```
  -h, --help   help for doctor
```

### Options inherited from parent commands

```
      --dir string      The root directory for smallweb
      --domain string   The domain for smallweb
```

## smallweb init

Initialize a new workspace

```
smallweb init <domain> [flags]
```

### Options

```
  -h, --help   help for init
```

### Options inherited from parent commands

```
      --dir string      The root directory for smallweb
      --domain string   The domain for smallweb
```

## smallweb list

List all smallweb apps

```
smallweb list [flags]
```

### Options

```
      --admin   filter by admin
  -h, --help    help for list
      --json    output as json
```

### Options inherited from parent commands

```
      --dir string      The root directory for smallweb
      --domain string   The domain for smallweb
```

## smallweb run

Run an app cli

```
smallweb run <app> [args...] [flags]
```

### Options

```
  -h, --help   help for run
```

### Options inherited from parent commands

```
      --dir string      The root directory for smallweb
      --domain string   The domain for smallweb
```

## smallweb up

Start the smallweb evaluation server

```
smallweb up [flags]
```

### Options

```
      --addr string              address to listen on
      --enable-crons             enable cron jobs
  -h, --help                     help for up
      --log-format string        log format (json, text or pretty)
      --log-output string        log output (stdout, stderr or filepath) (default "stderr")
      --on-demand-tls            enable on-demand tls
      --smtp-addr string         address to listen on for smtp
      --ssh-addr string          address to listen on for ssh/sftp
      --ssh-host-key string      ssh host key
      --ssh-private-key string   ssh private key
      --tls-cert string          tls certificate file
      --tls-key string           tls key file
```

### Options inherited from parent commands

```
      --dir string      The root directory for smallweb
      --domain string   The domain for smallweb
```



<!-- markdownlint-disable-file -->


<!-- File: reference/config.md -->
---
outline: [2, 3]
---

# Global Config

The smallweb config is located at `$SMALLWEB_DIR/.smallweb/config.json`.

If `SMALLWEB_DIR` is not set, it defaults to `~/smallweb`.

Only the `domain` field is required. The rest are optional.

## `domain`

The `domain` field defines the apex domain used for routing. By default, it is `localhost`.

```json
{
  // apps will be served at `<app>.example.com`
  "domain": "example.com"
}
```

The smallweb domain can be overriden by using the `--domain` flag when running the smallweb CLI.

```sh
smallweb up --domain smallweb.localhost
```

## `additionalDomains`

Additional domains that should be routed to the same smallweb instance.

```json
{
  "domain": "example.com",
  "additionalDomains": [
    // in addition to example.com, apps will be served at `<app>.example.org`
    "example.org",
  ]
}
```

## `authorizedKeys`

List of public ssh keys that are allowed to:

- access the smallweb cli
- access the smallweb sftp server
- access every app `run` entrypoint

```json
{
  "domain": "example.com",
  "authorizedKeys": [
    "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC7Z... user@host"
  ]
}
```

## `authorizedEmails`

List of emails that are allowed to access all private apps.

```json
{
  "domain": "example.com",
  "authorizedEmails": [
    "achille.lacoin@gmail.com"
  ]
}
```

## `authorizedGroups`

List of groups that are allowed to access all private apps.

```json
{
  "domain": "example.com",
  "authorizedGroups": [
    "admin"
  ]
}
```

## apps section

This section is used to set app specific config values.

### `apps.<app>.admin`

Give admin permissions to an app. Admin apps have read/write access to the whole smallweb dir (except the special `.smallweb` dir).

```json
{
  "domain": "example.com",
  "apps": {
    "vscode": {
      "admin": true
    }
  }
}
```

### `apps.<app>.authorizedEmails`

List of emails that are allowed to access a specific app.

```json
{
  "domain": "example.com",
  "apps": {
    "vscode": {
      "private": true,
      "authorizedEmails": [
        "achille.lacoin@gmail.com"
      ]
    }
  }
}
```

### `apps.<app>.authorizedGroups`

List of groups that are allowed to access all private apps.

```json
{
  "domain": "example.com",
  "authorizedGroups": [
    "admin"
  ],
  "apps": {
    "vscode": {
      "private": true,
      "authorizedGroups": [
        "dev"
      ]
    }
  }
}
```

### `apps.<app>.additionalDomains`

Additional domains that should be routed to the app.

```json
{
  "domain": "example.com",
  "apps": {
    "vscode": {
      "additionalDomains": [
        // in addition to vscode.example.com, the app will be served at vscode.me
        "vscode.me"
      ]
    }
  }
}
```

### `apps.<app>.authorizedKeys`

List of public ssh keys that are allowed to:

- the app `run` entrypoint
- the app directory using sftp

```json
{
  "domain": "example.com",
  "apps": {
    "vscode": {
      "authorizedKeys": [
        "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC7Z... user@host"
      ]
    }
  }
}
```

## `oidc` section

### `oidc.issuer`

The issuer of the OIDC provider. This is used to verify if a user is allowed to access a private app.

```json
{
  "domain": "example.com",
  "oidc": {
    "issuer": "https://lastlogin.net"
  }
}
```


<!-- File: reference/editor.md -->
# Editor Support

## VS Code

You can get completions for the app and global config files by adding the following snippet to your user settings:

```json
{
  "json.schemas": [
    {
      "url": "https://github.com/pomdtr/smallweb/releases/latest/download/manifest.schema.json",
      "fileMatch": [
        "smallweb.json",
        "smallweb.jsonc"
      ]
    },
    {
      "url": "https://github.com/pomdtr/smallweb/releases/latest/download/config.schema.json",
      "fileMatch": [
        ".smallweb/config.json",
        ".smallweb/config.jsonc"
      ]
    }
  ]
}
```

## AI Apps

Smallweb provides both `llms.txt` and `llms-full.txt` file that you can provide to your favourite AI model.

- <https://www.smallweb.run/llms.txt>
- <https://www.smallweb.run/llms-full.txt>


<!-- File: reference/manifest.md -->
# App Manifest

The smallweb manifest can be defined in a `smallweb.json[c]` file at the root of the project. This file is optional, and sensible defaults are used when it is not present.

## Available Fields

### `entrypoint`

The `entrypoint` field defines the file to serve. If this field is not provided, the app will look for a `main.[js,ts,jsx,tsx]` file in the root directory. If none is found, it will serves the directory statically.

### `root`

The `root` field defines the root directory of the app. If this field is not provided, the app will use the app directory as the root directory.

### `crons`

The `crons` field defines the crons to run. The crons are defined in a JSON array, and each cron is an object with the following fields:

- `args`: The arguments to pass to the cron
- `schedule`: The schedule to run the cron. This is a [cron expression](https://crontab.guru/) that defines when the cron should run.
- `description` (optional): A description of the cron. This is used for documentation purposes only.

```json
{
  "crons": [
    {
      "args": ["arg1", "arg2"],
      "schedule": "0 * * * *",
      "description": "This cron runs every hour"
    }
  ]
}
```

### `private`

Make an app private. Private apps are only accessible to users with the `authorizedEmails` or `authorizedGroups` set in the config.

```json
{
  "private": true
}
```

### `privateRoutes`

List of routes that are private. Private routes are only accessible to users with the `authorizedEmails` or `authorizedGroups` set in the config.

```json
{
  "privateRoutes": [
    "/private/**"
  ]
}
```

### `publicRoutes`

List of routes that are public. Public routes are accessible to everyone, even when the app itself is private.

```json
{
  "private": true,
  "publicRoutes": [
    "/public/**"
  ]
}
```
