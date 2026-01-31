# Q_Framework

Framework PHP ligero y modular para construir aplicaciones web y APIs con un núcleo pequeño, **routing potente**, **DB/QueryBuilder/Model**, **módulos**, **seguridad**, y capa de **observabilidad** (logs/tracing/métricas).

> Este repo contiene el **core** (`/system`) + un *scaffold* de aplicación (`/app`) con módulos de ejemplo (`/app/Modules`).

---

## Características

- **Kernel HTTP** con `Request`/`Response` y soporte de **redirect**, **JSON**, y **captura de salida** de controladores.
- **Router** con:
  - rutas `get/post/put/patch/delete/any`
  - `group()`
  - `auto()` por prefijo (útil para carpetas de controllers)
  - rutas con nombre (`as`) y `route_url()` / `Routes::url()`
  - opción de **case sensitive** (configurable)
- **Controladores** con inyección de contexto (`Request`/`Response`) desde el kernel.
- **Vistas** (`View::render`) con layout opcional y vistas del sistema vía prefijo `_system/`.
- **Base de datos**:
  - `Connection` + `QueryBuilder`
  - `Model` estilo “active record” (tabla, PK, allowedFields, soft deletes, etc.)
  - CRUD dinámico por tabla con `System\Database\Crud\TableCrud`
  - **DbResult** para respuestas consistentes (`ok`, `data`, `rowCount`, `insertId`, `error`, `meta`, etc.)
- **Paginación** con `paginate()` + `pager()` y vista de paginación del sistema.
- **Validación** con `Validator` y helper `validate()` (según helpers habilitados).
- **Seguridad**:
  - CSRF (cookie o session) + `CsrfFilter`
  - `SecurityHeadersFilter` (headers seguros)
  - `RateLimiter` + `RateLimitFilter`
  - guard por ruta (sesión/rol) con `GuardFilter`
- **Módulos** en `app/Modules/<Modulo>/` con `Routes.php`, `Controllers/`, `Views/`, etc. (auto discover opcional).
- **Multi-tenant** (`Tenant`) con resolución por header/query/subdominio/path y carpetas `write/tenants/<tenant>/...`.
- **Observabilidad**:
  - métricas tipo Prometheus (`/_metrics`) con driver file o redis
  - tracer simple (`Tracer`)
- Helpers del sistema (`system/Helpers/*_helper.php`) + helpers de app (`app/Helpers/...`) sin permitir “shadowing” de helpers core.

---

## Requisitos

- **PHP 8.0+** (usa `match`, `str_contains`, `str_starts_with`, etc.).
- Extensiones recomendadas: `pdo_mysql` (o el driver que uses), `openssl`, `mbstring`, `json`.
- Servidor web: Apache (mod_rewrite) o Nginx.

---

## Estructura del proyecto

```
.
├─ app/
│  ├─ Config/
│  ├─ Controllers/
│  ├─ Filters/
│  ├─ Helpers/
│  │  └─ system/        # helpers del proyecto (files/validadores/formularios)
│  ├─ Modules/          # módulos (Admin, Api, Auth, Blog, Demo, Files, ...)
│  ├─ Providers/        # DI container / servicios infra
│  └─ Views/
└─ system/
   ├─ Core/             # Kernel, Router, Request, Response, View, Csrf, Tenant, ...
   ├─ Database/         # Connection, QueryBuilder, Model, Migrations, Crud
   ├─ Filters/
   ├─ Helpers/
   ├─ Observability/
   └─ Views/_system/    # errors, installer, pagers
```

> Nota: el framework espera una carpeta `write/` (o el `WRITEPATH` que definas) para cache/logs/sesiones/métricas, etc.

---

## Instalación rápida (modo proyecto)

1) Clona el repo:

```bash
git clone <tu-repo>
cd Q_Framework
```

2) Crea la carpeta `write/` y asigna permisos (Linux):

```bash
mkdir -p write
chmod -R 775 write
```

3) Copia y ajusta tu `.env`:

- `baseURL` / `BASE_URL`
- `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASS`
- `APP_ENV`, `APP_DEBUG`

4) Crea el *entrypoint* HTTP.

En esta versión del repo, el core está listo para ejecutarse, pero necesitas un `public/index.php` (o un index en tu webroot) que:

- defina constantes (`ROOTPATH`, `APPPATH`, `SYSTEMPATH`, `WRITEPATH`)
- cargue helpers/funciones base
- cargue `.env`
- ejecute el kernel

Ejemplo mínimo:

```php
<?php
// public/index.php

declare(strict_types=1);

define('ROOTPATH', dirname(__DIR__) . DIRECTORY_SEPARATOR);
define('APPPATH', ROOTPATH . 'app' . DIRECTORY_SEPARATOR);
define('SYSTEMPATH', ROOTPATH . 'system' . DIRECTORY_SEPARATOR);
define('WRITEPATH', ROOTPATH . 'write' . DIRECTORY_SEPARATOR);

require_once SYSTEMPATH . 'Support/Functions.php';

// Carga .env (si existe)
\System\Core\Env::load(ROOTPATH . '.env');

// Carga constantes de Config\Constants (opcional)
\System\Core\Constants::load();

// Boot tenant (si lo tienes habilitado)
\System\Core\Tenant::boot();

$app = new \System\Core\App();
$app->run();
```

5) Configura rewrite:

### Apache (.htaccess en `public/`)

```apache
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ index.php [L]
```

### Nginx (fragmento)

```nginx
location / {
  try_files $uri $uri/ /index.php?$query_string;
}
```

