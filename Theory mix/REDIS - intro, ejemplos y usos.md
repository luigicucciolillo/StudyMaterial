# REDIS: intro, ejemplos y usos

Here a resume of personal notes gathered during the Study of [DevOps with docker](https://courses.mooc.fi/org/uh-cs/courses/devops-with-docker) course.

**_If you find any error, in the language, in the code or in the theory, please let me know._**

---

## Table of content
- [REDIS: intro, ejemplos y usos](#redis-intro-ejemplos-y-usos)
  - [Table of content](#table-of-content)
  - [1 Intro a Redis](#1-intro-a-redis)
    - [Características principales de Redis](#características-principales-de-redis)
    - [Por qué usarlo en una arquitectura backend?](#por-qué-usarlo-en-una-arquitectura-backend)
  - [2️ Redis como Caché con MongoDB: Mejora del Rendimiento](#2️-redis-como-caché-con-mongodb-mejora-del-rendimiento)
    - [Ventajas de usar Redis como caché](#ventajas-de-usar-redis-como-caché)
    - [Patrón Cache-Aside con Redis y MongoDB](#patrón-cache-aside-con-redis-y-mongodb)
    - [Ejemplo avanzado con Node.js, Express y Redis](#ejemplo-avanzado-con-nodejs-express-y-redis)
  - [3 Redis Pub/Sub: Comunicación en Tiempo Real entre Backend y Frontend](#3-redis-pubsub-comunicación-en-tiempo-real-entre-backend-y-frontend)
    - [Intro](#intro)
      - [Cómo funciona Redis Pub/Sub?](#cómo-funciona-redis-pubsub)
      - [Ventajas](#ventajas)
      - [Casos de uso](#casos-de-uso)
    - [Comparaccion con pipes y sockets para settear comunicaciones entre entidades](#comparaccion-con-pipes-y-sockets-para-settear-comunicaciones-entre-entidades)
      - [Summary:](#summary)
    - [Caso de Uso: Notificaciones de Pedidos en un E-commerce](#caso-de-uso-notificaciones-de-pedidos-en-un-e-commerce)
    - [Implementación con Redis Pub/Sub](#implementación-con-redis-pubsub)
      - [Backend (Publicador)](#backend-publicador)
      - [Frontend (Suscriptor con WebSockets)](#frontend-suscriptor-con-websockets)
      - [Ventajas del enfoque Pub/Sub:](#ventajas-del-enfoque-pubsub)
  - [4 Autenticación con Redis: Manejo Seguro de Sesiones](#4-autenticación-con-redis-manejo-seguro-de-sesiones)
    - [Implementación con JWT y Redis](#implementación-con-jwt-y-redis)
    - [An extract from the official website of REDIS about vulnerability and techinques to solve](#an-extract-from-the-official-website-of-redis-about-vulnerability-and-techinques-to-solve)
    - [Flujo Técnico: Inicio de Sesión](#flujo-técnico-inicio-de-sesión)
    - [Validación de Tokens](#validación-de-tokens)
    - [Cierre de Sesión](#cierre-de-sesión)
    - [Diagrama del Flujo](#diagrama-del-flujo)
  - [5 Implementación con Docker y Docker Compose](#5-implementación-con-docker-y-docker-compose)
    - [`docker-compose.yml` con Redis y Backend](#docker-composeyml-con-redis-y-backend)
      - [descriccion](#descriccion)

---

## 1 Intro a Redis

Redis, que significa **REmote DIctionary Server**, es una base de datos NoSQL de código abierto que almacena datos en memoria en lugar de en disco, lo que la hace extremadamente rápida. Es ampliamente utilizada como **caché**, **cola de mensajes**, **almacén de datos clave-valor** y más, garantizando muy flexibilidad de applicacion.

### Características principales de Redis
1. **Almacenamiento en memoria**:
   - Redis guarda los datos en la RAM, lo que permite tiempos de respuesta ultrarrápidos.
   - También ofrece opciones de persistencia para guardar datos en disco si es necesario.

2. **Estructuras de datos avanzadas**:
   - Redis no solo maneja pares clave-valor simples, sino también estructuras como listas, conjuntos, hashes, y más.

3. **Alta disponibilidad**:
   - Soporta replicación de datos entre servidores para garantizar disponibilidad y tolerancia a fallos.

4. **Escalabilidad**:
   - Redis permite manejar millones de solicitudes por segundo, ideal para aplicaciones de alto rendimiento.

5. **Casos de uso comunes**:
   - **Caché**: Almacena datos temporalmente para acelerar el acceso.
   - **Colas de mensajes**: Gestiona tareas en segundo plano.
   - **Sesiones**: Almacena sesiones de usuario en aplicaciones web.
   - **Análisis en tiempo real**: Procesa datos rápidamente para análisis instantáneos.

### Por qué usarlo en una arquitectura backend?

- **Clave-valor en memoria**: Todos los datos se almacenan en RAM, lo que permite tiempos de acceso de **microsegundos**.  
- **Persistencia opcional**: Redis permite guardar datos en disco para evitar pérdidas tras un reinicio.  
- **Estructuras de datos avanzadas**: No solo almacena strings, sino también **listas, conjuntos, hashes, geolocalización, streams, etc.**  
- **Pub/Sub**: Redis permite enviar mensajes en tiempo real a múltiples clientes suscritos a un canal.  
- **Cluster y replicación**: Soporta **replicación maestra-esclava** y **distribución horizontal** de datos.  

---

## 2️ Redis como Caché con MongoDB: Mejora del Rendimiento

### Ventajas de usar Redis como caché

- **Reduce la carga en MongoDB**: Las consultas repetitivas se sirven desde la memoria en lugar de la base de datos.  
- **Disminuye la latencia**: Responder con Redis es hasta **100 veces más rápido** que una consulta en MongoDB.  
- **Optimiza costos**: Menos consultas a la base de datos significa menor consumo de CPU y RAM en los servidores de MongoDB.  

### Patrón Cache-Aside con Redis y MongoDB

- **Paso 1:** El backend consulta Redis.  
- **Paso 2:** Si los datos están en caché (**cache hit**), se devuelven de inmediato.  
- **Paso 3:** Si los datos **no están** en caché (**cache miss**), se consulta MongoDB.  
- **Paso 4:** Se guarda en Redis con una expiración definida (ejemplo: 10 minutos).  

### Ejemplo avanzado con Node.js, Express y Redis

```javascript
const mongoose = require('mongoose');
const redis = require('redis');
const express = require('express');
const util = require('util');
const app = express();
const client = redis.createClient({ host: 'redis', port: 6379 });
  client.get = util.promisify(client.get); // Convertir get() en función asíncrona
    mongoose.connect('mongodb://localhost:27017/tienda', {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
  const Product = mongoose.model('Product', new mongoose.Schema({ name: String, price: Number }));
  app.get('/products', async (req, res) => {
    const cacheKey = 'products_all';
    try {
      // Intentar recuperar datos desde Redis
      const cachedData = await client.get(cacheKey);
      if (cachedData) {
        console.log('🔵 Datos obtenidos de Redis');
        return res.json(JSON.parse(cachedData));
      }
      console.log('🟠 Datos obtenidos de MongoDB');
      const products = await Product.find();
      // Almacenar en Redis con una expiración de 10 minutos (600 segundos)
      client.setex(cacheKey, 600, JSON.stringify(products));
      res.json(products);
    } catch (error) {
      console.error(error);
      res.status(500).send('Error en el servidor');
    }
  });
app.listen(3000, () => console.log('🚀 Servidor en puerto 3000'));
```
> <p style="color: red;  ">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>

---

## 3 Redis Pub/Sub: Comunicación en Tiempo Real entre Backend y Frontend

### Intro
Redis Pub/Sub (Publicar/Suscribir) es un sistema de mensajería en tiempo real que permite la comunicación eficiente entre el backend y el frontend. Este modelo sigue el paradigma de publicación y suscripción, donde los emisores (publicadores) y receptores (suscriptores) están desacoplados, lo que significa que no necesitan conocerse entre sí.

#### Cómo funciona Redis Pub/Sub?

Claro, aquí está tu contenido reformulado:

- **Canales**: Los "canales" son utilizados para enviar y recibir mensajes. Los publicadores envían información a un canal específico, y los suscriptores que están vinculados a ese canal reciben el mensaje automáticamente.
- **Publicadores**: Encargados de enviar mensajes a los canales correspondientes, los publicadores no necesitan conocer cuántos suscriptores hay ni detalles sobre ellos.
- **Suscriptores**: Los suscriptores se conectan a uno o más canales y reciben mensajes en tiempo real cada vez que se publica contenido en esos canales.
- **Redis como intermediario**: Redis funciona como un puente entre los publicadores y los suscriptores, asegurando que los mensajes se distribuyan correctamente de los emisores a los receptores. 

Ejemplo de uso en tiempo real:

- **Backend**: El backend se encarga de publicar eventos en un canal específico de Redis. Estos eventos pueden incluir actualizaciones de datos, notificaciones o cualquier otra información que necesite ser compartida con el frontend.
- **Frontend**: El frontend, por su parte, se suscribe al canal correspondiente de Redis. De esta forma, recibe los eventos en tiempo real y actualiza la interfaz de usuario dinámicamente, sin necesidad de recargar la página.

#### Ventajas
**Baja latencia**: Redis Pub/Sub es extremadamente rápido, ideal para aplicaciones en tiempo real.
**Escalabilidad**: Permite manejar múltiples publicadores y suscriptores sin problemas.
**Desacoplamiento**: Los publicadores y suscriptores no necesitan interactuar directamente, lo que simplifica la arquitectura.

#### Casos de uso
**Notificaciones en tiempo real**: Como alertas en aplicaciones web.
**Chats en vivo**: Donde los mensajes se transmiten instantáneamente entre usuarios.
**Actualización de dashboards**: Para mostrar métricas o datos en tiempo real.

### Comparaccion con pipes y sockets para settear comunicaciones entre entidades

Here’s a comparison of **Pub/Sub**, **Pipes**, and **Sockets**, highlighting their differences and use cases:

| **Feature**            | **Pub/Sub**                                                                 | **Pipes**                                                                 | **Sockets**                                                                 |
|-------------------------|-----------------------------------------------------------------------------|---------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| **Definition**          | Publish/Subscribe is a messaging pattern where publishers send messages to channels, and subscribers receive them. | Pipes are a unidirectional communication mechanism between processes.     | Sockets are endpoints for bidirectional communication between systems or processes. |
| **Communication Type**  | Many-to-many (M:N). Publishers and subscribers are decoupled.              | One-to-one. Data flows in a single direction (e.g., producer to consumer). | One-to-one or one-to-many. Supports bidirectional communication.            |
| **Broker/Intermediary** | Requires a broker (e.g., Redis, Kafka) to manage message distribution.     | No intermediary; direct communication between processes.                  | No broker required; direct communication between endpoints.                 |
| **Persistence**         | Messages can be persisted by the broker for later delivery.                | No persistence; data is transient.                                        | No persistence; data is transient unless explicitly handled.                |
| **Use Cases**           | Real-time notifications, chat systems, event-driven architectures.         | Inter-process communication (IPC) within the same machine.                | Network communication (e.g., client-server models, real-time apps).         |
| **Topology**            | Decoupled. Publishers and subscribers don’t need to know each other.       | Tight coupling between processes.                                         | Flexible; can be tightly or loosely coupled.                                |
| **Asynchronous**        | Yes, supports asynchronous communication.                                 | Typically synchronous.                                                    | Can be synchronous or asynchronous.                                         |
| **Example Technologies**| Redis Pub/Sub, Kafka, MQTT.                                                | Unix Pipes, Named Pipes (Windows).                                        | TCP/IP Sockets, WebSockets, ZeroMQ.                                         |

#### Summary:
- **Pub/Sub**: Best for decoupled, real-time communication where multiple subscribers need to receive updates from publishers.
- **Pipes**: Ideal for simple, direct communication between processes on the same machine.
- **Sockets**: Versatile for network communication, supporting both local and remote connections.

### Caso de Uso: Notificaciones de Pedidos en un E-commerce

- Cuando el estado de un pedido cambia en MongoDB, se notifica automáticamente al frontend sin necesidad de hacer peticiones constantes (**polling**).  

### Implementación con Redis Pub/Sub

#### Backend (Publicador)
```javascript
const redis = require('redis');
const publisher = redis.createClient();
  const actualizarEstadoPedido = (idPedido, nuevoEstado) => {
    const mensaje = JSON.stringify({ idPedido, estado: nuevoEstado });
    publisher.publish('pedidos', mensaje);
  };
```
> <p style="color: red;  ">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>


#### Frontend (Suscriptor con WebSockets)
```javascript
const redis = require('redis');
const subscriber = redis.createClient();
  subscriber.subscribe('pedidos');
subscriber.on('message', (channel, message) => {
  console.log(`📢 Notificación en ${channel}: ${message}`);
});
```
> <p style="color: red;">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>

#### Ventajas del enfoque Pub/Sub:
- Se eliminan peticiones constantes al backend.  
- Las actualizaciones llegan **en tiempo real** al usuario.  
- Reduce la carga del servidor y la base de datos.  

---

## 4 Autenticación con Redis: Manejo Seguro de Sesiones

Al combinar Redis con tokens como JWT (JSON Web Tokens), se logra una arquitectura escalable y eficiente para autenticar y gestionar accesos.

### Implementación con JWT y Redis 
**1. JWT (JSON Web Token):**
Los JWT son cadenas de texto codificadas que contienen datos sobre el usuario, normalmente divididos en tres partes:
- Header: Tipo de token y algoritmo de codificación (ej., HMAC SHA256).
- Payload: Información del usuario, como ID o roles.
- Signature: Se utiliza para verificar que el token no ha sido manipulado.
Los JWT son emitidos por el backend y son almacenados por el cliente (generalmente en cookies o local storage).

**2. Redis como almacén de sesiones**:
Redis se utiliza para guardar información clave sobre la sesión del usuario:
- Mapeo entre el ID de usuario y el JWT.
- Almacenamiento de datos temporales como estados de autenticación.
- Configuración de expiración para tokens y sesiones usando TTL (Time to Live).

**3. Flujo típico**:
Inicio de sesión:
- El cliente envía credenciales al backend.
- El backend genera un JWT y lo envía al cliente.
- El backend almacena una referencia de la sesión en Redis para verificar futuros accesos.

Autenticación:

- El cliente envía el JWT en las solicitudes.
- El backend valida el JWT usando Redis (por ejemplo, verifica si el token sigue siendo válido).
Cierre de sesión:
- Redis elimina el registro de la sesión y el token asociado.

**4. Ventajas**:
- Rapidez: Redis es extremadamente eficiente debido a su almacenamiento en memoria.
- Escalabilidad: Puede manejar miles de sesiones simultáneamente.
- Seguridad: Los datos sensibles pueden ser configurados para expirar automáticamente, evitando accesos no autorizados.

---

### An extract from the official website of REDIS about vulnerability and techinques to solve

[...] **_So why is JWT dangerous for user authentication?_** from redis offical doc: [JSON Web Tokens (JWT) are Dangerous for User Sessions—Here’s a Solution](https://redis.io/blog/json-web-tokens-jwt-are-dangerous-for-user-sessions/)


The biggest problem with JWT is the token revoke problem. Since it continues to work until it expires, the server has no easy way to revoke it.

Below are some use cases that’d make this dangerous.
- Logout doesn’t really log you out!

Imagine you logged out from Twitter after tweeting. You’d think that you are logged out of the server, but that’s not the case. Because JWT is self-contained and will continue to work until it expires. This could be 5 minutes or 30 minutes or whatever the duration that’s set as part of the token. So if someone gets access to that token during that time, they can continue to access it until it expires.

- Blocking users doesn’t immediately block them.

Imagine you are a moderator of Twitter or some online real-time game where real users are using the system. And as a moderator, you want to quickly block someone from abusing the system. You can’t, again for the same reason. Even after you block, the user will continue to have access to the server until the token expires.

- Could have stale data

Imagine the user is an admin and got demoted to a regular user with fewer permissions. Again this won’t take effect immediately and the user will continue to be an admin until the token expires.

- JWT’s are often not encrypted so anyone able to perform a man-in-the-middle attack and sniff the JWT now has your authentication credentials. This is made easier because the MITM attack only needs to be completed on the connection between the server and the client. [...]

---

### Flujo Técnico: Inicio de Sesión
Código de ejemplo (Node.js):
- Usuario envía credenciales al backend.
- Verifica las credenciales.
- Genera un token JWT.
- Almacena el JWT en Redis con un TTL (tiempo de expiración).

```javascript
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const redis = require('redis');
// Crear cliente Redis
const client = redis.createClient();
// Configuración
const SECRET_KEY = 'tu-clave-secreta';
const TOKEN_EXPIRATION = 3600; // 1 hora
// Simula inicio de sesión
async function login(username, password) {
  // Supongamos que el usuario está almacenado con hash de contraseña
  const user = {
    username: 'testUser',
    passwordHash: await bcrypt.hash('password123', 10) // Ejemplo de hash
  };
  // Verifica la contraseña
  const isValid = await bcrypt.compare(password, user.passwordHash);
  if (!isValid) {
    throw new Error('Credenciales inválidas');
  }
  // Genera el JWT
  const token = jwt.sign({ username: user.username }, SECRET_KEY, {
    expiresIn: TOKEN_EXPIRATION
  });
  // Almacena el token en Redis
  client.setex(`token:${user.username}`, TOKEN_EXPIRATION, token);
  return token;
}
```
> <p style="color: red;">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>

### Validación de Tokens

Cada vez que el cliente realiza una solicitud, envía el JWT al servidor (generalmente a través de un encabezado Authorization: Bearer <token>). El servidor valida el token y verifica su existencia en Redis.
Código de validación:

```javascript
async function authenticate(token) {
  try {
    // Verifica la firma del token
    const payload = jwt.verify(token, SECRET_KEY);
    // Busca el token en Redis
    client.get(`token:${payload.username}`, (err, storedToken) => {
      if (err) throw err;
      if (storedToken !== token) {
        throw new Error('Token no válido o expirado');
      }
      console.log('Usuario autenticado:', payload.username);
    });
  } catch (err) {
    console.error('Error de autenticación:', err.message);
  }
}
```
> <p style="color: red;">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>

### Cierre de Sesión

Para cerrar sesión, basta con eliminar el token del almacén Redis:

```javascript
async function logout(username) {
  client.del(`token:${username}`, (err, response) => {
    if (response === 1) {
      console.log('Sesión cerrada exitosamente');
    } else {
      console.log('No se encontró sesión activa');
    }
  });
}
```
> <p style="color: red;">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>

### Diagrama del Flujo

```
[Cliente] -- Credenciales --> [Backend]
   |
   v
[Verificación de usuario] -- JWT --> [Redis]
   |
   v
[Cliente] -- Solicitud + JWT --> [Backend]
   |
   v
[Backend] -- Verifica JWT en Redis --> [Redis]
   |
   v
[Respuesta enviada al Cliente]
```

**Beneficios de este enfoque:**  
- Tokens con **expiración automática** en Redis.  
- Mayor seguridad, ya que no almacenamos tokens en base de datos.  
- Mejor escalabilidad y rendimiento.  

---

## 5 Implementación con Docker y Docker Compose  

Para simplificar la implementación, Redis y el backend pueden ejecutarse en **contenedores Docker**.  

### `docker-compose.yml` con Redis y Backend
```yaml
version: '3.8'

services:
  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - backend_network
  backend:
    build: ./backend
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - redis
    networks:
      - backend_network
networks:
  backend_network:
    driver: bridge
```

> <p style="color: red;">⚠️ Attention: Code generated with AI for academic pourpose. Please double-check it! ⚠️</p>

#### descriccion

Redis y el backend están configurados en contenedores independientes, cada uno ejecutándose de manera aislada dentro del sistema Docker. Para establecer comunicación entre ambos contenedores, se utiliza una red de Docker interna. Esta red asigna un hostname a cada contenedor que funciona como un identificador único dentro del entorno de Docker.

El backend accede a Redis utilizando el hostname redis. Este hostname es asignado automáticamente por Docker al contenedor de Redis cuando se crea dentro de la misma red. De esta manera, no es necesario conocer la dirección IP del contenedor, ya que Docker realiza la resolución de nombres internamente.

Desde el punto de vista técnico, la red interna de Docker funciona como una interfaz virtual que conecta los contenedores, similar a una VLAN. Esto permite el enrutamiento de paquetes entre los contenedores sin exponerse al exterior. Aquí se utiliza el protocolo TCP/IP para la comunicación, asegurando que las solicitudes y respuestas sean entregadas de manera confiable.