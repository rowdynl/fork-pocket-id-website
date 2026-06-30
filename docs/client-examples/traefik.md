---
title: Traefik
description: Configure any Traefik reversed proxy service
---

Services that do not implement OIDC but are reverse proxied with Traefik can be protected by the [Traefik OpenID Connect Middleware](https://traefik-oidc-auth.sevensolutions.cc/docs/getting-started) plugin. There are many others, but this one is the most and best rated and seems the most well maintained. Please do your research if this plugin matches your needs.

The following example variables are used, and should be replaced with your actual URLS.

- `https://service.example.com` (The url of your service, reverse proxied by [Traefik](https://traefik.io))
- `https://id.example.com` (The url of your Pocket ID instance)

## Pocket ID Setup

1. In Pocket-ID create a new OIDC Client, name it i.e. `My non OIDC capable service`.
2. Set the _callback URL_ to: `https://service.mydomain.com/oidc/callback`.
3. Set the _logout callback URL_ to: `https://service.mydomain.com/oidc/callback`. This is because otherwise logout will not work. See [here](https://github.com/pocket-id/pocket-id/issues/238) and [here](https://github.com/sevensolutions/traefik-oidc-auth/issues/153).
4. _(Optional)_ Upload a logo for your app or download them from [Self-Hosted Dashboard Icons](https://selfh.st/icons).
5. Copy the `Client ID` and `Client Secret` for use in the next steps.

> [!TIP]
> You can create one OIDC client in Pocket ID for each Traefik service.

## Traefik Setup

This is the setup needed in the configuration files of Traefik.

### Configure the plugin
Enable the plugin in your traefik configuration. For the latest version number, see the [Traefik OpenID Connect Middleware](https://plugins.traefik.io/plugins/66b63d12d29fd1c421b503f5/oidc-authentication) plugin page.

Add the below code to your `traefik.yml` file and restart the container.

```yaml
# /etc/traefik/traefik.yml
experimental:
  plugins:
    traefik-oidc-auth:
      moduleName: "github.com/sevensolutions/traefik-oidc-auth"
      version: "v0.20.1"
```

### Configure the middleware

This is a short YAML example, full configuration options for the middleware can be found [here](https://traefik-oidc-auth.sevensolutions.cc/docs/getting-started/middleware-configuration). You can use one middleware to protect one or more services, but best practices is to create one for each service.

> [!WARNING]
>It is highly recommended to change the default encryption-secret by providing your own 32-character secret using the Secret-option. You can generate a random one [here](https://it-tools.tech/token-generator?length=32).

```yaml
# /etc/traefik/dynamic/oidc-auth.yml
http:
  middlewares:
    oidc-auth: # This is the name of your middleware
      plugin:
        traefik-oidc-auth:
          Secret: "MLFs4TT99kOOq8h3UAVRtYoCTDYXiRcZ" # Please change this secret for your setup!
          Provider:
            Url: "https://id.example.com"
            ClientId: "<YourClientId>"
            ClientSecret: "<YourClientSecret>"
          Scopes: ["openid", "profile", "email"]  # "groups" also supported
```

> [!TIP]
> You can create a OIDC client in Pocket ID for each service. Just create a file like above for each service or put them in one. _Don't forget to change the name of your middleware!_

### Use the middleware
Using the configured client(s) can be done using `docker labels` or `yaml`. Pick your poison.

#### Docker labels

```yaml
# compose.yaml
services:
  my-not-oidc-capable-service:
    image: foo/bar
    labels:
      - traefik.enable=true
      - traefik.http.routers.my-not-oidc-capable-service.rule=Host(`service.example.com`)
      - traefik.http.routers.my-not-oidc-capable-service.middlewares=oidc-auth@file # here goes the name of your middleware: oidc-auth 
```

#### YAML

```yaml
# /etc/traefik/dynamic/my-not-oidc-capable-service.yml
http:
  services:
    my-not-oidc-capable-service:
      loadBalancer:
        servers:
          - url: http://localhost:2222

  routers:
    my-not-oidc-capable-service:
      rule: "Host(`service.example.com`)"
      middlewares:
        - oidc-auth@file # here goes the name of your middleware: oidc-auth 
      service: my-not-oidc-capable-service
```