---

## Conceptos clave

### 1) Rutas

Las rutas viven en `app/Config/Routes.php` y también pueden venir desde módulos (`app/Modules/*/Routes.php`).

Ejemplo:

```php
$routes->get('/', 'Modules\\Blog\\Controllers\\Blog@index', ['as' => 'blog.index']);
$routes->post('/auth/login', 'Modules\\Auth\\Controllers\\Auth@loginPost', ['as'=>'auth.login_post']);
```

**Rutas con nombre**:

```php
$url = route_url('blog.index');
$url2 = route_url('blog.show', ['id' => 10]);
```

**Auto routes por prefijo** (útil cuando tienes una carpeta con múltiples controllers):

```php
$routes->auto('/funciones', 'App\\Controllers\\Funciones', [
  'defaultController' => 'Home',
  'defaultMethod'     => 'index',
]);
```

### 2) Controladores

- Puedes extender `System\Core\Controller` o `App\Controllers\BaseController`.
- El kernel inyecta `Request` y `Response`.

Ejemplo simple:

```php
namespace Modules\Api\Controllers;

use System\Core\Controller;

final class Status extends Controller
{
  public function index()
  {
    return $this->response->json(['ok' => true, 'time' => time()]);
  }
}
```

### 3) Vistas

```php
return view('blog/index', ['title' => 'Blog', 'items' => $items]);
```

- App: `app/Views/blog/index.php`
- Sistema: `view('_system/errors/404')` -> `system/Views/_system/errors/404.php`

### 4) Base de datos

#### Connection / QueryBuilder

```php
$db = service('db');
$rows = $db->table('users')->where('active', 1)->get()->getResultArray();
```

#### Model

```php
use System\Database\Model;

final class UserModel extends Model
{
  protected string $table = 'users';
  protected string $primaryKey = 'id';
  protected array $allowedFields = ['name','email','role'];
}

$users = new UserModel();
$id = $users->insert(['name'=>'Ana','email'=>'ana@demo.com','role'=>'user']);
$user = $users->find($id);
```

#### CRUD dinámico por tabla (TableCrud)

Si no quieres crear un Model por tabla:

```php
use System\Database\Crud\TableCrud;

$crud = new TableCrud(service('db'), 'blog_posts', [
  'primaryKey'     => 'id',
  'protectFields'  => true,
  'allowedFields'  => ['title','slug','content','author_id'],
  'useTimestamps'  => true,
  'useSoftDeletes' => false,
]);

$r = $crud->createResult(['title'=>'Hola', 'slug'=>'hola', 'content'=>'...']);
if ($r->ok) {
  // $r->insertId
}
```

#### DbResult (PRO)

Varios métodos tienen versión `*Result()` para devolver un `DbResult` con:

- `ok` (bool)
- `action` (create/update/delete/select)
- `data` (fila o filas)
- `rowCount`
- `insertId`
- `error` (mensaje)
- `meta` (extra)

Esto ayuda a estandarizar respuestas (especialmente para APIs / fetch).

---

## Paginación

En `QueryBuilder` existe `paginate()` que **lee la página automáticamente** desde el query string (por defecto `?page=2`, configurable).

Ejemplo:

```php
$qb = service('db')->table('blog_posts')->orderBy('id','DESC');
$items = $qb->paginate(10);         // devuelve array de filas
$pager = $qb->pager();              // objeto paginator

return view('blog/index', [
  'items' => $items,
  'pager' => $pager,
]);
```

En la vista:

```php
echo $pager->links(); // usa el template por defecto (system/Views/_system/pagers/qfw_default.php)
```

> Recomendación DX: mantén `paginate()` + `pager()` como API única para paginación (evita mezclar “compat” con otra implementación).

---

## Filters (middleware)

Puedes proteger rutas con `filters`/`guard` en la definición:

```php
$routes->get('/demo/secure', 'Modules\\Demo\\Controllers\\Demo@secure', [
  'as' => 'demo.secure',
  'guard' => ['require'=>'user', 'redirect'=>'@auth.login', 'remember'=>true],
]);
```

Incluye filtros del sistema como:

- `csrf`
- `ratelimit`
- `security_headers`
- `guard`

Y filtros de app como:

- `feature` (ejemplo de feature flags)
- `auth` / `api_token` (según tu implementación)

---

## Módulos

Estructura recomendada:

```
app/Modules/Blog/
  Controllers/
  Models/
  Views/
  Routes.php
  Database/Migrations/
```

Si `Modules::autoDiscover` está habilitado, el framework detecta módulos automáticamente y registra:

- rutas del módulo (si existe `Routes.php`)
- rutas de vistas del módulo
- rutas de migrations del módulo

---

## Observabilidad

- **Métricas**: endpoint configurable (por defecto `/_metrics`).
- Driver: `file` o `redis` (según config `Observability`).

Esto permite tener contadores/histogramas simples sin acoplarte a un vendor pesado.

---

## CLI (Maker / herramientas)

El core incluye clases en `system/CLI` (Maker/Tools/Console). Si quieres un ejecutable tipo `bin/console`, puedes crear un script simple que cargue el bootstrap y llame al runner.

---

## Contribuir

1. Haz fork del repo
2. Crea una rama `feature/<nombre>`
3. Agrega pruebas/manuales de verificación (si aplica)
4. Envía PR
---

## Licencia

Este proyecto está licenciado bajo la **MIT License**. Revisa el archivo `LICENSE` en la raíz del repositorio.

---

## Créditos

- Core: `system/`
- Scaffold + módulos: `app/`

