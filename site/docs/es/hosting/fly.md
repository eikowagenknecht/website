# Hosting: Fly

Esta guía te explica las formas de alojar tus bots de grammY en [Fly](https://fly.io), ya sea usando Deno o Node.js.

## Preparando tu código

Puedes ejecutar tu bot usando ambos [webhooks o long polling](../guide/deployment-types.md).

### Webhooks

> Recuerda que no debes llamar a `bot.start()` en tu código cuando uses webhooks.

1. Asegúrate de tener un archivo que exporte tu objeto `Bot`, para poder importarlo después y ejecutarlo.
2. Crea un archivo llamado `app.ts` o `app.js`, o en realidad cualquier nombre que te guste (pero deberías recordarlo y usarlo como el archivo principal para desplegar), con el siguiente contenido:

<CodeGroup>
<CodeGroupItem title="Deno" active>

```ts{11}
import { serve } from "https://deno.land/std/http/server.ts";
import { webhookCallback } from "https://deno.land/x/grammy/mod.ts";
// Podrías modificar esto a la forma correcta de importar tu objeto `Bot`.
import { bot } from "./bot.ts";

const port = 8000;
const handleUpdate = webhookCallback(bot, "std/http");

serve(async (req) => {
  const url = new URL(req.url);
  if (req.method === "POST" && url.pathname.slice(1) === bot.token) {
    try {
      return await handleUpdate(req);
    } catch (err) {
      console.error(err);
    }
  }
  return new Response();
}, { port });
```

</CodeGroupItem>
<CodeGroupItem title="Node.js" active>

```ts{10}
import express from "express";
import { webhookCallback } from "grammy";
// Podrías modificar esto a la forma correcta de importar tu objeto `Bot`.
import { bot } from "./bot";

const port = 8000;
const app = express();

app.use(express.json());
app.use(`/${bot.token}`, webhookCallback(bot, "express"));
app.use((_req, res) => res.status(200).send());

app.listen(port, () => console.log(`listening on port ${port}`));
```

</CodeGroupItem>
</CodeGroup>

Le aconsejamos que tenga su manejador en alguna ruta secreta en lugar de la raíz (`/`).
Como se muestra en la línea resaltada arriba, estamos usando el token del bot (`/<bot token>`) como ruta secreta.

### Long Polling

Crea un archivo llamado `app.ts` o `app.js`, o en realidad cualquier nombre que te guste (pero deberías recordar y usar este como el archivo principal para desplegar), con el siguiente contenido:

<CodeGroup>
<CodeGroupItem title="Deno" active>

```ts{4}
import { Bot } from "https://deno.land/x/grammy/mod.ts";

// Aquí, tomamos el token del bot de la variable de entorno "BOT_TOKEN".
const bot = new Bot(Deno.env.get("BOT_TOKEN") ?? ""); 

bot.command(
  "start",
  (ctx) => ctx.reply("I'm running on Fly using long polling!"),
);

Deno.addSignalListener("SIGINT", () => bot.stop());
Deno.addSignalListener("SIGTERM", () => bot.stop());

bot.start();
```

</CodeGroupItem>
<CodeGroupItem title="Node.js">

```ts{4}
import { Bot } from "grammy";

// Aquí, tomamos el token del bot de la variable de entorno "BOT_TOKEN".
const bot = new Bot(process.env.BOT_TOKEN ?? "");

bot.command(
  "start",
  (ctx) => ctx.reply("I'm running on Fly using long polling!"),
);

process.once("SIGINT", () => bot.stop());
process.once("SIGTERM", () => bot.stop());

bot.start();
```

</CodeGroupItem>
</CodeGroup>

Como puedes ver en la línea resaltada arriba, tomamos algunos valores sensibles (tu token de bot) de las variables de entorno.
Fly nos permite almacenar ese secreto ejecutando este comando:

```sh:no-line-numbers
flyctl secrets set BOT_TOKEN="AAAA:12345"
```

Puedes especificar otros secretos de la misma manera.
Para más información sobre estos _secretos_, véase <https://fly.io/docs/reference/secrets/>.

## Despliegue

### Método 1: Con `flyctl`

Este es el método más sencillo.

1. Instalar [flyctl](https://fly.io/docs/hands-on/install-flyctl) e [inicia la sesión](https://fly.io/docs/hands-on/sign-in/).
2. Ejecuta `flyctl launch` para generar un `Dockerfile` y un archivo `fly.toml` para el despliegue.
   Pero **NO** despliega.

<CodeGroup>
<CodeGroupItem title="Deno" Active>

```sh
flyctl launch
```

```log:no-line-numbers{10}
Creating app in /my/telegram/bot
Scanning source code
Detected a Deno app
? App Name (leave blank to use an auto-generated name): grammy
Automatically selected personal organization: CatDestroyer
? Select region: ams (Amsterdam, Netherlands)
Created app grammy in organization personal
Wrote config file fly.toml
? Would you like to set up a Postgresql database now? No
? Would you like to deploy now? No
Your app is ready. Deploy with `flyctl deploy`
```

</CodeGroupItem>
<CodeGroupItem title="Node.js">

```sh
flyctl launch
```

```log:no-line-numbers{12}
Creating app in /my/telegram/bot
Scanning source code
Detected a NodeJS app
Using the following build configuration:
        Builder: heroku/buildpacks:20
? App Name (leave blank to use an auto-generated name): grammy
Automatically selected personal organization: CatDestroyer
? Select region: ams (Amsterdam, Netherlands)
Created app grammy in organization personal
Wrote config file fly.toml
? Would you like to set up a Postgresql database now? No
? Would you like to deploy now? No
Your app is ready. Deploy with `flyctl deploy`
```

</CodeGroupItem>
</CodeGroup>

3. **Deno**: Cambiar la versión de Deno y eliminar `CMD` si existe en el archivo `Dockerfile`.
   Por ejemplo, en este caso, actualizamos `DENO_VERSION` a `1.25.2`.

   **Node.js**: Para cambiar la versión de Node.js, necesitas insertar una propiedad `"node"` dentro de una propiedad `"engines"` dentro de `package.json`.
   Por ejemplo, actualizamos la versión de Node.js a `16.14.0` en el siguiente ejemplo.

<CodeGroup>
<CodeGroupItem title="Deno" Active>

```dockerfile{2,26}
# Dockerfile
ARG DENO_VERSION=1.25.2
ARG BIN_IMAGE=denoland/deno:bin-${DENO_VERSION}
FROM ${BIN_IMAGE} AS bin

FROM frolvlad/alpine-glibc:alpine-3.13

RUN apk --no-cache add ca-certificates

RUN addgroup --gid 1000 deno \
  && adduser --uid 1000 --disabled-password deno --ingroup deno \
  && mkdir /deno-dir/ \
  && chown deno:deno /deno-dir/

ENV DENO_DIR /deno-dir/
ENV DENO_INSTALL_ROOT /usr/local

ARG DENO_VERSION
ENV DENO_VERSION=${DENO_VERSION}
COPY --from=bin /deno /bin/deno

WORKDIR /deno-dir
COPY . .

ENTRYPOINT ["/bin/deno"]
# CMD es eliminado
```

</CodeGroupItem>
<CodeGroupItem title="Node.js" Active>

```json{19}
// package.json
{
  "name": "grammy",
  "version": "1.0.0",
  "description": "grammy",
  "main": "app.js",
  "author": "itsmeMario",
  "license": "MIT",
  "dependencies": {
    "express": "^4.18.1",
    "grammy": "^1.11.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.14",
    "@types/node": "^18.7.18",
    "typescript": "^4.8.3"
  },
  "engines": {
    "node": "16.14.0"
  }
}
```

</CodeGroupItem>
</CodeGroup>

4. Edita `app` dentro del archivo `fly.toml`.
   La ruta `./app.ts` (o `./app.js` para Node.js) en el ejemplo de abajo se refiere al directorio del archivo principal.
   Puedes modificarlos para que coincidan con el directorio de tu proyecto.
   Si estás usando webhooks, asegúrate de que el puerto es el mismo que el de tu [configuración](#webhooks) (`8000`).

<CodeGroup>
<CodeGroupItem title="Deno (Webhooks)" Active>

```toml{7,11,12}
# fly.toml
app = "grammy"
kill_signal = "SIGINT"
kill_timeout = 5

[processes]
  app = "run --allow-net ./app.ts"

[[services]]
  http_checks = []
  internal_port = 8000
  processes = ["app"]
  protocol = "tcp"
  script_checks = []
  [services.concurrency]
    hard_limit = 25
    soft_limit = 20
    type = "connections"

  [[services.ports]]
    force_https = true
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443

  [[services.tcp_checks]]
    grace_period = "1s"
    interval = "15s"
    restart_limit = 0
    timeout = "2s"
```

</CodeGroupItem>
<CodeGroupItem title="Deno (Long polling)" Active>

```toml{7}
# fly.toml
app = "grammy"
kill_signal = "SIGINT"
kill_timeout = 5

[processes]
  app = "run --allow-net ./app.ts"

# Simply omitting the whole [[services]] section 
# since we are not listening to HTTP
```

</CodeGroupItem>
<CodeGroupItem title="Node.js (Webhooks)" Active>

```toml{7,11,18,19}
# fly.toml
app = "grammy"
kill_signal = "SIGINT"
kill_timeout = 5

[processes]
  app = "node ./build/app.js"

# Adjust the NODE_ENV environment to suppress the warning
[build.args]
  NODE_ENV = "production"

[build]
  builder = "heroku/buildpacks:20"

[[services]]
  http_checks = []
  internal_port = 8000
  processes = ["app"]
  protocol = "tcp"
  script_checks = []
  [services.concurrency]
    hard_limit = 25
    soft_limit = 20
    type = "connections"

  [[services.ports]]
    force_https = true
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443

  [[services.tcp_checks]]
    grace_period = "1s"
    interval = "15s"
    restart_limit = 0
    timeout = "2s"
```

</CodeGroupItem>
<CodeGroupItem title="Node.js (Long polling)" Active>

```toml{7,11,22,23}
# fly.toml
app = "grammy"
kill_signal = "SIGINT"
kill_timeout = 5

[processes]
  app = "node ./build/app.js"

# Adjust the NODE_ENV environment to suppress the warning
[build.args]
  NODE_ENV = "production"

[build]
  builder = "heroku/buildpacks:20"

# Simplemente omitiendo toda la sección de [[servicios]] ya que no estamos escuchando HTTP.
```

</CodeGroupItem>
</CodeGroup>

5. Ejecuta `flyctl deploy` para desplegar tu código.

### Método 2: Con acciones de GitHub

La principal ventaja del siguiente método es que Fly vigilará los cambios en tu repositorio que incluye el código de tu bot, y desplegará las nuevas versiones automáticamente.
Visite <https://fly.io/docs/app-guides/continuous-deployment-with-github-actions> para obtener instrucciones más detalladas.

1. Instala [flyctl](https://fly.io/docs/hands-on/install-flyctl) e [inicia la sesión](https://fly.io/docs/hands-on/sign-in/).
2. Obtén un token de la API de Fly ejecutando `flyctl auth token`.
3. Crea un repositorio en GitHub, puede ser privado o público.
4. Ve a Configuración, elige Secretos y crea un secreto llamado `FLY_API_TOKEN` con el valor del token del paso 2.
5. Crea `.github/workflows/main.yml` con estos contenidos:

```yml
name: Fly Deploy
on: [push]
env:
  FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
jobs:
  deploy:
      name: Deploy app
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v2
        - uses: superfly/flyctl-actions/setup-flyctl@master
        - run: flyctl deploy --remote-only
```

6. Sigue los pasos 2 a 4 del [Método 1](#metodo-1-con-flyctl) anterior.
   Recuerda saltarte el último paso (paso 5) ya que no vamos a desplegar el código directamente.
7. Confirma tus cambios y envíalos a GitHub.
8. A partir de ahora, cada vez que envíes un cambio, la aplicación se desplegará automáticamente.

### Configuración de la URL del Webhook

Si estás utilizando webhooks, después de poner en marcha tu aplicación, debes configurar los ajustes de webhook de tu bot para que apunte a tu aplicación.
Para ello, envía una petición a

```md:no-line-numbers
https://api.telegram.org/bot<token>/setWebhook?url=<url>
```

sustituyendo `<token>` por el token de tu bot, y `<url>` por la URL completa de tu app junto con la ruta al manejador del webhook.

### Optimización de Dockerfile

Cuando nuestro `Dockerfile` se ejecuta, copia todo desde el directorio a la imagen Docker.
Para las aplicaciones Node.js, algunos directorios como `node_modules` van a ser reconstruidos de todos modos, así que no hay necesidad de copiarlos.
Crea un archivo `.dockerignore` y añade `node_modules` a él para hacer esto.
También puedes utilizar `.dockerignore` para no copiar ningún otro archivo que no sea necesario en tiempo de ejecución.

## Reference

- <https://fly.io/docs/languages-and-frameworks/deno/>
- <https://fly.io/docs/languages-and-frameworks/node/>