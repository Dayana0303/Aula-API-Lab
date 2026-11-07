# Tienda Aula API Lab (PHP)

Aplicación web que consume **Aula API Lab** mediante solicitudes HTTP y respuestas JSON: consulta productos (búsqueda, categoría, paginación, detalle), arma un carrito, crea pedidos y consulta el historial, con validaciones y manejo de errores de la API.

- **Backend:** PHP 8.1+ (cURL) — guarda la API key y es el único que habla con la API.
- **Frontend:** HTML + CSS + JavaScript puro (sin frameworks ni dependencias).
- **Sin Composer ni npm:** se ejecuta con el servidor integrado de PHP.

## Arquitectura y flujo de autenticación

```
Navegador (public/index.html + assets/app.js)
      │  fetch → api/products.php, api/orders.php ...   (sin API key)
      ▼
Backend PHP (public/api/*.php)
      │  valida entradas (src/Validator.php)
      │  AulaApiClient → HTTPS + X-API-Key + JSON
      ▼
Aula API Lab  https://darkred-rook-511555.hostingersite.com/api/v1
```

**Dónde se guarda la API key y cómo se evita exponerla**

1. Vive en la variable `AULA_API_KEY` del archivo `.env` (o del entorno del servidor). `.env` está en `.gitignore`, así que nunca llega a GitHub.
2. Solo la lee `config/config.php` y la usa `src/AulaApiClient.php`, que la envía en el encabezado `X-API-Key` (nunca en la URL).
3. El navegador solo conoce las rutas `api/*.php` de este mismo servidor; ninguna respuesta incluye la clave (hay una prueba automática que lo verifica).
4. El backend no confía en el cliente: el **total del pedido y el stock los calcula Aula API Lab**; el frontend solo envía `product_id` y `quantity`.

## Estructura

```
├── config/config.php        # lee .env / variables de entorno (secreto)
├── src/
│   ├── AulaApiClient.php    # cliente HTTP reutilizable (cURL + X-API-Key)
│   ├── ApiException.php     # error con status HTTP + código de la API
│   ├── Validator.php        # validación de pedidos, ids y parámetros
│   ├── http.php             # respuestas JSON, método permitido, manejo de errores
│   └── bootstrap.php
├── public/                  # única carpeta expuesta al navegador
│   ├── index.html
│   ├── assets/ (app.js, styles.css)
│   └── api/ health, products, product, categories, orders, order (.php)
├── tests/
│   ├── run.php              # 41 pruebas automáticas (API simulada)
│   ├── mock_api.php         # simulador de Aula API Lab
│   └── smoke.php            # prueba contra la API REAL con tu clave
└── docs/PRUEBAS.md          # checklist de pruebas de funcionamiento
```

## Instalación y ejecución

Requisitos: PHP 8.1 o superior con las extensiones `curl` y `mbstring`.

```bash
git clone <URL-DE-TU-REPO>
cd aula-api-lab-tienda
cp .env.example .env          # en Windows: copy .env.example .env
# editá .env y pegá tu API key (lab_...)
php -S localhost:8000 -t public
```

Abrí <http://localhost:8000>. Si la API key falta, la página muestra un aviso.

**Hosting (Apache / Hostinger):** apuntá el dominio a la carpeta `public/` y creá el `.env` un nivel arriba. Si no podés cambiar la raíz, el `.htaccess` incluido bloquea `.env`, `config/`, `src/` y `tests/`.

## Endpoints de la tienda (backend propio)

| Método | Ruta | Función | Endpoint de Aula API Lab |
|--------|------|---------|--------------------------|
| GET | `api/health.php` | Estado de la API y de la configuración | `/health.php` |
| GET | `api/products.php?q=&category=&page=&per_page=` | Listar, buscar, filtrar, paginar | `/products.php` |
| GET | `api/product.php?id=2` | Detalle de producto | `/product.php` |
| GET | `api/categories.php` | Categorías (derivadas del catálogo) | `/products.php` |
| POST | `api/orders.php` | Crear pedido (JSON) | `/orders.php` |
| GET | `api/orders.php?page=&per_page=` | Historial | `/orders.php` |
| GET | `api/order.php?id=15` | Detalle de un pedido | `/order.php` |

Cuerpo para crear un pedido:

```json
{ "customer_name": "María Solano", "notes": "Entregar en recepción",
  "items": [ { "product_id": 2, "quantity": 1 }, { "product_id": 4, "quantity": 2 } ] }
```

## Validaciones y manejo de errores

- **Entradas** (cliente y servidor): nombre 2–100 caracteres, notas ≤ 500, 1–20 líneas, cantidad entera 1–100 (y ≤ stock conocido en el carrito), `per_page` ≤ 50, ids enteros positivos.
- **Respuestas de la API:** se revisa el código HTTP, que el JSON sea válido y `success === true`.
- **Errores mostrados con mensajes claros** (nunca el error técnico crudo): `401` clave inválida, `404` producto/pedido no existe, `409` inventario insuficiente (se refresca el stock del carrito), `422` datos inválidos (errores por campo), `500/502/504` fallo de servidor, red o tiempo de espera.

## Pruebas

```bash
php tests/run.php            # 41 pruebas automáticas contra una API simulada
php tests/smoke.php          # pruebas de solo lectura contra la API real (usa tu .env)
php tests/smoke.php --order  # además crea 1 pedido real de 1 unidad
```

Ver el checklist completo en [`docs/PRUEBAS.md`](docs/PRUEBAS.md).

## Usar el cliente en otra aplicación PHP que ya tengas

Copiá estos archivos a tu proyecto y listo:

```
src/ApiException.php
src/AulaApiClient.php
src/Validator.php   (opcional, para validar pedidos)
```

```php
require 'ApiException.php';
require 'AulaApiClient.php';

$api = new AulaApiClient('https://darkred-rook-511555.hostingersite.com/api/v1', getenv('AULA_API_KEY'));

try {
    $productos = $api->listProducts(['q' => 'monitor', 'page' => 1])['data'];
    $pedido = $api->createOrder([
        'customer_name' => 'María Solano',
        'items' => [['product_id' => 2, 'quantity' => 1]],
    ])['data'];
} catch (ApiException $e) {
    echo $e->status . ' ' . $e->errorCode . ': ' . $e->getMessage(); // 401, 404, 409, 422...
}
```

Si tu app es JavaScript/otro lenguaje, podés dejar esta tienda corriendo como backend y llamar a sus rutas `api/*.php`.
