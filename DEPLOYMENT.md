# Plan de despliegue e integración (OpnForm)

Este documento resume lo que se hizo en la instancia de desarrollo, los cambios verificables, los backups realizados y los pasos recomendados para integrar estos cambios y promoverlos a producción. Incluye los comandos exactos que se ejecutaron, el procedimiento para reconstruir imágenes Docker, el arreglo de la advertencia de collation en PostgreSQL y la checklist final para la promoción a production.

> Nota: todos los comandos se ejecutan desde el directorio del proyecto en el host donde está desplegado (/opt/OpnForm). Ajusta paths y nombres de servicios si difieren.

---

## Resumen de cambios aplicados en la instancia local / dev

- Ejecutado `docker-compose pull` para traer las últimas imágenes remotas.
- Detectado que el script `scripts/docker-setup.sh` en el repo usa `docker compose` (v2) mientras que el host tiene `docker-compose` (v1). Se creó un fallback para compatibilidad (se documentó y/o actualizó el script localmente).
- Se inspeccionó y respaldó almacenamiento y volúmenes (volúmenes Docker: `opnform_opnform_storage`, `opnform_postgres-data`, `opnform_redis-data`). Se identificó que el almacenamiento de usuario apunta por symlink a `/persist/storage`.
- Se resolvió el warning de collation en PostgreSQL (colly mismatch) ejecutando REINDEX / ALTER DATABASE y confirmando que `datcollversion` quedó en `2.41` para `forge`, `postgres` y `template1`.
- Se creó (o se recomendó crear) `docker-compose.override.yml` para forzar `build` desde el código de `./api` y `./client` cuando queramos reconstruir imágenes localmente.
- Procedimiento para reconstruir imágenes y recrear contenedores fue probado y documentado.

---

## Backups realizados / recomendados

Antes de cualquier cambio en producción se deben tener respaldos fuera del host.

1. Backup de la base de datos (ya hecho en la instancia):
   - `pg_dump` creado y almacenado en `/opt/OpnForm/backups/` (ejemplo: `pg_dump_forge_YYYY-MM-DD.sql`).

2. Backup del storage (archivos subidos por usuarios):
   - Volumen principal detectado: `opnform_opnform_storage` (contiene `app/`, `framework/`, `logs`, y `storage -> /persist/storage`).
   - Backup de volumen (ejemplo):
     ```bash
     mkdir -p /opt/OpnForm/backups
     docker run --rm -v opnform_opnform_storage:/data -v /opt/OpnForm/backups:/backup alpine \
       sh -c "cd /data && tar czf /backup/opnform_storage_$(date +%F_%H%M%S).tgz ."
     ```
   - Backup del volumen apuntado por `/persist/storage` (sustituir `<PERSIST_VOL>` por el nombre real):
     ```bash
     docker run --rm -v <PERSIST_VOL>:/data -v /opt/OpnForm/backups:/backup alpine \
       sh -c "cd /data && tar czf /backup/persist_storage_${PERSIST_VOL}_$(date +%F_%H%M%S).tgz ."
     ```

3. (Opcional pero recomendado) Snapshot de las imágenes Docker:
   ```bash
   docker save jhumanj/opnform-api:latest -o /opt/OpnForm/backups/jhumanj_opnform-api_latest_$(date +%F).tar
   docker save jhumanj/opnform-client:latest -o /opt/OpnForm/backups/jhumanj_opnform-client_latest_$(date +%F).tar
   ```

4. Verificar checksums y mover fuera del host:
   ```bash
   sha256sum /opt/OpnForm/backups/* | tee /opt/OpnForm/backups/checksums.txt
   # copiar fuera
   scp /opt/OpnForm/backups/* user@backup-host:/path/to/backup/
   ```

---

## Collation mismatch en PostgreSQL — pasos ejecutados

Situación: logs mostraban avisos como:
```
WARNING: database "forge" has a collation version mismatch
DETAIL: The database was created using collation version 2.36, but the operating system provides version 2.41.
HINT: Rebuild all objects ... ALTER DATABASE forge REFRESH COLLATION VERSION
```

Acciones realizadas (ejecutadas en contenedor `opnform-db`):

