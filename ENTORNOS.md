# Diferencias entre Entorno de Desarrollo y Producción en ShipFree

## Índice
1. [Introducción](#introducción)
2. [Comparación General](#comparación-general)
3. [Configuración de Docker](#configuración-de-docker)
4. [Variables de Entorno](#variables-de-entorno)
5. [Proceso de Construcción](#proceso-de-construcción)
6. [Comandos de Uso](#comandos-de-uso)
7. [Mejores Prácticas](#mejores-prácticas)

## Introducción

ShipFree utiliza dos entornos distintos para el desarrollo y la producción, cada uno optimizado para su propósito específico. Esta guía explica las diferencias clave entre ambos entornos y cómo configurarlos correctamente.

## Comparación General

| Característica | Desarrollo | Producción |
|----------------|-----------|------------|
| **NODE_ENV** | `development` | `production` |
| **Modo de ejecución** | `npm run dev` (Turbopack) | `npm start` |
| **Hot Reload** | ✅ Sí (modo watch) | ❌ No |
| **Optimización** | Mínima | Completa |
| **Tamaño de imagen Docker** | Mayor (~500MB+) | Menor (~200MB) |
| **Construcción Docker** | Una sola etapa | Multi-etapa |
| **Volúmenes montados** | ✅ Código fuente montado | ❌ Código copiado en build |
| **Recarga automática** | ✅ Sí | ❌ No |
| **Seguridad** | Básica | Estricta (headers de seguridad) |
| **Base de datos** | Credenciales simples | Credenciales seguras |

## Configuración de Docker

### Entorno de Desarrollo

#### Dockerfile de Desarrollo
Ubicación: `docker/dev/Dockerfile`

```dockerfile
FROM node:21-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --no-audit --prefer-offline --legacy-peer-deps
COPY . .
EXPOSE 3000
RUN npx drizzle-kit push:pg
CMD ["npm", "run", "dev"]
```

**Características clave:**
- **Una sola etapa**: Instalación directa sin optimización
- **Todas las dependencias**: Incluye dev dependencies
- **Comando**: `npm run dev` ejecuta Next.js en modo desarrollo con Turbopack
- **Drizzle**: Ejecuta migrations automáticamente

#### docker-compose.yml de Desarrollo
Ubicación: `docker/dev/docker-compose.yml`

```yaml
services:
  app:
    build:
      context: ../..
      dockerfile: docker/dev/Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - HUSKY=0
    volumes:
      - ../../:/app          # Código fuente montado
      - /app/node_modules    # node_modules en volumen
    restart: unless-stopped
```

**Características clave:**
- **Volúmenes montados**: El código fuente se monta en tiempo real
- **Hot Reload**: Los cambios en el código se reflejan inmediatamente
- **NODE_ENV=development**: Habilita características de desarrollo
- **Portainer**: Incluido para gestión visual de contenedores

### Entorno de Producción

#### Dockerfile de Producción
Ubicación: `docker/prod/Dockerfile`

```dockerfile
# Etapa 1: Builder
FROM node:21-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install --production --no-audit --prefer-offline --legacy-peer-deps --ignore-scripts
COPY . .
RUN npm run build

# Etapa 2: Runner
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/.next ./.next  
COPY --from=builder /app/public ./public  
EXPOSE 3000
CMD ["npm", "start"]
```

**Características clave:**
- **Multi-etapa**: Reduce el tamaño final de la imagen
- **Solo producción**: `--production` excluye dev dependencies
- **Build optimizado**: `npm run build` crea versión optimizada
- **Imagen final pequeña**: Solo contiene lo necesario para ejecutar
- **Node 18**: Versión estable para runtime

#### docker-compose.yml de Producción
Ubicación: `docker/prod/docker-compose.yml`

```yaml
services:
  app:
    image: ghcr.io/idee8/shipfree:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - HUSKY=0
    restart: unless-stopped
```

**Características clave:**
- **Imagen pre-construida**: Usa imagen de registro de GitHub
- **Sin volúmenes**: Código compilado dentro de la imagen
- **NODE_ENV=production**: Optimizaciones de Next.js
- **restart: unless-stopped**: Alta disponibilidad

## Variables de Entorno

### Variables Comunes (requeridas en ambos entornos)

Archivo: `.env` (basado en `.env.example`)

```bash
# Supabase (Autenticación)
NEXT_PUBLIC_SUPABASE_URL=https://abc123.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=tu_clave_anon

# Stripe (Pagos)
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_12345
STRIPE_SECRET_KEY=sk_12345

# LemonSqueezy (Pagos alternativos)
LEMON_SQUEEZY_API_KEY=tu_api_key
LEMON_SQUEEZY_STORE_ID=tu_store_id

# Mailgun (Emails)
MAILGUN_API_KEY=tu_mailgun_api_key
MAILGUN_DOMAIN=tu_mailgun_domain
MAILGUN_FROM_EMAIL=noreply@tudominio.com
```

### Variables Específicas de Desarrollo

```bash
NODE_ENV=development
DATABASE_URL=postgresql://devuser:devpass@postgres:5432/shipfreedev
```

**Características:**
- Credenciales de base de datos simples
- Logs detallados habilitados
- Hot reload habilitado

### Variables Específicas de Producción

```bash
NODE_ENV=production
DATABASE_URL=postgresql://produser:contraseña_segura@postgres:5432/shipfreeprod
```

**Características:**
- Credenciales de base de datos seguras
- Logs optimizados
- Caché habilitado
- Optimizaciones de rendimiento

## Proceso de Construcción

### Desarrollo

1. **Instalación de dependencias**: Todas las dependencias (incluidas dev)
2. **Sin build**: El código se ejecuta directamente con Turbopack
3. **Watch mode**: Next.js monitorea cambios en archivos
4. **Hot Module Replacement (HMR)**: Actualización instantánea sin recargar página

```bash
npm install              # Instala todas las dependencias
npm run dev              # Inicia servidor de desarrollo
# El servidor inicia en http://localhost:3000
```

### Producción

1. **Instalación**: Solo dependencias de producción
2. **Build**: Compilación optimizada con `next build`
3. **Optimizaciones aplicadas**:
   - Minificación de JavaScript y CSS
   - Tree-shaking (eliminación de código no usado)
   - Optimización de imágenes
   - Generación de rutas estáticas
   - Code splitting automático
4. **Ejecución**: Servidor optimizado con `next start`

```bash
npm install --production  # Solo dependencias de producción
npm run build            # Compila aplicación
npm start                # Inicia servidor de producción
```

## Comandos de Uso

### Comandos de Desarrollo

#### Sin Base de Datos
```bash
docker-compose -f docker/dev/docker-compose.yml up --build
```

#### Con PostgreSQL
```bash
docker-compose -f docker/dev/docker-compose.yml -f docker/dev/docker-compose.postgres.yml up --build
```

**Servicios adicionales disponibles:**
- **PostgreSQL**: Base de datos en `localhost:5432`
- **pgAdmin**: Interfaz web en `http://localhost:5050`
  - Email: `admin@example.com`
  - Contraseña: `admin`
- **Portainer**: Gestión de contenedores en `http://localhost:9000`

#### Con MongoDB
```bash
docker-compose -f docker/dev/docker-compose.yml -f docker/dev/docker-compose.mongodb.yml up --build
```

**Para detener:**
```bash
docker-compose -f docker/dev/docker-compose.yml down
```

**Para ver logs:**
```bash
docker-compose -f docker/dev/docker-compose.yml logs -f app
```

### Comandos de Producción

#### Sin Base de Datos
```bash
docker-compose -f docker/prod/docker-compose.yml up --build -d
```

#### Con PostgreSQL
```bash
docker-compose -f docker/prod/docker-compose.yml -f docker/prod/docker-compose.postgres.yml up --build -d
```

#### Con MongoDB
```bash
docker-compose -f docker/prod/docker-compose.yml -f docker/prod/docker-compose.mongodb.yml up --build -d
```

**Nota**: El flag `-d` ejecuta los contenedores en segundo plano (detached mode).

**Para detener:**
```bash
docker-compose -f docker/prod/docker-compose.yml down
```

**Para ver logs:**
```bash
docker-compose -f docker/prod/docker-compose.yml logs -f app
```

**Para reconstruir después de cambios:**
```bash
docker-compose -f docker/prod/docker-compose.yml up --build -d
```

## Configuración de Next.js

### next.config.ts

El archivo `next.config.ts` contiene configuraciones que aplican a ambos entornos:

```typescript
const nextConfig = {
  typescript: { ignoreBuildErrors: true },
  pageExtensions: ["ts", "tsx", "mdx"],
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          {
            key: "Strict-Transport-Security",
            value: "max-age=31536000; includeSubDomains; preload",
          },
          {
            key: "X-Frame-Options",
            value: "DENY",
          },
          {
            key: "X-Content-Type-Options",
            value: "nosniff",
          },
          {
            key: "Referrer-Policy",
            value: "strict-origin-when-cross-origin",
          },
        ],
      },
    ];
  },
};
```

**Headers de seguridad aplicados:**
- **Strict-Transport-Security**: Fuerza HTTPS
- **X-Frame-Options**: Previene clickjacking
- **X-Content-Type-Options**: Previene MIME sniffing
- **Referrer-Policy**: Controla información de referencia

## Diferencias en el Funcionamiento

### Desarrollo

**Cómo funciona:**
1. Docker monta el código fuente como volumen
2. Next.js ejecuta en modo desarrollo con Turbopack
3. Cada cambio en el código activa:
   - Recompilación automática del módulo modificado
   - Hot Module Replacement (HMR)
   - Actualización del navegador sin perder estado
4. TypeScript se compila on-the-fly
5. Logs detallados en consola

**Ventajas:**
- ✅ Desarrollo rápido y ágil
- ✅ Feedback inmediato
- ✅ Debugging facilitado
- ✅ No requiere reconstruir Docker

**Desventajas:**
- ❌ Mayor consumo de recursos
- ❌ Rendimiento más lento
- ❌ No apto para producción

### Producción

**Cómo funciona:**
1. Docker construye la aplicación en etapa de builder
2. Next.js compila todo el código:
   - Genera páginas estáticas cuando es posible
   - Optimiza bundles de JavaScript
   - Procesa y optimiza CSS
   - Optimiza imágenes
3. Copia solo archivos necesarios a imagen final
4. Ejecuta servidor optimizado de Next.js
5. Sirve contenido pre-compilado

**Ventajas:**
- ✅ Máximo rendimiento
- ✅ Tamaño de imagen reducido
- ✅ Mayor seguridad
- ✅ Estabilidad

**Desventajas:**
- ❌ Requiere rebuild para cada cambio
- ❌ Proceso de build más largo
- ❌ Debugging más complejo

## Configuración de Base de Datos con Drizzle ORM

### drizzle.config.ts

```typescript
import "dotenv/config";
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

### Diferencias en Database

#### Desarrollo
- **DATABASE_URL**: `postgresql://devuser:devpass@postgres:5432/shipfreedev`
- **Usuario**: `devuser`
- **Contraseña**: `devpass`
- **Base de datos**: `shipfreedev`
- **pgAdmin disponible**: Sí (puerto 5050)

#### Producción
- **DATABASE_URL**: `postgresql://produser:prodpass@postgres:5432/shipfreeprod`
- **Usuario**: `produser`
- **Contraseña**: `prodpass` (debe cambiarse a contraseña segura)
- **Base de datos**: `shipfreeprod`
- **pgAdmin disponible**: No (por seguridad)

## Mejores Prácticas

### Para Desarrollo

1. **Usar variables de entorno locales**
   ```bash
   cp .env.example .env
   # Editar .env con valores de desarrollo
   ```

2. **Mantener dependencias actualizadas**
   ```bash
   npm update
   ```

3. **Usar hot reload efectivamente**
   - No requiere reiniciar Docker para cambios de código
   - Solo reiniciar si cambias dependencias

4. **Monitorear logs**
   ```bash
   docker-compose -f docker/dev/docker-compose.yml logs -f
   ```

5. **Usar pgAdmin para debugging de base de datos**
   - Acceder a `http://localhost:5050`
   - Conectar a servidor PostgreSQL: `postgres:5432`

### Para Producción

1. **Usar secretos seguros**
   - Nunca usar contraseñas por defecto
   - Usar gestores de secretos (AWS Secrets Manager, HashiCorp Vault)

2. **Variables de entorno seguras**
   ```bash
   # NO commitear archivo .env a git
   # Usar variables de entorno del sistema o secretos de Docker
   docker-compose -f docker/prod/docker-compose.yml \
     -e DATABASE_URL="postgresql://user:pass@host:5432/db" \
     up -d
   ```

3. **Monitorear recursos**
   ```bash
   docker stats
   ```

4. **Backups regulares**
   ```bash
   # Backup de PostgreSQL
   docker exec postgres pg_dump -U produser shipfreeprod > backup.sql
   ```

5. **Actualizar imagen regularmente**
   ```bash
   docker pull ghcr.io/idee8/shipfree:latest
   docker-compose -f docker/prod/docker-compose.yml up -d
   ```

6. **Revisar logs de producción**
   ```bash
   docker-compose -f docker/prod/docker-compose.yml logs --tail=100 app
   ```

7. **Health checks**
   - Configurar alertas para caídas de servicio
   - Usar Portainer para monitoreo visual

## Scripts de package.json

```json
{
  "scripts": {
    "dev": "next dev --turbopack",      // Desarrollo con Turbopack
    "build": "next build",              // Build de producción
    "start": "next start",              // Servidor de producción
    "lint": "eslint .",                 // Linting
    "format": "prettier . --write"      // Formateo de código
  }
}
```

### Uso de Scripts

**Desarrollo local (sin Docker):**
```bash
npm install
npm run dev
```

**Build local:**
```bash
npm run build
npm start
```

**Linting:**
```bash
npm run lint
```

**Formateo:**
```bash
npm run format
```

## Resumen de Diferencias Clave

### 🔧 Desarrollo
- **Propósito**: Facilitar el desarrollo rápido
- **Performance**: Más lento, pero con hot reload
- **Tamaño**: Más grande (incluye todo)
- **Seguridad**: Básica
- **Logs**: Detallados
- **Cambios**: Inmediatos sin rebuild

### 🚀 Producción
- **Propósito**: Máximo rendimiento y estabilidad
- **Performance**: Optimizado al máximo
- **Tamaño**: Imagen pequeña (multi-stage build)
- **Seguridad**: Headers de seguridad configurados
- **Logs**: Optimizados
- **Cambios**: Requieren rebuild completo

## Solución de Problemas Comunes

### Desarrollo

**Problema**: Los cambios no se reflejan
```bash
# Reiniciar contenedor
docker-compose -f docker/dev/docker-compose.yml restart app
```

**Problema**: Error de permisos en node_modules
```bash
# Reconstruir volúmenes
docker-compose -f docker/dev/docker-compose.yml down -v
docker-compose -f docker/dev/docker-compose.yml up --build
```

### Producción

**Problema**: Aplicación no inicia
```bash
# Ver logs
docker-compose -f docker/prod/docker-compose.yml logs app

# Verificar variables de entorno
docker-compose -f docker/prod/docker-compose.yml config
```

**Problema**: Base de datos no conecta
```bash
# Verificar que PostgreSQL está corriendo
docker-compose -f docker/prod/docker-compose.yml ps

# Probar conexión
docker exec -it postgres psql -U produser -d shipfreeprod
```

## Recursos Adicionales

- **Documentación oficial**: [ShipFree Docs](https://shipfree.idee8.agency/docs)
- **Next.js**: [https://nextjs.org/docs](https://nextjs.org/docs)
- **Docker**: [https://docs.docker.com](https://docs.docker.com)
- **Drizzle ORM**: [https://orm.drizzle.team](https://orm.drizzle.team)

---

**Desarrollado con ❤️ por [Revoks](https://revoks.dev)**
