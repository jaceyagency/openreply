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