1. Ver estado:
```bash
docker exec -it opnform-db psql -U forge -d postgres -c "SELECT datname, datcollversion FROM pg_database ORDER BY datname;"
```

2. REINDEX (reconstrucción de índices) — para `forge` y, si aplica, `postgres`:
```bash
# Reindex para forge
docker exec -it opnform-db psql -U forge -d forge -c "REINDEX DATABASE forge;"

# Reindex para postgres (si pendiente)
docker exec -it opnform-db psql -U forge -d postgres -c "REINDEX DATABASE postgres;"
```

3. Actualizar versión de collation registrada:
```bash
# Para forge
docker exec -it opnform-db psql -U forge -d postgres -c "ALTER DATABASE forge REFRESH COLLATION VERSION;"

# Para postgres y template1 (si corresponde)
docker exec -it opnform-db psql -U forge -d postgres -c "ALTER DATABASE postgres REFRESH COLLATION VERSION;"
docker exec -it opnform-db psql -U forge -d postgres -c "ALTER DATABASE template1 REFRESH COLLATION VERSION;"
```

4. Verificación final:
```bash
docker exec -it opnform-db psql -U forge -d postgres -c "SELECT datname, datcollversion FROM pg_database WHERE datname IN ('forge','postgres','template1') ORDER BY datname;"
# salida esperada: datcollversion = 2.41 para todas
```

Estado final verificado: `forge`, `postgres` y `template1` mostraron `datcollversion = 2.41` y las advertencias dejaron de aparecer en los logs.

> Precaución: `REINDEX DATABASE` puede bloquear y tardar; ejecutar en ventana de mantenimiento si la base es grande o tráfico alto.

---

## Flujo recomendado para integrar cambios locales y desplegarlos en esta instancia (pasos concretos)

Objetivo: trabajar en la instancia actual (que ya tiene las imágenes actualizadas) y, cuando quieras desplegar cambios de código, reconstruir las imágenes para esa instancia y posteriormente promover a producción.

1. Trabajo local / ramas Git
   - Trabajar en una rama de feature:
     ```bash
     git checkout -b feature/mi-cambio
     # hacer cambios, commits
     git add .
     git commit -m "feat: descripción"
     ```
   - Subir a GitHub y abrir PR contra `main`:
     ```bash
     git push origin feature/mi-cambio
     # crear PR desde la UI de GitHub
     ```

2. Integración / CI
   - Configurar pipeline CI que:
     - Ejecuta tests
     - Construye imágenes (opcional)
     - Etiqueta imágenes (preferible usar tag con versión o commit SHA, NO usar `latest` en producción)
   - Recomendación: cada merge a `main` produce una build y un tag `vX.Y` o `sha-<commit>` y push a registry.

3. Para desplegar en esta instancia dev/staging (reconstruir imágenes desde el código en el host)
   - Crear/actualizar `docker-compose.override.yml` en el host para usar `build:` (ya se hizo/propuso). Ejemplo:
     ```yaml
     services:
       api:
         build:
           context: ./api
           dockerfile: Dockerfile
         image: jhumanj/opnform-api:latest

       ui:
         build:
           context: ./client
           dockerfile: Dockerfile
         image: jhumanj/opnform-client:latest
     ```
   - Parar servicios si es necesario y construir:
     ```bash
     cd /opt/OpnForm
     # opcional: detener contenedores que escriben
     docker-compose -f docker-compose.yml -f docker-compose.override.yml stop api api-worker api-scheduler ui ingress || true

     # rebuild
     docker-compose -f docker-compose.yml -f docker-compose.override.yml build --pull api ui

     # recrear contenedores (mínimo downtime)
     docker-compose -f docker-compose.yml -f docker-compose.override.yml up -d --force-recreate --no-deps api ui
     ```

   - Post-build (si la imagen no hace install/build dentro del Dockerfile):
     ```bash
     docker-compose -f docker-compose.yml -f docker-compose.override.yml run --rm api composer install --no-dev --optimize-autoloader
     docker-compose -f docker-compose.yml -f docker-compose.override.yml run --rm ui npm ci
     docker-compose -f docker-compose.yml -f docker-compose.override.yml run --rm ui npm run build
     ```

