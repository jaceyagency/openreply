# OpenReply para JACEY y 78 ESTUDIO

## Flujo y arquitectura

La aplicación Next.js gestiona cuentas, campañas y webhooks. El endpoint `/api/webhook` valida la firma HMAC de Meta y encola los comentarios en Redis mediante BullMQ. El worker selecciona la campaña por cuenta, publicación y palabra clave, y llama a la API oficial de Meta con el ID del comentario en `recipient.comment_id`. PostgreSQL y Prisma conservan la configuración y los registros de entrega.

Los trabajos usan IDs deterministas. PostgreSQL identifica cada entrega por campaña y comentario. El worker reclama la entrega de forma atómica antes de llamar a Meta y conserva el bloqueo si la entrega queda sin confirmar. Revisar el buzón antes de resolver una entrega incierta. La deduplicación cubre la repetición del mismo evento. Dos comentarios distintos de una persona pueden generar dos respuestas.

## Preparación de la campaña

1. Obtener de Ceyma la URL HTTPS definitiva del curso, la cuenta profesional autorizada y la publicación o reel que anunciará el curso.
2. Tras autorización expresa, usar la conexión directa de Meta en [la guía de OpenReply](setup.md#the-meta-app). Confirmar el flujo de inicio de sesión disponible para la app y los permisos que Meta exige para la cuenta cliente. Completar App Review o Advanced Access si corresponde. Configurar el webhook `comments` y comprobar que los eventos llegan firmados. [Referencia de webhooks de Instagram](https://developers.facebook.com/docs/graph-api/webhooks/reference/instagram/)
3. En el espacio de JACEY, un propietario o administrador puede enviar una copia de [la configuración de campaña](78estudio-campaign.json) al endpoint autenticado `POST /api/automations/import`. Completar `instagramAccountId` con el ID interno de la conexión Meta, `postId` con el ID de la publicación y añadir la URL HTTPS validada al final de `dmMessage`. Mantener `isActive` en `false`. Los identificadores vacíos hacen que la plantilla original sea rechazada.
4. Revisar la campaña pausada en el panel. Usa `ACUARELA` como palabra completa, sin distinción de mayúsculas, en la publicación seleccionada. El enlace viaja directamente en el primer mensaje privado. Dejar desactivados el mensaje de apertura, la condición de seguimiento, los seguimientos automáticos, la respuesta pública y el disparador desde mensajes entrantes. Deshabilitar el enlace compartido del informe si solo lo consultará JACEY.
5. Con autorización posterior, probar con una cuenta de prueba ajena a 78 ESTUDIO. Comprobar que «¡ACUARELA! 🎨» recibe una sola respuesta con la URL correcta, que «ACUARELAS» y otra publicación quedan fuera, y que repetir el mismo evento conserva un solo envío. Revisar las campañas activas para evitar solapamientos con esa publicación. Usar una publicación de prueba sin comentarios elegibles previos, ya que el reconciliador puede recuperar comentarios recientes. Volver a pausar tras la prueba hasta recibir autorización de activación.

## Infraestructura sin suscripciones

Usar una máquina Linux existente con Docker Engine y Compose 2.24.4 o posterior para ejecutar web, worker, cron, PostgreSQL y Redis. Reutilizar un SMTP TLS para los enlaces de acceso. El funcionamiento continuo depende de que la máquina permanezca encendida y accesible. Meta necesitará un endpoint HTTPS público cuando se autorice la conexión. Reutilizar un dominio y proxy HTTPS existentes si están disponibles.

Aplicar [la configuración de JACEY](../docker-compose.jacey.yml) después de los dos archivos Compose del proyecto. PostgreSQL y Redis quedan en la red privada de contenedores y el puerto de la aplicación solo escucha en el host local, donde se conectaría el proxy HTTPS. Redis usa persistencia AOF. Conservar y respaldar los volúmenes de PostgreSQL y Redis. La configuración establece `INSTAGRAM_META_ONLY=true`, que deshabilita la gestión y recepción de eventos de Zernio, además de bloquear sus peticiones salientes.

Preparar un archivo local `.env` con las variables de [la guía original](setup.md#environment-variables). Establecer `JACEY_POSTGRES_PASSWORD` para una instancia nueva, y configurar `DOCKER_DATABASE_URL` con el host `postgres` y esa misma contraseña codificada como componente de URL. Validar la composición con `docker compose -f docker-compose.yml -f docker-compose.app.yml -f docker-compose.jacey.yml config --quiet`. Este comando valida la configuración sin iniciar servicios ni mostrar sus secretos.

Guardar los secretos fuera del repositorio y usar una lista `ALLOWED_EMAILS` con las personas autorizadas de JACEY. Generar valores independientes para `NEXTAUTH_SECRET`, `CRON_SECRET`, `WEBHOOK_VERIFY_TOKEN` y `ENCRYPTION_KEY`. La aplicación cifra los tokens de Meta con AES-256-GCM. Conservar la clave de cifrado junto con el procedimiento de recuperación y limitar el acceso a los registros, que contienen datos de Instagram.

Quedan pendientes la URL del curso, los identificadores, las credenciales, el SMTP y el alojamiento HTTPS. La conexión de Instagram, la configuración externa de Meta, el despliegue y la activación requieren autorización posterior.

## Verificación local

- `npm ci --cache .npm-cache` y `npm run db:generate` completados con Node 24.22.3. La auditoría de la instalación señaló 36 vulnerabilidades en las dependencias existentes, entre ellas 4 críticas. El archivo de bloqueo se conserva para revisión.
- `npm test` completado con 417 pruebas aprobadas y 38 omitidas en cuatro suites que requieren `TEST_DATABASE_URL` o `TEST_REDIS_URL`.
- `npm run lint`, `npm run typecheck` y `npm run build` completados. Next.js 16.2.6 generó las 63 páginas previstas durante la compilación.
- Una comprobación local de la plantilla confirmó el estado pausado y la coincidencia por palabra completa. El YAML de la configuración JACEY pasó una comprobación de sintaxis, puertos y modo Meta. Docker Compose no está instalado en este entorno.
- La prueba de entrega integral con Meta queda pendiente de una cuenta y un entorno autorizados.

## Revisión de seguridad

La instalación registró 36 entradas de auditoría, cuatro críticas. npm cuenta paquetes afectados, incluidos los que heredan un aviso de otra dependencia. La caché conserva los avisos originales y permite identificar las cuatro entradas críticas. El registro de npm no respondió a una nueva consulta desde este entorno. Una ejecución offline arroja un informe vacío porque npm omite la consulta. El archivo de bloqueo sigue registrando las versiones afectadas.

| Entrada crítica | Causa y alcance en OpenReply |
| --- | --- |
| `next@16.2.6` | La caché recoge avisos de ejecución de código en servidores Windows, el optimizador de imágenes, `ImageResponse` y funciones con Cache Components. Se desactivó el optimizador. Las imágenes que usan `next/image` son SVG locales. No se usa `next/og` ni están habilitados los Cache Components. El aviso de Windows y los demás avisos de Next.js siguen pendientes de actualización. [Windows](https://github.com/advisories/GHSA-p293-qw3h-jr36) · [AVIF](https://github.com/advisories/GHSA-2xp9-vwfh-vxw4) · [ImageResponse](https://github.com/advisories/GHSA-vcvr-r3jv-pc5j) · [SVG](https://github.com/advisories/GHSA-2464-fhr4-w9xw) · [Cache Components](https://github.com/advisories/GHSA-673q-mjm6-r5xr) |
| `next-auth@5.0.0-beta.31` | El inicio de sesión por correo podía validar una dirección antes de normalizar Unicode. Ambos proveedores aplican ahora la misma normalización antes de validar un único destinatario. Otro aviso afecta a código que concede acceso por la mera existencia de una sesión. Las comprobaciones de acceso revisadas requieren `session.user.id`. [Correo](https://github.com/advisories/GHSA-7rqj-j65f-68wh) · [Sesiones](https://github.com/advisories/GHSA-8fpg-xm3f-6cx3) |
| `@auth/core@0.41.2` | Contiene el normalizador afectado. La mitigación en los proveedores cubre el flujo de correo de OpenReply. El paquete sigue dentro del rango vulnerable hasta la versión 0.41.3. [Aviso](https://github.com/advisories/GHSA-7rqj-j65f-68wh) |
| `@auth/prisma-adapter@2.11.2` | Hereda la entrada crítica por su dependencia de `@auth/core`. La actualización debe verificarse junto con `next-auth` y la creación y consumo de enlaces de acceso. |

Los demás avisos de la caché se agrupan en estas cadenas. El alcance debe reconfirmarse con `npm audit --json` y `npm audit --omit=dev --json` cuando haya acceso al registro.

- **Aplicación y compilación.** Next.js arrastra PostCSS, Sharp y datos de compatibilidad de navegadores. También aparecen avisos de Nodemailer, incluidas su gestión de mensajes y su exposición a consumo de recursos, y de Zod por agotamiento de memoria al validar colecciones grandes. Se necesitan versiones corregidas y una prueba de los flujos SMTP y de validación.
- **Prisma y utilidades transitivas.** La cadena de Prisma incluye Hono, su servidor Node, MySQL, Effect, Deepmerge, Valibot y utilidades de esquemas y URI. Los avisos abarcan control de acceso, manejo de rutas y consumo de recursos. OpenReply usa PostgreSQL y no inicia Prisma Studio ni el servidor local de Prisma como parte del stack JACEY. Hay que verificar el generador, las migraciones y las consultas tras actualizar Prisma.
- **Herramientas de desarrollo.** Vitest, su mocker, Vite, esbuild, Babel, ESLint, TypeScript ESLint, Tailwind y utilidades transitivas incluyen avisos de acceso a archivos, análisis de entradas y consumo de recursos. Su superficie principal está en pruebas, lint, compilación y servidores de desarrollo. Hay que verificar la cadena de herramientas completa después de actualizarla.

`npm audit fix --force` puede sacar dependencias directas de los rangos de `package.json`. Cambiar Next.js sin su configuración de ESLint, alterar las versiones de Auth.js por separado o sustituir dependencias internas de Prisma puede romper la compilación, el inicio de sesión o las migraciones. No se ejecutó ese comando. La remediación de dependencias queda bloqueada por el acceso al registro npm.

El webhook contrasta la firma HMAC con los bytes recibidos antes de decodificar JSON. Las firmas rechazadas devuelven 401 sin generar escrituras en PostgreSQL. El reto GET sigue requiriendo `WEBHOOK_VERIFY_TOKEN`. Solo se encolan eventos de las cuentas conectadas al proveedor Meta. Las pruebas locales no verifican una entrega real de Meta.

La deduplicación de comentarios usa el ID determinista de BullMQ y la restricción única de PostgreSQL por campaña y comentario. La reclamación atómica previa al envío protege frente a reintentos, concurrencia y eliminación del trabajo en Redis. Un resultado incierto queda reclamado y puede dejar un mensaje sin entregar. Dos campañas solapadas o dos comentarios distintos pueden producir dos respuestas. Esta revisión no verificó la reclamación contra PostgreSQL.

La revisión estática del Compose de JACEY confirma el cierre de los puertos de PostgreSQL y Redis y la publicación de la web en `127.0.0.1`. Redis carece de autenticación y confía en los contenedores de la red del proyecto. La contraseña de PostgreSQL es obligatoria y solo inicializa volúmenes nuevos. Reutilizar un volumen con la contraseña del archivo Compose base requiere cambiar la contraseña en PostgreSQL y sincronizar la URL antes de usarlo. El archivo de ejemplo contiene valores de muestra y los archivos locales de credenciales están excluidos del repositorio y de la imagen. El aislamiento efectivo y la fusión de los tres archivos Compose siguen pendientes de validación con Docker.

### Pruebas con PostgreSQL y Redis locales

En este entorno Docker no está disponible, WSL deniega la enumeración de distribuciones y no se encontraron servicios PostgreSQL o Redis locales. Cuando el motor Docker esté disponible, usar contenedores desechables separados del stack JACEY y publicar sus puertos solo en loopback. Estos comandos PowerShell requieren que las imágenes estén disponibles localmente.

```powershell
$testDbPassword = node -p "require('node:crypto').randomBytes(24).toString('hex')"
docker run --pull=never --rm -d --name openreply-test-postgres -p 127.0.0.1:55432:5432 -e POSTGRES_PASSWORD=$testDbPassword postgres:16
docker run --pull=never --rm -d --name openreply-test-redis -p 127.0.0.1:6390:6379 redis:7-alpine
docker exec openreply-test-postgres pg_isready -U postgres
docker exec openreply-test-redis redis-cli ping
$env:TEST_DATABASE_URL = "postgresql://postgres:${testDbPassword}@127.0.0.1:55432/postgres"
$env:TEST_REDIS_URL = "redis://127.0.0.1:6390"
npm test
docker stop openreply-test-postgres openreply-test-redis
Remove-Item Env:TEST_DATABASE_URL, Env:TEST_REDIS_URL
```
