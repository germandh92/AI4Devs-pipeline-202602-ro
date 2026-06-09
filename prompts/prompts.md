# Prompts — Ejercicio Pipeline GitHub Actions — GDH

Este fichero recoge los prompts de la sesión. La versión detallada y agrupada por
fase del pipeline está en [`prompts-iniciales.md`](./prompts-iniciales.md).

## Prompt principal

> Crea un pipeline en GitHub Actions que se dispare con un push a una rama con un
> Pull Request abierto y que: (1) pase los tests del backend, (2) genere el build
> del backend y (3) despliegue el backend en una EC2 (capa gratuita de AWS).
> Documenta los prompts utilizados para cada paso.

## Prompts por paso (resumen)

1. **Trigger** — Forma correcta de disparar solo en push a rama con PR abierto
   → evento `pull_request` con `types: [opened, synchronize, reopened]`.
2. **Tests** — Job con Node 20 + caché npm, `npm ci`, `npx prisma generate`, `npm test`
   (los tests mockean Prisma, no necesitan BBDD).
3. **Build** — Job `needs: test`: `npm run build` (tsc → `dist`) y empaquetado del
   artefacto mínimo de producción (`dist`, manifiestos npm, `prisma`).
4. **Deploy EC2** — Job `needs: build` con SSH + PM2 (`scp-action` + `ssh-action`):
   `npm ci --omit=dev`, `npx prisma generate`, `pm2 reload || pm2 start`.

> El detalle de cada prompt, las decisiones tomadas y las notas de verificación
> están en `prompts-iniciales.md`.