4. Verificar:
   - Healthchecks y logs:
     ```bash
     docker-compose -f docker-compose.yml -f docker-compose.override.yml ps
     docker-compose -f docker-compose.yml -f docker-compose.override.yml logs --tail=200 api
     docker logs --tail 200 opnform-db
     ```

5. Migraciones (solo si la release las requiere y después de backups):
   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.override.yml run --rm api php artisan migrate --force
   docker-compose -f docker-compose.yml -f docker-compose.override.yml run --rm api php artisan config:clear
   docker-compose -f docker-compose.yml -f docker-compose.override.yml run --rm api php artisan cache:clear
   docker-compose -f docker-compose.yml -f docker-compose.override.yml run --rm api php artisan config:cache
   ```

---

## Integración con GitHub / release flow sugerido

1. Remotos / verificar:
```bash
git remote -v
git fetch --all
git checkout main
git pull origin main
```

2. Etiquetado y releases (no usar `latest` para producción)
- Al mergear a `main`, la pipeline CI debe:
  - Build + test
  - Tag con `vX.Y.Z` o con `sha-<short>`
  - Push imagen con nombre versionado: `registry.example.com/opnform-api:v1.2.3` y `.../opnform-client:v1.2.3`
- En el servidor de producción, actualizar `docker-compose.yml` para apuntar a la etiqueta versionada y hacer:
```bash
docker-compose pull
docker-compose up -d --force-recreate --no-deps api ui
```

3. Documentar release en GitHub (changelog) y en este repo (archivo `RELEASES.md` o en GitHub Releases).

---

## Rollback plan (si algo falla tras deploy)

1. Si el problema es con la imagen nueva:
   - Revertir a la imagen anterior (si se guardó con tag/version):
     ```bash
     docker-compose -f docker-compose.yml -f docker-compose.override.yml pull
     docker-compose -f docker-compose.yml -f docker-compose.override.yml up -d --force-recreate --no-deps api ui
     ```
   - O usar `docker image` cargada desde snapshot:
     ```bash
     docker load -i /opt/OpnForm/backups/jhumanj_opnform-api_latest_YYYY-MM-DD.tar
     docker-compose up -d --force-recreate --no-deps api
     ```

2. Si el problema es migración:
   - Restaurar la base desde el pg_dump (en un ambiente de recuperación o temporal) y revertir migraciones manualmente (si es posible).

3. Si hay pérdida de archivos en storage:
   - Restaurar desde el tar del volumen:
     ```bash
     docker volume create opnform_opnform_storage_restore
     docker run --rm -v opnform_opnform_storage_restore:/data -v /opt/OpnForm/backups:/backup \
       sh -c "cd /data && tar xzf /backup/persist_storage_<VOL>_YYYY-...tgz"
     ```

---

## Checklist antes de promover a producción

- [ ] Backup de la base de datos (pg_dump) completado y copiado a almacenamiento externo.
- [ ] Backup del storage (volúmenes) completado y copiado a almacenamiento externo.
- [ ] Imágenes buildadas y guardadas (o etiquetadas en registry) con tags versionados — no usar `latest` en prod.
- [ ] Tests automatizados verdes en CI.
- [ ] Revisadas y aprobadas PRs en GitHub.
- [ ] Ventana de mantenimiento programada para ejecutar `REINDEX` / migraciones si aplica.
- [ ] Documentación del release (CHANGELOG / Release notes) incluida en GitHub Release.
- [ ] Plan de rollback documentado y verificado.

---

## Documentación breve para el equipo (puntos clave)

- La instancia actual está operativa y lista para reconstruir imágenes desde el código.
- Se resolvió la advertencia de collation en PostgreSQL (se verificó `datcollversion=2.41`).
- Antes de cualquier despliegue a producción: crear backups (DB y storage), taggear imágenes y preferir deployments con imágenes versionadas.
- Si en producción se usa `image:` apuntando a un registry, un `docker-compose pull` solo descarga imágenes: *hay que recrear* los contenedores (`docker-compose up -d --force-recreate`) para que tomen la nueva imagen.

---

## Próximos pasos

- Crear una release en GitHub con este documento y el changelog correspondiente.
- Integrar CI que cree imágenes versionadas por cada merge a main.
- Verificar el procedimiento en un entorno staging antes de promover a producción.

---

End of file
