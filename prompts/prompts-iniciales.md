# Prompts iniciales — Ejercicio Pipeline (GitHub Actions + EC2) — GDH

Modelo asistente: Claude Opus 4.8 (1M context) vía Claude Code.
Alumno: Diego G.H. (germandh92). Iniciales: **GDH**.
Convenciones usadas: carpeta `AI4Devs-pipeline-GDH`, rama `pipeline-GDH`, PR dentro del propio fork.

Repositorio base: [`LIDR-academy/AI4Devs-pipeline-202602-ro`](https://github.com/LIDR-academy/AI4Devs-pipeline-202602-ro)
(backend LTI en TypeScript: Express + Prisma + Jest; scripts `npm test`, `npm run build` → `dist`, `npm start`).

A continuación se recoge una transcripción **editada** de los prompts más relevantes,
agrupados por fase del pipeline. Se omiten respuestas largas y ruido de exploración:
el objetivo es poder reproducir el razonamiento que llevó al `pipeline.yml` final.

---

## 0. Prompt inicial (enunciado del ejercicio)

> Crea un pipeline en GitHub Actions que se dispare con un push a una rama con un
> Pull Request abierto y que: (1) pase los tests del backend, (2) genere el build
> del backend y (3) despliegue el backend en una EC2 (capa gratuita de AWS).
> Configúralo en `.github/workflows/pipeline.yml` y documenta los prompts en
> `prompts/prompts-iniciales.md`.

Decisiones de alineación previas:
- Repo: trabajo sobre el fork del repo base; rama de entrega `pipeline-GDH`.
- Estrategia de despliegue elegida: **SSH + PM2** (la más estándar y ligera para
  una `t2.micro` de capa gratuita).

---

## 1. Trigger: "push a una rama con un PR abierto"

> ¿Cuál es la forma correcta en GitHub Actions de disparar un workflow exactamente
> cuando se hace push a una rama que tiene un Pull Request abierto, y no en cualquier
> push suelto?

**Conclusión aplicada:** usar el evento `pull_request` con
`types: [opened, synchronize, reopened]`. El tipo `synchronize` se emite en cada
push a la rama origen de un PR **ya abierto** — que es justo el requisito. Un push a
una rama sin PR no dispara el workflow. Se añade `concurrency` con
`cancel-in-progress` para cancelar ejecuciones obsoletas del mismo PR.

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
```

---

## 2. Tests del backend

> El backend está en la subcarpeta `backend/`, usa TypeScript, Prisma y Jest
> (`npm test`). Escríbeme el job de tests: checkout, Node 20 con caché de npm,
> `npm ci`, generar el cliente de Prisma y ejecutar los tests. ¿Necesito una base
> de datos para los tests?

**Conclusión aplicada:** job `test` con `working-directory: backend`,
`actions/setup-node@v4` (Node 20 + caché npm apuntando a `backend/package-lock.json`),
`npm ci`, `npx prisma generate` (necesario para los tipos generados) y `npm test`.
Los tests unitarios mockean Prisma, por lo que **no requieren** una BBDD real.

> Si en el futuro hubiera tests de integración que sí necesitan PostgreSQL, ¿cómo lo añado?

**Nota documentada:** se añadiría un `services: postgres:16` al job con `DATABASE_URL`
apuntando a `localhost`, más un paso `npx prisma migrate deploy` antes de los tests.

---

## 3. Build del backend

> Ahora el job de build: debe depender de que pasen los tests, compilar el
> TypeScript (`npm run build`, que genera `dist`) y dejar listo un artefacto con lo
> mínimo para producción para que el deploy lo consuma.

**Conclusión aplicada:** job `build` con `needs: test`. Repite `npm ci` +
`npx prisma generate`, ejecuta `npm run build` y empaqueta **solo lo imprescindible**
(`dist`, `package.json`, `package-lock.json`, `prisma`) en un `.tar.gz` que se sube
con `actions/upload-artifact@v4`. Así el deploy no recompila ni arrastra
`node_modules` ni `devDependencies`.

---

## 4. Despliegue en EC2 con PM2

> Genera el job de deploy por SSH + PM2: que dependa del build, descargue el
> artefacto, lo copie por scp a la EC2, instale solo dependencias de producción,
> regenere Prisma y arranque/recargue el proceso con PM2. Usa secrets para host,
> usuario y clave SSH. Que solo despliegue cuando el PR va contra `main`.

**Conclusión aplicada:** job `deploy` con `needs: build` y guarda
`if: github.event.pull_request.base.ref == 'main'`. Pasos:
1. `actions/download-artifact@v4` para recuperar el `.tar.gz`.
2. `appleboy/scp-action` para copiarlo a `/home/<user>/app` en la EC2.
3. `appleboy/ssh-action` que descomprime, hace `npm ci --omit=dev`,
   `npx prisma generate` y `pm2 reload backend --update-env || pm2 start dist/index.js --name backend`.

Secrets usados: `EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY` y opcional `EC2_PORT`.
Las variables de entorno de la app (`DATABASE_URL`, `PORT`, …) viven en
`/home/<user>/app/.env` dentro de la propia instancia, no en el repo.

> ¿Qué hay que tener preparado en la EC2 para que esto funcione?

**Nota documentada:** Node 20 + npm, PM2 instalado global (`npm i -g pm2`),
el grupo de seguridad con el puerto 22 (SSH) y el de la app abiertos, la clave
pública del par SSH en `~/.ssh/authorized_keys`, y el directorio `~/app` creado.
Detalle completo en `SETUP-GDH.md`.

---

## 5. Verificación / criterios de aceptación

> Repásame que el workflow cumple el enunciado: trigger correcto, los tres pasos
> en orden (test → build → deploy) y que el deploy depende de que test y build
> hayan pasado.

**Conclusión aplicada:** cadena `test` → `build (needs: test)` → `deploy (needs: build)`.
Si fallan los tests, no hay build; si falla el build, no hay deploy. El trigger
`pull_request/synchronize` cubre "push a una rama con un PR abierto".
