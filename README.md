# devopsy-recipe-registry

A private Docker registry ([CNCF distribution](https://distribution.github.io/distribution/))
on a [devopsy](https://github.com/hanoii/devopsy-cli) server, behind
[devopsy-traefik](https://github.com/hanoii/devopsy-traefik), with a user and
password. Use it for devopsy's image mode: build and push images from CI or a
build server, and have each environment pull its commit's image.

## Try it

Point the target at your server in `.devopsy/.env` (gitignored), or in the
environment:

```sh
echo DEVOPSY_TARGET_HOST=devopsy@203.0.113.10 >> .devopsy/.env
```

It gets `<project>.<server's wildcard domain>`, which
each release imports from the server's proxy (devopsy-traefik), through the
`devopsy.import` label in `compose.yaml`: a release fails while no proxy
runs. The release says what it imported; `devopsy @prod --debug imports`
shows it later, and whether the proxy has changed it since. Override it,
or set it empty for none, per target: `devopsy @prod --vars set --show
DEVOPSY_WILDCARD_DOMAIN`.

```sh
devopsy @prod --release          # runs deploy: https://registry-prod.<server's wildcard domain>
devopsy @prod credentials      # URL, user and password for docker login
devopsy @prod logs -f registry
```

Then, wherever you push or pull:

```sh
docker login registry-prod.example.com -u devopsy
docker tag myapp registry-prod.example.com/myapp:abc123
docker push registry-prod.example.com/myapp:abc123
```

## How it works

- **Secrets.** The first `deploy` generates `REGISTRY_PASSWORD` and
  `REGISTRY_HTTP_SECRET` into the server's `shared/.env`. Every `deploy`
  checks `mnt/auth/htpasswd` against `REGISTRY_USER` (default `devopsy`) and
  `REGISTRY_PASSWORD`, and rewrites it (bcrypt, through `httpd`'s `htpasswd`)
  when they differ: to change the password, edit `shared/.env` and deploy.
- **Storage.** Images live in `shared/mnt/registry`: back it up, or accept
  rebuilding images. The registry runs as UID 10001 with a read-only root
  filesystem; a one-shot `init` service gives it the storage directory,
  which the deploy user cannot do.
- **Uploads.** Traefik limits how long a request may take, body included, to
  60 seconds by default, which can cut a large layer pushed over a slow link.
  Raise it with `DEVOPSY_READ_TIMEOUT` in Traefik's `.devopsy/.env` (for
  example `600s`, or `0` for no limit) and `devopsy restart` there.

## Cleaning up

Pushing a tag again, or deleting it, leaves its old layers behind. Delete
tags with any registry client (for example `crane delete`), then:

```sh
devopsy @prod gc             # stops the registry, collects, starts it again
devopsy @prod gc --dry-run   # only lists what would go
```

`gc` removes untagged manifests and the layers nothing else uses. An image
pushed only by digest, never tagged, counts as untagged. Older registry
versions deleted the platform images of multi-platform tags this way; on
3.1.2 a kept tag still pulls on every platform (checked with
`crane export --platform`). Check again when upgrading the registry image.

## Locally

With a local devopsy-traefik, the registry is at
`https://devopsy-recipe-registry.localhost`:

```sh
devopsy deploy
devopsy credentials
```

## License

MIT, so you can start your own project from it. See [LICENSE](LICENSE).
