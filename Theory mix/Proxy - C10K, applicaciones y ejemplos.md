# Proxy - C10K, applicaciones y ejemplos

Here a resume of personal notes gathered during the Study of [DevOps with docker](https://courses.mooc.fi/org/uh-cs/courses/devops-with-docker) course.

**_If you find any error, in the language, in the code or in the theory, please let me know._**

---

<!-- TOC -->
## Table of content
- [Proxy](#proxy)
  - [Table of content](#table-of-content)
- [Problema C10k](#problema-c10k)
  - [Qué es el problema C10k?](#qué-es-el-problema-c10k)
  - [Por qué es un problema?](#por-qué-es-un-problema)
- [Soluciones arquitectónicas](#soluciones-arquitectónicas)
  - [Modelo orientado a eventos](#modelo-orientado-a-eventos)
  - [Rol del proxy en la solución](#rol-del-proxy-en-la-solución)
  - [Historia y evolución](#historia-y-evolución)
    - [Por qué se usa un proxy?](#por-qué-se-usa-un-proxy)
    - [Tipos de proxies](#tipos-de-proxies)
  - [Proxies recomendados para C10k](#proxies-recomendados-para-c10k)
- [Aplicaciones en red](#aplicaciones-en-red)
  - [1. Proxy en la red de backend](#1-proxy-en-la-red-de-backend)
  - [2. Proxy en el frontend](#2-proxy-en-el-frontend)
  - [Representacion basica de red con proxy](#representacion-basica-de-red-con-proxy)
- [Cómo un proxy soluciona C10k](#cómo-un-proxy-soluciona-c10k)
- [Qué pasa si el hardware está saturado?](#qué-pasa-si-el-hardware-está-saturado)
- [C10k En microservicios](#c10k-en-microservicios)
  - [Cómo un proxy ayuda en C10k con microservicios](#cómo-un-proxy-ayuda-en-c10k-con-microservicios)
  - [Si a pesar del proxy, el sistema no soporta la carga:](#si-a-pesar-del-proxy-el-sistema-no-soporta-la-carga)
- [Architectura interna del codigo de un proxy](#architectura-interna-del-codigo-de-un-proxy)
  - [Arquitectura interna de un proxy](#arquitectura-interna-de-un-proxy)
    - [1. Recepción de la solicitud](#1-recepción-de-la-solicitud)
    - [2. Procesamiento de la solicitud](#2-procesamiento-de-la-solicitud)
    - [3. Enrutamiento y balanceo de carga](#3-enrutamiento-y-balanceo-de-carga)
    - [4. Recepción de la respuesta del backend](#4-recepción-de-la-respuesta-del-backend)
    - [5. Envío de la respuesta al cliente](#5-envío-de-la-respuesta-al-cliente)
  - [Código base de un proxy simple en Node.js](#código-base-de-un-proxy-simple-en-nodejs)
- [Balanceo de Carga en un Proxy](#balanceo-de-carga-en-un-proxy)
  - [Tipos de Balanceo de Carga](#tipos-de-balanceo-de-carga)
    - [1. Balanceo de carga basado en DNS (Round-robin DNS)](#1-balanceo-de-carga-basado-en-dns-round-robin-dns)
    - [2. Balanceo de carga en Capa 4 (TCP/UDP)](#2-balanceo-de-carga-en-capa-4-tcpudp)
    - [3. Balanceo de carga en Capa 7 (HTTP/HTTPS)](#3-balanceo-de-carga-en-capa-7-httphttps)
  - [Estrategias de Balanceo de Carga](#estrategias-de-balanceo-de-carga)
    - [1. Round-robin (Simple pero básico)](#1-round-robin-simple-pero-básico)
    - [2. Least Connections (Óptimo para tráfico irregular)](#2-least-connections-óptimo-para-tráfico-irregular)
    - [3. IP Hash (Permanencia del usuario en un backend)](#3-ip-hash-permanencia-del-usuario-en-un-backend)
    - [4. Weighted Round-robin (Distribución basada en capacidad del backend)](#4-weighted-round-robin-distribución-basada-en-capacidad-del-backend)
  - [Código de un Proxy con Balanceo de Carga en Node.js](#código-de-un-proxy-con-balanceo-de-carga-en-nodejs)
- [Ejemplo basico de reverse proxy con Nginx en red de contenedor](#ejemplo-basico-de-reverse-proxy-con-nginx-en-red-de-contenedor)
  - [Escenario](#escenario)
  - [Configuración del archivo `nginx.conf`:](#configuración-del-archivo-nginxconf)
    - [Esplicaccion dettallada del nginx.conf](#esplicaccion-dettallada-del-nginxconf)
  - [Archivo `docker-compose.yml`](#archivo-docker-composeyml)
  - [Pasos para implementar:](#pasos-para-implementar)

---

# Problema C10k
## Qué es el problema C10k?

El problema C10k se refiere al desafío de manejar _10.000 conexiones concurrentes en un servidor_, especialmente en el contexto de aplicaciones web o de red. Surgió en los años 90, cuando los servidores tradicionales basados _en modelos de hilos (threads) o procesos por conexión_ no escalaban bien a miles de conexiones simultáneas.

## Por qué es un problema?
- Consumo de memoria por cada conexión activa.
- Cambio de contexto costoso entre hilos o procesos.
- Bloqueo en operaciones de I/O, que ralentiza todo el sistema.
- Los modelos tradicionales (como Apache en modo prefork) no estaban diseñados para ese nivel de concurrencia.

---

# Soluciones arquitectónicas
## Modelo orientado a eventos
Una arquitectura basada en eventos y I/O no bloqueante (como la de Node.js o Nginx) permite manejar muchas conexiones con pocos recursos, ya que no se necesita un hilo por conexión.
> Ejemplo: Node.js usa el event loop y epoll (en Linux) para manejar miles de conexiones sin bloquear.

_Multiplexación de conexiones_: Herramientas como epoll, kqueue, IOCP permiten monitorear múltiples sockets desde un solo hilo, de manera eficiente.

## Rol del proxy en la solución

_Un proxy inverso puede actuar como primer punto de entrada_, gestionando conexiones y balanceando carga entre procesos o servicios. 

**Ventajas del uso de un proxy:**
- Termina conexiones lentas y habla con el backend más rápido.
- Hace balanceo de carga entre procesos/hilos/microservicios.
- Reduce la carga en el backend.
- Permite cacheo, compresión, y otras optimizaciones.


> Un **proxy** es un intermediario entre un cliente y un servidor que facilita la comunicación entre ambos sin que interactúen directamente. ___Su función principal es recibir solicitudes del cliente, procesarlas si es necesario y reenviarlas al servidor de destino.___ Luego, devuelve la respuesta del servidor al cliente.  

## Historia y evolución
Los proxies surgieron en la década de 1990 con el auge de la web y la necesidad de gestionar mejor el tráfico en redes empresariales. Inicialmente, se usaban para almacenamiento en caché y control de acceso. Con el tiempo, evolucionaron para incluir funciones avanzadas como inspección de paquetes, balanceo de carga y seguridad contra ataques cibernéticos. Hoy en día, los proxies juegan un papel clave en infraestructuras modernas como microservicios, redes empresariales y la computación en la nube. 

### Por qué se usa un proxy?
Los proxies se utilizan por diversas razones:  

1. **Seguridad y anonimato**: Ocultan la dirección IP del cliente, protegiendo la privacidad y evitando rastreos.  
2. **Control de acceso**: Se usan en redes empresariales y escolares para bloquear sitios web no permitidos.  
3. **Optimización del rendimiento**: Los proxies de caché almacenan respuestas para reducir la carga en el servidor y mejorar la velocidad.  
4. **Balanceo de carga**: Distribuyen el tráfico entre varios servidores para mejorar la escalabilidad.  
5. **Filtrado de contenido**: Inspeccionan y bloquean contenido malicioso o inapropiado.  
6. **Supervisión y auditoría**: Se usan para registrar y analizar el tráfico de una red.  

### Tipos de proxies
- **Proxy directo**: Intermedia entre el cliente y el servidor sin modificar la solicitud.  
- **Proxy inverso**: Se coloca frente a un servidor, gestionando el tráfico entrante, mejorando seguridad y rendimiento.  
- **Proxy transparente**: No altera la solicitud ni la respuesta, el usuario no nota su presencia.  
- **Proxy anónimo**: Oculta la identidad del cliente sin revelar que se está usando un proxy.  
- **Proxy de caché**: Guarda copias de respuestas para reducir la latencia y la carga en el servidor.  

## Proxies recomendados para C10k
1. **NGINX**:  
   - Arquitectura basada en eventos (Epoll/kqueue).  
   - Ideal para servidores HTTP y balanceo de carga.  
2. **HAProxy**:  
   - Optimizado para alto rendimiento y baja latencia.  
   - Soporte para balanceo de carga en TCP y HTTP.  
3. **Envoy Proxy**:  
   - Diseñado para microservicios.  
   - Conexiones concurrentes eficientes y observabilidad avanzada.  
4. **Traefik**:  
   - Orientado a contenedores y Kubernetes.  
   - Buen manejo de tráfico dinámico.  

---

# Aplicaciones en red

Depende del objetivo, pero un **proxy para C10k** generalmente se aplica en la **red de backend** para gestionar conexiones eficientes entre clientes y servidores, especialmente en arquitecturas de microservicios. Sin embargo, también puede estar en el **frontend** en ciertos casos.  

## 1. Proxy en la red de backend
(más común) Se usa para balanceo de carga, caché, seguridad y escalabilidad en los servicios backend.  

- **Ejemplo**:  
  - Un **API Gateway** (como NGINX, HAProxy, Envoy) gestiona solicitudes hacia microservicios.  
  - Redis actúa como proxy de caché para reducir la carga de la base de datos.  
  - HAProxy distribuye tráfico entre múltiples instancias de backend.  

## 2. Proxy en el frontend
(Menos común) pero útil cuando el frontend necesita optimizar conexiones o protegerse de tráfico malicioso.  

- **Ejemplo**:  
  - Un proxy inverso en el frontend (como NGINX o Varnish) puede servir contenido estático con caché.  
  - Un CDN con proxy (Cloudflare, Fastly) reduce la latencia al acercar los datos a los usuarios.  
  - Un WebSocket proxy (como Traefik o Envoy) puede manejar conexiones en tiempo real.  

## Representacion basica de red con proxy

Aquí tienes una representación de una red con proxy:  

```
+-------------+       +-------------+       +--------------------+       +-------------+
|   Cliente   | --->  |    Proxy    | --->  |   Servidor Web     | --->  |   Cliente    |
+-------------+       +-------------+       +--------------------+       +-------------+
       |                     |                       |                      |
       | Solicitud HTTP/     | Procesa solicitudes,  | Recursos estáticos,  | Respuesta
       | HTTPS               | Filtrado, Seguridad, | dinámicos y API/     | procesada
       |                     | Caché, Anonimato,    | Servicios            | y optimizada
       |                     | Balanceo de carga    |                      |
       v                     v                       v                      v

```

En esta estructura:  
- Los **clientes**: envían solicitudes a través del proxy. Inicia solicitudes HTTP/HTTPS para acceder a recursos web. 
- El **proxy**: Filtra solicitudes, protege la seguridad, oculta la identidad del cliente (anonimato), almacena en caché respuestas recurrentes y distribuye las solicitudes entre múltiples servidores (balanceo de carga).  
- El **servidor web**: Procesa las solicitudes entregando recursos estáticos (imágenes, archivos), dinámicos (contenido generado en tiempo real) y puntos de conexión de API/Servicios. 
- El **Cliente**: Recibe la respuesta final, optimizada por el proxy.

---

# Cómo un proxy soluciona C10k

Un proxy como **NGINX, HAProxy o Envoy** usa técnicas avanzadas para manejar muchas conexiones sin colapsar:  

| Técnica                         | Qué hace                                                                 | Beneficio principal                                         | Ejemplo / Estrategia                                       |
|---------------------------------|-------------------------------------------------------------------------|-------------------------------------------------------------|------------------------------------------------------------|
| **E/S basada en eventos**       | Los servidores tradicionales como Apache crean un **hilo o proceso por conexión**, lo que consume mucha RAM y CPU. Proxies modernos usan **event loops** (Epoll en Linux, Kqueue en BSD/macOS), manejando **miles de conexiones con pocos hilos**. | Maneja miles de conexiones con menos consumo de RAM y CPU.  | Epoll (Linux), Kqueue (BSD/macOS).                        |
| **Multiplexación de conexiones**| Agrupa conexiones de clientes en pocas hacia el backend.                | Reduce la carga del backend y mejora la eficiencia.         | 10,000 clientes → 500 conexiones reales al backend.       |
| **Caché de respuestas**         | Guarda respuestas comunes para servirlas sin consultar el backend. Si muchas solicitudes piden lo mismo, el proxy responde sin consultar el backend.      | Descarga el backend y acelera respuestas al cliente.        | Usa caché de respuestas para peticiones recurrentes.       |
| **Balanceo de carga**           | Distribuye conexiones entre múltiples servidores backend.               | Evita la saturación de un solo servidor.                    | Round-robin, Least Connections, IP Hash, etc.             |
| **Limitación y filtrado**       | Bloquea tráfico malicioso o limita conexiones por IP.                   | Protege recursos y previene abusos. Evita que un solo usuario consuma demasiados recursos. | Reglas por IP o detección de tráfico malicioso.            |

---

# Qué pasa si el hardware está saturado?  
Si el **nivel físico está agotado** (CPU, RAM, ancho de banda), el proxy también sufrirá. En ese caso:  

1. **Optimización del kernel**  
- Ajustar `ulimit` y `sysctl` en Linux para aumentar conexiones simultáneas.  
- Configurar SO_REUSEPORT y TCP_FASTOPEN para mejorar eficiencia.  

2. **Escalado horizontal**  
- Desplegar múltiples instancias del proxy y balancear carga entre ellas.  
- Usar una CDN o Anycast para distribuir tráfico globalmente.  

3. **Offloading de SSL/TLS**  
- Si el proxy maneja TLS, usar aceleración hardware o delegar TLS a un CDN.  

4. **Optimizar backend**  
- Usar bases de datos escalables (Redis, Cassandra).  
- Implementar colas de mensajes (Kafka, RabbitMQ).  

> El proxy **reduce la carga del backend**, pero si el hardware está al límite, la solución es escalar horizontalmente con más proxies y optimizar el sistema. ¿Tienes un caso específico en mente (microservicios, WebSockets, API Gateway)?

---

# C10k En microservicios

Si el problema **C10k** ocurre en una arquitectura de **microservicios**, el proxy ayuda de varias maneras, pero también es necesario optimizar todo el sistema.  

## Cómo un proxy ayuda en C10k con microservicios

1. **API Gateway (Proxy inverso centralizado)**  
- Un **API Gateway** (NGINX, Envoy, Kong) recibe todas las peticiones y las distribuye a los microservicios.  
- Reduce la cantidad de conexiones directas a los servicios backend.  
- Implementa **rate limiting** para evitar sobrecarga.  

2. **Balanceo de carga entre microservicios**  
- Un proxy como **HAProxy o Envoy** distribuye las conexiones entre múltiples instancias de un servicio.  
- Previene la sobrecarga en una sola instancia y permite escalar horizontalmente.  

3. **Multiplexación y keep-alive**  
- Usa conexiones persistentes y multiplexación HTTP/2 para reducir la cantidad de conexiones al backend.  
- Ejemplo: 10,000 clientes → solo 500 conexiones al backend gracias a reuso de conexiones.  

4. **Caché de respuestas**  
- Un proxy como **Varnish o NGINX** almacena respuestas y evita que cada petición golpee el backend.  
- Útil para microservicios que devuelven datos estáticos o de poca actualización.  

5. **Optimización de WebSockets**  
- Si los microservicios usan WebSockets, el proxy puede manejar miles de conexiones de manera eficiente.  
- **NGINX o Traefik** pueden actuar como WebSocket proxies para reducir la carga en los servicios backend.  

6. **Filtrado y protección contra sobrecarga**  
- Bloquea tráfico malicioso o limita conexiones desde una misma IP.  
- Implementa circuit breakers con **Envoy o Istio** para evitar fallos en cascada entre microservicios.  

## Si a pesar del proxy, el sistema no soporta la carga:  

- **Escalar horizontalmente**: Añadir más instancias de API Gateway y microservicios.  
- **Usar un Service Mesh**: Envoy + Istio para gestionar comunicación interna de microservicios.  
- **Optimizar el kernel**: Ajustar `sysctl`, `ulimit`, TCP tuning en Linux.  
- **Distribuir tráfico con CDN**: Cloudflare, AWS CloudFront para evitar que todo el tráfico llegue al backend.  
- **Descentralizar la caché**: Redis/Memcached en cada servicio en lugar de solo en el API Gateway.  

> El **proxy** en una arquitectura de microservicios **ayuda a mitigar** C10k, pero no es la única solución. También es clave escalar horizontalmente, optimizar el sistema operativo y usar estrategias como caché y circuit breakers.  

---

# Architectura interna del codigo de un proxy

Un **proxy** internamente tiene varias partes clave para gestionar conexiones de forma eficiente. Su arquitectura depende del propósito (proxy inverso, balanceador de carga, caché, etc.), pero en términos generales, sigue una estructura basada en **entrada-salida asíncrona y multiplexación de conexiones**.

## Arquitectura interna de un proxy
Un proxy sigue un flujo básico:  

### 1. Recepción de la solicitud
- Abre un **socket** y escucha conexiones entrantes en un puerto específico.  
- Usa **E/S no bloqueante** para manejar múltiples conexiones simultáneamente.  

### 2. Procesamiento de la solicitud
- **Parsea** la petición (HTTP, WebSocket, TCP, etc.).  
- Aplica **reglas de filtrado** (caché, seguridad, rate limiting).  
- Decide si responder directamente (caché) o reenviar la petición al servidor de destino.  

### 3. Enrutamiento y balanceo de carga
- Si es un proxy inverso, reenvía la solicitud al backend adecuado.  
- Si es un balanceador de carga, elige el mejor backend usando una estrategia (round-robin, least connections, etc.).  

### 4. Recepción de la respuesta del backend
- Recoge la respuesta del servidor de destino.  
- Puede modificarla antes de enviarla al cliente (por ejemplo, agregando headers de seguridad).  
- Almacena en caché si es necesario.  

### 5. Envío de la respuesta al cliente
- Devuelve la respuesta al cliente utilizando **E/S asíncrona**.  
- Mantiene conexiones persistentes para mejorar eficiencia (Keep-Alive, HTTP/2).  

---

## Código base de un proxy simple en Node.js
Este ejemplo implementa un **proxy HTTP básico** que reenvía las peticiones al servidor backend usando `http-proxy`.  

```javascript
const http = require('http');
const httpProxy = require('http-proxy');

const proxy = httpProxy.createProxyServer({});

// Servidor proxy
const server = http.createServer((req, res) => {
    console.log(`Redirigiendo petición: ${req.url}`); 
    // Enviar la solicitud al backend
    proxy.web(req, res, { target: 'http://localhost:5000' }, (err) => {
        console.error("Error en el proxy:", err);
        res.writeHead(500);
        res.end("Proxy error");
    });
});

// Escuchar en el puerto 8080
server.listen(8080, () => {
    console.log("Proxy escuchando en http://localhost:8080");
});
```
> <p style="color: red;  ">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>

**Explicación:**  
- El proxy escucha en **puerto 8080**.  
- Reenvía todas las peticiones al backend en **puerto 5000**.  
- Maneja errores si el backend falla.  

---

# Balanceo de Carga en un Proxy

El **balanceo de carga** permite distribuir solicitudes entre múltiples servidores backend para **evitar la sobrecarga de un solo nodo** y **aumentar la disponibilidad y escalabilidad**.  

Un **proxy con balanceo de carga** actúa como un intermediario que recibe solicitudes de clientes y las redirige a diferentes servidores backend siguiendo una estrategia específica.  

---

## Tipos de Balanceo de Carga

### 1. Balanceo de carga basado en DNS (Round-robin DNS)
   - Se configuran múltiples registros A/CNAME en el DNS para distribuir tráfico entre varias IPs.  
   - No es un balanceador real, solo reparte tráfico a nivel de resolución de nombres.  
   - No detecta fallos en los servidores.  

### 2. Balanceo de carga en Capa 4 (TCP/UDP)
   - Opera en el nivel de transporte (OSI) y distribuye tráfico según la IP de origen/destino y el puerto.  
   - Ejemplo: **HAProxy, IPVS, AWS ELB (nivel de transporte)**.  


### 3. Balanceo de carga en Capa 7 (HTTP/HTTPS)
   - Opera a nivel de aplicación, analizando peticiones HTTP.  
   - Puede aplicar reglas avanzadas (headers, cookies, URL).  
   - Ejemplo: **NGINX, Traefik, Envoy, AWS ALB**.  

**Flujo:**  
```
Cliente → Proxy (Balanceador HTTP/TCP o repartidor de trafico) → Servidor backend
```

---

## Estrategias de Balanceo de Carga  

### 1. Round-robin (Simple pero básico)
   - Cada nueva petición se redirige al siguiente servidor en la lista.  
   - No tiene en cuenta carga real del servidor.  

```nginx
upstream backend {
    server 192.168.1.1;
    server 192.168.1.2;
}
server {
    location / {
        proxy_pass http://backend;
    }
}
```

### 2. Least Connections (Óptimo para tráfico irregular)
   - Envía la nueva petición al servidor con **menos conexiones activas**.  
   - Mejora balanceo en tráfico dinámico.  

```nginx
upstream backend {
    least_conn;
    server 192.168.1.1;
    server 192.168.1.2;
}
```

### 3. IP Hash (Permanencia del usuario en un backend)
   - La misma IP del cliente siempre se dirige al mismo servidor.  
   - Útil en sesiones sin almacenamiento centralizado.  

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.1;
    server 192.168.1.2;
}
```

### 4. Weighted Round-robin (Distribución basada en capacidad del backend)
   - Se asigna más tráfico a servidores más potentes.  

```nginx
upstream backend {
    server 192.168.1.1 weight=3;
    server 192.168.1.2 weight=1;
}
```

---

## Código de un Proxy con Balanceo de Carga en Node.js
Aquí hay un ejemplo de un **proxy balanceador de carga en Node.js** usando `http-proxy`:  

```javascript
const http = require('http');
const httpProxy = require('http-proxy');

const proxy = httpProxy.createProxyServer({});
const servers = ['http://localhost:5001', 'http://localhost:5002']; // Backends
let current = 0; // Control de round-robin

const server = http.createServer((req, res) => {
    const target = servers[current];
    current = (current + 1) % servers.length; // Round-robin simple

    console.log(`Redirigiendo a: ${target}`);
    proxy.web(req, res, { target }, (err) => {
        console.error("Error en el proxy:", err);
        res.writeHead(500);
        res.end("Proxy error");
    });
});

server.listen(8080, () => {
    console.log("Balanceador en http://localhost:8080");
});
``` 
> <p style="color: red;  ">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>

> El **balanceo de carga** es clave para manejar tráfico masivo en microservicios y evitar colapsos. Se pueden usar **estrategias** como **round-robin, least connections e IP hash** según la necesidad. Para manejar **C10k**, el sistema debe estar optimizado y preparado para escalar.  

---

# Ejemplo basico de reverse proxy con Nginx en red de contenedor

Sigue un ejemplo práctico de cómo configurar un proxy reverso con Nginx que redirija solicitudes entre un contenedor de frontend y un backend en un entorno con Docker:

## Escenario
- Contenedor frontend escuchando en el puerto `3000`.
- Contenedor backend escuchando en el puerto `5000`.
- Nginx actúa como un proxy reverso que redirige las solicitudes según la ruta.

## Configuración del archivo `nginx.conf`:
Este archivo se usará dentro del contenedor de Nginx:

```nginx
server {
    listen 80;

    server_name localhost;

    # Redirigir tráfico a Frontend
    location / {
        proxy_pass http://frontend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Redirigir tráfico a Backend
    location /api/ {
        proxy_pass http://backend:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Esplicaccion dettallada del nginx.conf

Sigue una descripción detallada de cada línea del código de ariba:

```nginx
server {
    listen 80;
```
- **`listen 80;`**: Indica que este servidor Nginx escuchará las solicitudes HTTP en el puerto 80, que es el puerto estándar para tráfico web no cifrado.

```nginx
    server_name localhost;
```
- **`server_name localhost;`**: Define el nombre del servidor. En este caso, el nombre es `localhost`, lo que significa que Nginx procesará solicitudes dirigidas al host local (127.0.0.1).

```nginx
    location / {
```
- **`location / {`**: Esta directiva define un bloque de ubicación para la raíz (`/`). Todas las solicitudes que no especifiquen una ruta específica (como `/api/`) serán manejadas por este bloque.

```nginx
        proxy_pass http://frontend:3000;
```
- **`proxy_pass http://frontend:3000;`**: Redirige el tráfico que llega a la raíz (`/`) al contenedor llamado `frontend`, que está ejecutándose en el puerto 3000.

```nginx
        proxy_set_header Host $host;
```
- **`proxy_set_header Host $host;`**: Establece el encabezado HTTP `Host` de la solicitud redirigida. `$host` será reemplazado por el nombre del host que el cliente envió en la solicitud original.

```nginx
        proxy_set_header X-Real-IP $remote_addr;
```
- **`proxy_set_header X-Real-IP $remote_addr;`**: Añade un encabezado personalizado `X-Real-IP` en la solicitud redirigida. Este encabezado contiene la dirección IP del cliente que realizó la solicitud original.

```nginx
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```
- **`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`**: Añade el encabezado estándar `X-Forwarded-For` que incluye las direcciones IP de los clientes en caso de múltiples proxies en la cadena de red. Este encabezado ayuda a rastrear la ruta de la solicitud.

```nginx
    }
```
- **`}`**: Cierra el bloque de ubicación para la raíz `/`.

```nginx
    location /api/ {
```
- **`location /api/ {`**: Define otro bloque de ubicación, esta vez para las solicitudes que comiencen con `/api/`. Estas solicitudes serán redirigidas al backend.

```nginx
        proxy_pass http://backend:5000;
```
- **`proxy_pass http://backend:5000;`**: Redirige las solicitudes con la ruta `/api/` al contenedor llamado `backend`, que está ejecutándose en el puerto 5000.

```nginx
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```
- Estas tres líneas tienen la misma funcionalidad que se describió anteriormente para el bloque `/`. Establecen los encabezados HTTP adecuados para las solicitudes redirigidas al backend.

```nginx
    }
}
```
- **`}`**: Cierra el bloque del servidor y el bloque de ubicación `/api/`.

-----------------

## Archivo `docker-compose.yml`
Aquí tienes un ejemplo de cómo definir los servicios en un archivo `docker-compose`:

```yaml
version: '3.8'
services:
  frontend:
    image: frontend-image
    container_name: frontend
    ports:
      - "3000:3000"

  backend:
    image: backend-image
    container_name: backend
    ports:
      - "5000:5000"

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend
      - backend
```

## Pasos para implementar:
1. Crea el archivo `nginx.conf` en la misma carpeta que tu archivo `docker-compose.yml`.
2. Ejecuta los servicios con Docker Compose:
   ```bash
   docker-compose up -d
   ```
3. El tráfico será redirigido de la siguiente manera:
   - Solicitudes a `/` irán al contenedor `frontend`.
   - Solicitudes a `/api/` irán al contenedor `backend`.