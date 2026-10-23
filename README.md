# dokku-tailscale

Make apps available to your tailnet.

## Install

```sh
dokku plugin:install https://github.com/andrew-womeldorf/dokku-tailscale.git tailscale
```

## Prepare

### Storing the auth key

Store your Tailscale auth key (or OAuth client secret) with
`tailscale:set-key`. The key is read from **stdin** — never pass it as an
argument, so it stays out of process lists:

```sh
# from a file (delete the file afterwards)
ssh dokku@host tailscale:set-key < key.txt

# or interactively: paste the key at the prompt (it is not echoed)
ssh -t dokku@host tailscale:set-key
```

Avoid typing the key into a command line (for example
`echo tskey-... | ssh ...`): anything typed on the command line lands in your
shell history.

The key must start with `tskey-`. If you use a custom control server
(`TS_LOGIN_SERVER` is set, e.g. Headscale), any single-line key without spaces
is accepted. It is written atomically to
`/var/lib/dokku/data/tailscale/.authkey` (`${DOKKU_LIB_ROOT}/data/tailscale`)
with mode `600`, readable only by the `dokku` user, and is never printed.

Check where the key comes from (the key itself is never shown):

```sh
dokku tailscale:key-status
```

The key is passed only to the `ts-<app>` sidecar container (as `TS_AUTHKEY`,
handed to docker through the environment rather than on its command line),
not to your app containers. Anyone with root/docker access on the host can
still see it with `docker inspect ts-<app>`. A possible future hardening is to
mount the key file read-only into the sidecar and use Tailscale's
`TS_AUTHKEY=file:<path>` form instead.

#### Migrating from `config:set --global TS_AUTHKEY`

Earlier versions told you to put the key in Dokku's global config. Dokku
injects global config into **every app container's environment**, so any app
on the host could read a reusable, pre-approved key that adds `tag:dokku`
machines to your tailnet. The global value is still used as a fallback (with a
deprecation warning), but you should move it:

1. Generate a **new** auth key in the Tailscale admin console (Settings → Keys).
   The old key was exposed to every app, so don't reuse it.
2. Store the new key: `ssh dokku@host tailscale:set-key < key.txt`
3. Remove the old key from global config:
   `dokku config:unset --global TS_AUTHKEY`
4. Restart apps: `dokku ps:restart --all`. Running containers keep the old
   global environment until they are restarted.
5. Revoke the old key in the Tailscale admin console.
6. Confirm with `dokku tailscale:key-status`. It should say just `key file`,
   with no warning about a leftover global `TS_AUTHKEY`.

&nbsp;

_(Optional)_ Set login server url to the global config:

```sh
dokku config:set --global TS_LOGIN_SERVER=https://headscale.example.com
```

_(Optional)_ Set tailscale extra args to the global config:

```sh
# Example to force re-authentication
dokku config:set --global TS_EXTRA_ARGS="--force-reauth"
```

Presently, all apps will be tagged in the tailnet with `dokku`.

## Usage

Tailscale can expose a container to your tailnet as a machine by running the
tailscale image as a sidecar, and setting your app container's network mode to
share the tailscale container's network. See Tailscale's [docker
documentation](https://tailscale.com/kb/1282/docker) or their [blog
post](https://tailscale.com/blog/docker-tailscale-guide) on the topic.

This plugin manages the sidecar for you, and can optionally serve traffic
straight to the app port with `tailscale serve`.

See the available commands:

```sh
dokku tailscale:help
```

### Examples

#### From Scratch

1. Create an app `dokku apps:create <app>`
2. Run `dokku tailscale:up <app>` on the app
3. Tailscale resources will be removed when you `dokku apps:destroy`

The tailscale machine will take the same name as the app.

For example, to run nginx on port 80:

```sh
dokku apps:create testing-nginx-on-dokku
dokku tailscale:up testing-nginx-on-dokku
dokku git:from-image testing-nginx-on-dokku nginx
```

Visit `http://testing-nginx-on-dokku` from a device on your tailnet. You should
see the default nginx page.

Now destroy the app:

```sh
dokku apps:destroy testing-nginx-on-dokku
```

#### Add to existing container

If you added tailscale to an app which already had a container running in it,
you will need to restart the container so that it joins the tailscale
container's network:

```sh
dokku ps:restart <app>
```

The tailscale plugin doesn't do that automatically, as this may cause downtime
during the restart, and it's better for the caller to decide when and how to do
this.

#### Containers running not on port 80

Create a container which runs on any other port. Here, we'll start a Traefik v2
container with the dashboard/api exposed insecurely on port 8080:

```sh
dokku apps:create testing-traefik-on-dokku
dokku tailscale:up testing-traefik-on-dokku
dokku config:set testing-traefik-on-dokku TRAEFIK_API=true TRAEFIK_API_INSECURE=true
dokku git:from-image testing-traefik-on-dokku traefik:v2.5
```

Test by visiting http://testing-traefik-on-dokku:8080 in your browser.

To remove the need for appending `:8080` to the end of the url, you can use
`tailscale serve`, which will also add a Let's Encrypt certificate and serve
traffic over https:

```
dokku tailscale:serve testing-traefik-on-dokku 8080
```
