# Taller: Sistema Web Seguro como Control de Riesgo Empresarial

Sistema web pequeño compuesto por:

- Un **frontend en React** (Vite) que autentica usuarios con **Auth0**.
- Un **backend mínimo en Spring Boot** protegido como **OAuth 2.0 Resource Server** (valida JWT emitidos por Auth0).
- Una **dirección pública HTTPS** usando **Duck DNS**, **Apache HTTP Server** y un certificado de **Let's Encrypt**.

## Tabla de contenido

1. [URL desplegada](#url-desplegada)
2. [Estado del proyecto](#estado-del-proyecto)
3. [Arquitectura](#arquitectura)
4. [Estructura del repositorio](#estructura-del-repositorio)
5. [Autenticación vs. autorización](#autenticación-vs-autorización)
6. [Parte 1. Configuración de Auth0](#parte-1-configuración-de-auth0)
7. [Parte 2. Backend (ejecución local)](#parte-2-backend-ejecución-local)
8. [Parte 3. Frontend (ejecución local)](#parte-3-frontend-ejecución-local)
9. [Parte 4. Servidor y Duck DNS](#parte-4-servidor-y-duck-dns)
10. [Parte 5. Despliegue del backend](#parte-5-despliegue-del-backend)
11. [Parte 6. Apache y HTTPS](#parte-6-apache-y-https)
12. [Parte 7. Lista de verificación](#parte-7-lista-de-verificación)
13. [Evidencias](#evidencias)
14. [Reflexión: controles de riesgo](#reflexión-controles-de-riesgo)
15. [Notas de seguridad](#notas-de-seguridad)
16. [Referencias](#referencias)

## URL desplegada

> **https://YOUR-SUBDOMAIN.duckdns.org** ← _reemplazar por el subdominio real después del despliegue_

## Estado del proyecto

**Completado**

- [x] Código del backend: Resource Server JWT, endpoint protegido, CORS restringido, log de auditoría.
- [x] Pruebas del backend (4/4): 401 sin token, 200 con JWT, health público, otras rutas denegadas.
- [x] Código del frontend: login con Auth0, llamada a la API con `Bearer` token, logout.
- [x] Archivos de despliegue: unidad systemd, VirtualHost de Apache y script de Duck DNS.

**Pendiente**

- [ ] Crear la SPA y la API en Auth0 ([Parte 1](#parte-1-configuración-de-auth0)).
- [ ] Crear `frontend/.env.local` y poner el tenant real en `application.properties`.
- [ ] Probar el flujo completo en local con un usuario real.
- [ ] Crear el subdominio en Duck DNS y preparar el servidor ([Parte 4](#parte-4-servidor-y-duck-dns)).
- [ ] Desplegar el backend, Apache y el certificado ([Partes 5](#parte-5-despliegue-del-backend) y [6](#parte-6-apache-y-https)).
- [ ] Reemplazar `YOUR-SUBDOMAIN` y `YOUR_TENANT` en este README, en `deploy/` y en Auth0.
- [ ] Completar la [lista de verificación](#parte-7-lista-de-verificación).
- [ ] Agregar las [capturas de evidencia](#evidencias) en `docs/`.

## Arquitectura

```
Navegador
  │ login y access token
  ▼
Auth0 ───────────────► Frontend React ── Bearer token ──► API Spring Boot
                                                        │
Internet ── Duck DNS ── Apache + TLS ──────────────────┘
```

1. El usuario inicia sesión en Auth0 desde el frontend (Authorization Code Flow + PKCE).
2. Auth0 entrega un **access token** (JWT firmado con RS256) con audiencia `https://enterprise-risk-api`.
3. El frontend llama a `GET /api/risk-status` enviando `Authorization: Bearer <token>`.
4. Apache termina TLS y hace *reverse proxy* de `/api/` hacia `127.0.0.1:8080`.
5. Spring Security valida firma, emisor (`iss`), audiencia (`aud`) y expiración del token.

## Estructura del repositorio

```
.
├── backend/                         API Spring Boot (Java 21, Maven)
│   ├── pom.xml
│   └── src/main/java/edu/eci/tdse/riskapi/
│       ├── EnterpriseRiskApiApplication.java
│       ├── SecurityConfig.java      Reglas de autorización, Resource Server JWT y CORS
│       └── RiskController.java      Endpoint protegido /api/risk-status
├── frontend/                        SPA React (Vite + @auth0/auth0-react)
│   ├── .env.example                 Plantilla de variables (copiar a .env.local)
│   └── src/{main.jsx, App.jsx}
├── deploy/
│   ├── enterprise-risk-api.service  Unidad systemd del backend
│   ├── enterprise-risk.conf         VirtualHost de Apache
│   └── duck.sh.example              Script de actualización de Duck DNS
└── docs/                            Capturas de evidencia
```

## Autenticación vs. autorización

- **Autenticación**: verificar *quién* es el usuario. La realiza **Auth0** (login, contraseñas, MFA, etc.); el frontend nunca maneja credenciales.
- **Autorización**: decidir *qué puede hacer*. La realiza el **backend**: solo permite `/api/**` si el JWT es válido, fue emitido por nuestro tenant y está dirigido a nuestra API (`aud`). Todo lo demás se deniega (`denyAll`), excepto `/actuator/health`.

---

## Parte 1. Configuración de Auth0

1. Crear un tenant en Auth0 (si no existe).
2. **Applications → Create Application → Single Page Application** con nombre `Enterprise Risk Lab Frontend`.
3. **APIs → Create API**:
   - Nombre: `Enterprise Risk Lab API`
   - Identifier (audience): `https://enterprise-risk-api`
   - Signing algorithm: `RS256`
4. En la configuración de la SPA:

   | Campo | Valor |
   |---|---|
   | Allowed Callback URLs | `http://localhost:5173,https://YOUR-SUBDOMAIN.duckdns.org` |
   | Allowed Logout URLs | `http://localhost:5173,https://YOUR-SUBDOMAIN.duckdns.org` |
   | Allowed Web Origins | `http://localhost:5173,https://YOUR-SUBDOMAIN.duckdns.org` |

5. Registrar `AUTH0_DOMAIN`, `AUTH0_CLIENT_ID` y `AUTH0_AUDIENCE`. **Nunca** usar el *client secret* en el navegador.

## Parte 2. Backend (ejecución local)

Requisitos: Java 21.

1. Editar `backend/src/main/resources/application.properties` y reemplazar `YOUR_TENANT` por el dominio real del tenant (o exportar las variables de entorno equivalentes).
2. Ejecutar:

   ```bash
   cd backend
   ./mvnw spring-boot:run        # Windows: mvnw.cmd spring-boot:run
   ```

3. Verificar que sin token la API responde **401**:

   ```bash
   curl -i http://localhost:8080/api/risk-status
   # HTTP/1.1 401
   curl http://localhost:8080/actuator/health
   # {"status":"UP"}
   ```

Pruebas automáticas (401 sin token, 200 con JWT, health público, rutas desconocidas denegadas):

```bash
./mvnw test
```

> Nota: el backend escucha solo en `127.0.0.1` (`server.address`), así que el puerto 8080 nunca queda expuesto directamente aunque el firewall falle.

## Parte 3. Frontend (ejecución local)

Requisitos: Node.js 20+.

```bash
cd frontend
cp .env.example .env.local    # completar con los valores de Auth0 (no se sube a git)
npm install
npm run dev
```

Abrir http://localhost:5173 → **Log in** → **Call protected API**. La respuesta esperada es:

```json
{
  "status": "protected",
  "subject": "auth0|...",
  "control": "JWT validation and authenticated access"
}
```

## Parte 4. Servidor y Duck DNS

1. En un servidor Linux propio: cambiar credenciales por defecto, crear un usuario no-root para desplegar y activar el firewall:

   ```bash
   sudo ufw default deny incoming
   sudo ufw allow OpenSSH
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw enable
   ```

2. Crear el subdominio en https://www.duckdns.org y mantener la IP actualizada con `deploy/duck.sh.example` (vía cron). Verificar: `nslookup YOUR-SUBDOMAIN.duckdns.org`.
3. En el proveedor cloud / router, permitir solo **TCP 80 y 443** de entrada. **No** exponer el 8080.
4. Instalar paquetes:

   ```bash
   sudo apt update
   sudo apt install -y apache2 certbot python3-certbot-apache openjdk-21-jre
   ```

5. Compilar en una máquina de confianza (con `VITE_API_URL=https://YOUR-SUBDOMAIN.duckdns.org` en `.env.local`):

   ```bash
   (cd frontend && npm run build)
   (cd backend && ./mvnw clean package)
   ```

6. Copiar los artefactos de forma segura:

   ```bash
   scp -r frontend/dist backend/target/enterprise-risk-api-0.0.1-SNAPSHOT.jar deploy/ usuario@servidor:~/
   ```

## Parte 5. Despliegue del backend

```bash
sudo mkdir -p /opt/enterprise-risk-api
sudo cp ~/enterprise-risk-api-0.0.1-SNAPSHOT.jar /opt/enterprise-risk-api/app.jar
sudo useradd --system --no-create-home riskapi
sudo chown -R riskapi:riskapi /opt/enterprise-risk-api

# editar YOUR_TENANT y YOUR-SUBDOMAIN antes de copiar
sudo cp ~/deploy/enterprise-risk-api.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now enterprise-risk-api
curl http://127.0.0.1:8080/actuator/health
```

## Parte 6. Apache y HTTPS

```bash
sudo mkdir -p /var/www/enterprise-risk-frontend
sudo cp -r ~/dist/* /var/www/enterprise-risk-frontend/
sudo a2enmod ssl proxy proxy_http rewrite headers

# editar YOUR-SUBDOMAIN antes de copiar
sudo cp ~/deploy/enterprise-risk.conf /etc/apache2/sites-available/
sudo a2ensite enterprise-risk.conf
sudo a2dissite 000-default.conf
sudo apachectl configtest
sudo systemctl reload apache2

sudo certbot --apache -d YOUR-SUBDOMAIN.duckdns.org
sudo certbot renew --dry-run
```

Certbot crea el VirtualHost `*:443` con el certificado y redirige HTTP → HTTPS. Finalmente, actualizar en Auth0 las Callback/Logout URLs y Web Origins con la dirección HTTPS.

## Parte 7. Lista de verificación

- [ ] Las peticiones sin autenticar a `/api/risk-status` devuelven **401**.
- [ ] Un usuario autenticado puede llamar a `/api/risk-status` desde el frontend.
- [ ] El backend valida el emisor (`issuer`) y la audiencia (`audience`) del JWT.
- [ ] La aplicación está disponible en `https://YOUR-SUBDOMAIN.duckdns.org`.
- [ ] El navegador muestra un certificado válido y el candado HTTPS.
- [ ] El puerto 8080 no es accesible desde Internet.
- [ ] `sudo certbot renew --dry-run` termina con éxito.

Comandos útiles de verificación:

```bash
curl -i https://YOUR-SUBDOMAIN.duckdns.org/api/risk-status          # 401
openssl s_client -connect YOUR-SUBDOMAIN.duckdns.org:443 \
  -servername YOUR-SUBDOMAIN.duckdns.org </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates                       # certificado Let's Encrypt
nc -zv -w 5 YOUR-SUBDOMAIN.duckdns.org 8080                          # debe fallar (desde fuera)
sudo journalctl -u enterprise-risk-api -n 50                          # logs de auditoría
```

## Evidencias

| Evidencia | Archivo |
|---|---|
| Llamada autenticada exitosa a la API | `docs/authenticated-call.png` |
| Certificado válido (navegador u `openssl`) | `docs/certificate.png` |

<!-- Descomentar cuando las capturas estén en docs/
![Llamada autenticada](docs/authenticated-call.png)
![Certificado válido](docs/certificate.png)
-->

## Reflexión: controles de riesgo

| Control | Riesgo que reduce | Evidencia |
|---|---|---|
| Autenticación con Auth0 | Suplantación de identidad y manejo débil de credenciales | Flujo de login de Auth0 e identidad del usuario |
| Validación del JWT en la API | Acceso no autorizado a la API | 401 sin un token válido |
| Validación de audiencia | Reutilización de tokens contra otra API | La API solo acepta tokens con su audiencia |
| HTTPS con certificado válido | Exposición de credenciales o tokens en tránsito | Detalles del certificado en el navegador |
| Reverse proxy con Apache | Exposición directa del puerto de la aplicación | Solo los puertos 80 y 443 son públicos |
| Logs centralizados del servicio | Investigación tardía de incidentes | Salida de `journalctl -u enterprise-risk-api` |

## Notas de seguridad

- Nunca subir `.env.local`, credenciales del servidor, secretos de Auth0 ni tokens de Duck DNS (están en `.gitignore`).
- La SPA usa solo el **client ID** de Auth0, nunca un *client secret*.
- CORS restringido a los orígenes exactos de desarrollo y producción (`app.cors.allowed-origins`), solo método `GET`.
- Los access tokens son datos sensibles: no se muestran, no se registran en logs y no se almacenan manualmente (el log de auditoría registra solo el `sub`).
- Mínimo privilegio: el backend corre con el usuario de sistema `riskapi`, sin shell ni home. Aplicar actualizaciones del sistema operativo regularmente.

## Referencias

- [Auth0 React quickstart](https://auth0.com/docs/quickstart/spa/react)
- [Spring Security – JWT Resource Server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
- [Documentación de Certbot](https://eff-certbot.readthedocs.io/)
