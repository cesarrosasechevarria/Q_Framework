# Q_Framework (QFW)

Microframework PHP **modular + multi‑tenant** (estilo CodeIgniter, pero liviano y orientado a “servicios”) para construir apps administrativas, APIs y portales con módulos.

> Repo: `cesarrosasechevarria/Q_Framework` (este README está pensado para usarse tal cual en GitHub).

---

## Tabla de contenido

- [Qué incluye](#qué-incluye)
- [Requisitos](#requisitos)
- [Descarga rápida](#descarga-rápida)
- [Instalación local (XAMPP)](#instalación-local-xampp)
- [Estructura de carpetas](#estructura-de-carpetas)
- [Configuración (.env y Config)](#configuración-env-y-config)
- [Arranque y front controller](#arranque-y-front-controller)
- [Rutas](#rutas)
  - [Parámetros tipados](#parámetros-tipados)
  - [Auto-routes por prefijo](#auto-routes-por-prefijo)
  - [Rutas con nombre](#rutas-con-nombre)
- [Controladores](#controladores)
- [Vistas y layouts](#vistas-y-layouts)
- [Servicios (`service()`)](#servicios-service)
- [Helpers](#helpers)
  - [Helpers del sistema](#helpers-del-sistema)
  - [Funciones globales core](#funciones-globales-core)
  - [Autoload de helpers (evita “undefined function”)](#autoload-de-helpers-evita-undefined-function)
- [Filters (middleware)](#filters-middleware)
- [Auth (guard de sesión)](#auth-guard-de-sesión)
- [Base de datos](#base-de-datos)
  - [QueryBuilder + paginate](#querybuilder--paginate)
  - [`DbResult` y métodos `*Result()`](#dbresult-y-métodos-result)
  - [CRUD por tabla (`TableCrud`)](#crud-por-tabla-tablecrud)
  - [Schema “ensure” (DDL)](#schema-ensure-ddl)
- [Migrations y Seeds](#migrations-y-seeds)
- [Módulos](#módulos)
  - [Auto‑discover](#auto-discover)
  - [Activar/desactivar por tenant (runtime)](#activardesactivar-por-tenant-runtime)
- [Multi‑tenancy](#multi-tenancy)
- [Cache](#cache)
- [Seguridad](#seguridad)
  - [CSRF](#csrf)
  - [Rate limiting](#rate-limiting)
  - [Security headers](#security-headers)
- [Observabilidad](#observabilidad)
- [Errores comunes / troubleshooting](#errores-comunes--troubleshooting)
- [Roadmap sugerido](#roadmap-sugerido)
- [Licencia](#licencia)

---

## Qué incluye

- Routing con:
  - rutas explícitas (`get/post/put/delete/any`)
  - **parámetros tipados** (`{id:int}`, `{slug:slug}`, `{path:*}`, `{id:re:\d{4}}`)
  - rutas con nombre + generador `route_url()`
  - auto‑routes por prefijo (útil para “carpetas” de controllers)
- Controladores con `Request` y `Response` inyectados.
- Vistas PHP simples + soporte de **layout**.
- Service locator `service('...')` y Container (Providers) para extender/override sin tocar core.
- DB: Connection + QueryBuilder + Model base + `TableCrud`.
- Paginación `paginate()` integrada al QueryBuilder.
- Filters/middleware: `csrf`, `guard`, `rate`, `security_headers`, `api_token`, etc.
- Módulos `app/Modules/*` con autodiscover y rutas propias.
- Multi‑tenancy con resolución por header/query/subdominio/path y **carpetas write por tenant**.
- Cache `file` o `redis` (tenant-aware).
- Observabilidad: métricas HTTP y filtro `obs`.

---

## Requisitos

- PHP 8.1+ (recomendado)
- Extensiones comunes: `pdo`, `pdo_mysql` (o el driver que uses), `mbstring`, `json`
- Servidor web (Apache/Nginx) o `php -S` para desarrollo
- MySQL/MariaDB (si usarás DB)

---

## Descarga rápida

En el repo puedes mantener zips (recomendación):
- `Q_Framework.zip` (framework listo)
- `Manual.zip` (documentación offline)

---

## Instalación local (XAMPP)

1) Clona o descomprime el proyecto dentro de `htdocs`:

```bash
git clone https://github.com/cesarrosasechevarria/Q_Framework.git
```

2) Asegura permisos de escritura:

- `write/` (y subcarpetas)
- en Windows/XAMPP: sólo verifica que no sea “solo lectura”

3) Configura `base_url`:

- **Opción A:** en `.env`
- **Opción B:** en `app/Config/App.php` (si no quieres `.env`)

4) Apunta el DocumentRoot a `public/` (recomendado)  
En XAMPP puedes usar VirtualHost, o entrar por `http://localhost/tu_proyecto/public/`.

---

## Estructura de carpetas

```txt
app/
  Config/
  Controllers/
  Modules/
  Views/
system/
  Core/
  Database/
  Helpers/
  Filters/
  Observability/
public/
  index.php
write/
  cache/
  logs/
  sessions/
  tenants/
```

---

## Configuración (.env y Config)

### 1) `.env` (opcional pero recomendado)

Crea `.env` en la raíz:

```ini
APP_ENV=development
APP_DEBUG=true
APP_BASE_URL=http://localhost/Q_Framework/public/

# DB
DB_GROUP=default
DB_HOST=127.0.0.1
DB_NAME=test
DB_USER=root
DB_PASS=
DB_PORT=3306

# Tenancy
TENANCY_ENABLED=true
TENANCY_STRATEGY=auto
TENANCY_DEFAULT=default
TENANCY_HEADER=X-Tenant-ID
TENANCY_QUERY=tenant
TENANCY_PATH_PREFIX=t
TENANCY_ONLY_KNOWN=false
TENANCY_STRICT_UNKNOWN=false
TENANCY_LOAD_TENANT_ENV=true

# Modules
MODULES_AUTO_DISCOVER=true
MODULES_RUNTIME_TOGGLE=true
MODULES_STATE_MODE=override

# Cache
CACHE_DRIVER=file
CACHE_PATH=write/cache
# CACHE_DRIVER=redis
# CACHE_REDIS_HOST=127.0.0.1
# CACHE_REDIS_PORT=6379
```

### 2) Config por `app/Config/*`

Todo puede configurarse por clases en `app/Config/` (ideal para entornos sin `.env`).

---

## Arranque y front controller

**Archivo recomendado:** `public/index.php`

- Define `ROOTPATH`
- carga bootstrap/autoload
- (opcional) boot de Tenant
- ejecuta `App->run()`

Ejemplo:

```php
define('ROOTPATH', dirname(__DIR__) . DIRECTORY_SEPARATOR);

// Boot tenant (si lo tienes habilitado)
\System\Core\Tenant::boot();

$app = new \System\Core\App();
$app->run();
```

---

## Rutas

Rutas base en: `app/Config/Routes.php`  
Además, cada módulo puede traer su `app/Modules/<Module>/Routes.php`.

Ejemplos:

```php
$routes->get('/', 'Modules\\Blog\\Controllers\\Blog@index', ['as' => 'blog.index']);
$routes->post('/auth/login', 'Modules\\Auth\\Controllers\\Auth@loginPost', ['as'=>'auth.login_post']);
```

### Parámetros tipados

Sintaxis:

- `{id:int}` → `\d+` y castea a `(int)`
- `{price:num}` → float/decimal y castea a `(float)`
- `{slug:slug}` → `[A-Za-z0-9_-]+`
- `{uuid:uuid}` → UUID v1–v5
- `{path:*}` o `{path:rest}` → captura con `/` (regex `.+`)
- `{id:re:\d{4}}` → **regex custom** (sin cast automático)

Ejemplos:

```php
$routes->get('/users/{id:int}', 'App\\Controllers\\Users@show');
$routes->get('/posts/{slug:slug}', 'App\\Controllers\\Blog@show');
$routes->get('/files/{path:*}', 'App\\Controllers\\Files@show');
$routes->get('/year/{y:re:\d{4}}', 'App\\Controllers\\Reports@year');
```

> Nota: Si necesitas “opcionalidad” tipo `/posts/{id?}`, define 2 rutas: una para “sin param” y otra con param.

### Auto-routes por prefijo

Útil si tienes un folder/controller con muchas acciones:

```php
$routes->auto('/funciones', 'App\\Controllers\\Funciones', [
  'defaultController' => 'Home',
  'defaultMethod'     => 'index',
]);
```

### Rutas con nombre

Genera URLs por nombre:

```php
$url1 = route_url('blog.index');
$url2 = route_url('blog.show', ['id' => 10]);
```

---

## Controladores

Puedes extender `System\Core\Controller` o tu `App\Controllers\BaseController`.

Ejemplo mínimo JSON:

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

Ejemplo completo (listado + create con `TableCrud` + JSON/redirect) está en la sección de DB.

---

## Vistas y layouts

Render sin layout:

```php
return view('blog/index', ['title' => 'Blog', 'items' => $items]);
```

Render con layout (3er parámetro):

```php
return $this->view('platform/dashboard', [
  'title' => 'Dashboard',
], 'platform/_layout');
```

Escapado:

```php
<h1><?= esc($title) ?></h1>
<p><?= e($subtitle) ?></p>
```

Assets y URLs:

```php
<link rel="stylesheet" href="<?= asset_url('assets/app.css') ?>">
<a href="<?= site_url('/users') ?>">Usuarios</a>
```

---

## Servicios (`service()`)

Uso:

```php
$db   = service('db');         // Connection
$pdo  = service('pdo');        // PDO
$req  = service('request');    // Request
$res  = service('response');   // Response
$sess = service('session');    // Session (lazy)
$enc  = service('encrypter');  // Encrypter
$val  = service('validator');  // Validator
$lang = service('lang');       // I18n
$cache= service('cache');      // CacheInterface (file/redis)
$rate = service('ratelimiter');// RateLimiter
$schema = service('schema');   // Schema (DDL/ensure)
$crud   = service('crud');     // Crud (dinámico por tabla)
$migrator = service('migrator'); // Migrator (migrations)
$seeder   = service('seeder');   // SeederRunner
$auth   = service('auth');     // AuthGuardInterface (sesión por defecto)
$log    = service('logger');   // LoggerInterface
$mailer = service('mailer');   // MailerInterface
```

Multi-DB / multi-grupo (si lo configuraste):

```php
$db2  = \Config\Services::db('analytics');
$pdo2 = \Config\Services::pdo('analytics');
```

---

## Helpers

### Helpers del sistema

Cargar:

```php
helper(['url', 'security', 'text', 'array', 'cookie']);
```

Listado (helpers y funciones):

- `app_helper` → `app_timezone()`, `app_locale()`
- `array_helper` → `array_get()`, `array_has()`, `array_set()`, `array_forget()`
- `cookie_helper` → `cookie_get()`, `cookie_set()`, `cookie_delete()`
- `pagination_helper` → `paginate()`
- `security_helper` →
  - CSRF: `csrf_token()`, `csrf_field()`, `csrf_key()`, `csrf_header()`, `csrf_meta()`
  - Password: `hash_password()`, `verify_password()`
  - Auth: `auth_user()`
  - CSP: `csp_nonce()`, `csp_attr()`
  - Otros: `random_str()`
- `text_helper` → `str_limit()`, `word_limiter()`, `slugify()`
- `url_helper` → `site_url()`, `asset_url()`, `current_url()`, `previous_url()`, `uri_string()`, `url_is()`, `prep_url()`, `anchor()`, `add_query()`, `remove_query()`, `build_url()`, `redirect_now()`, `redirect()`
- `view_helper` → `esc()`, `e()`

### Funciones globales core

Estas vienen del core (sin cargar helpers):

- Paths/Env: `base_path()`, `env()`, `env_bool()`, `env_int()`, `env_json()`
- Config: `config()`, `config_clear_cache()`
- Core: `service()`, `response()`, `view()`, `cache()`, `session()`, `session_optional()`
- URL: `base_url()`, `url()`, `route()`, `route_url()`
- Validación: `validate()`, `validate_or_fail()`
- Eventos: `on_event()`, `trigger_event()`
- Debug: `dd()`
- Salidas rápidas: `q_out()`, `q_json_out()`
- Abort: `abort()`

### Autoload de helpers (evita “undefined function”)

Si en tus vistas/controladores usas `asset_url()`, `csrf_meta()`, `auth_user()`, etc., debes **cargar el helper** correspondiente.

Opciones recomendadas:

**A) Autoload global (simple, ideal para starters):**  
En `app/Config/App.php`:

```php
public array $globalHelpers = ['url', 'security', 'text', 'array', 'cookie'];
public bool  $autoloadGlobalHelpers = true;
```

**B) Cargar por controller (más controlado):**

```php
final class Dashboard extends \System\Core\Controller
{
  public function __construct()
  {
    helper(['url', 'security']);
  }
}
```

---

## Filters (middleware)

Aliases en `app/Config/Filters.php` (ejemplos incluidos):
- `auth`, `guard`, `csrf`, `sec`, `obs`, `api`, `rate`, `feature`

Aplicar en una ruta:

```php
$routes->get('/demo/secure', 'Modules\\Demo\\Controllers\\Demo@secure', [
  'as' => 'demo.secure',
  'guard' => [
    'require'  => 'user',
    'redirect' => '@auth.login', // o '/auth/login'
    'remember' => true,
  ],
]);
```

Global before/after (para toda la app) se define en `globals` dentro del config.

---

## Auth (guard de sesión)

Por defecto el starter trae un guard basado en sesión (contrato `AuthGuardInterface`).

- En controllers: `service('auth')`
- En vistas (helper): `auth_user()`

Ejemplo típico:

```php
$user = service('auth')->user();
if (!$user) return redirect('/auth/login');
```

En una vista/layout:

```php
<?php helper('security'); ?>
<?php $u = auth_user(); ?>
<?= esc($u['name'] ?? 'Invitado') ?>
```

---

## Base de datos

### QueryBuilder + paginate

```php
$qb = service('db')->table('usuarios')->orderBy('id','DESC');
$rows  = $qb->paginate(10); // lee automáticamente ?page=
$pager = $qb->pager();

return view('usuarios/index', compact('rows','pager'));
```

En la vista:

```php
<?= $pager?->links() ?>
```

### `DbResult` y métodos `*Result()`

Muchos métodos tienen versión `*Result()` que retorna un objeto resultado estándar (`ok`, `action`, `data`, `rowCount`, `insertId`, `error`, `meta`).  
Útil para APIs y `fetch()`.

### CRUD por tabla (`TableCrud`)

Sin crear Model por entidad:

```php
use System\Database\Crud\TableCrud;

$crud = new TableCrud(service('db'), 'usuarios', [
  'primaryKey'     => 'id',
  'protectFields'  => true,
  'allowedFields'  => ['nombres','email','activo'],
  'useTimestamps'  => true,
  'useSoftDeletes' => false,
]);

$r = $crud->createResult([
  'nombres' => 'Ana',
  'email'   => 'ana@demo.com',
  'activo'  => 1,
]);

if (!$r->ok) {
  return response()->json(['ok'=>false,'error'=>$r->error], 400);
}
return response()->json(['ok'=>true,'insertId'=>$r->insertId], 201);
```

### Schema “ensure” (DDL)

Para crear/alterar tablas de forma declarativa:

```php
$schema = service('schema');

$r = $schema->ensure('usuarios', [
  ['nombre'=>'nombre', 'type'=>'VARCHAR', 'max_length'=>120, 'nullable'=>false],
  ['nombre'=>'email',  'type'=>'VARCHAR', 'max_length'=>120, 'nullable'=>false],
  ['nombre'=>'meta',   'type'=>'JSON',    'nullable'=>true],
], [
  'alter'        => true,
  'base_columns' => true,
]);

if (!$r->ok) return response()->json($r, 500);
```

---

## Migrations y Seeds

### Migrations

Carpeta típica:
- `app/Database/Migrations/` (y módulos si `includeModules=true`)

Ejecutar:

```php
$st = service('migrator')->status();   // ['applied'=>[], 'pending'=>[]]
$res = service('migrator')->migrate(); // ['applied'=>X, 'skipped'=>Y]
$rb  = service('migrator')->rollback();// int (rollback último batch)
```

### Seeds

```php
service('seeder')->run('UsersSeeder'); // App\Database\Seeds\UsersSeeder
```

---

## Módulos

Estructura recomendada:

```txt
app/Modules/Blog/
  Controllers/
  Models/
  Views/
  Routes.php
  Database/Migrations/
```

### Auto-discover

Si `Config\Modules::$autoDiscover=true`, el core detecta módulos automáticamente y registra:
- rutas (`Routes.php`)
- vistas
- migrations

### Activar/desactivar por tenant (runtime)

Si `runtimeToggle=true`, el estado puede guardarse por tenant en:

```txt
write/tenants/<tenant>/modules/modules.json
```

Ejemplo `modules.json`:

```json
{
  "enabled": ["Auth", "Admin"],
  "disabled": ["Platform"]
}
```

> Tip: Útil para “feature rollout” por cliente/empresa sin tocar el deploy.

---

## Multi-tenancy

`System\Core\Tenant::boot()` resuelve el tenant por estrategia:

- `auto`: header → query → subdomain → path
- `header`: por `X-Tenant-ID`
- `query`: `?tenant=...`
- `subdomain`: `<tenant>.tudominio.com`
- `path`: `/t/<tenant>/...`

Carpetas de write por tenant:

```txt
write/tenants/<tenant>/
  logs/
  cache/
  sessions/
  metrics/
  modules/
```

Opcional: `loadTenantEnv=true` permite overrides desde:

- `app/Tenants/<tenant>/.env`
- `write/tenants/<tenant>/.env`

---

## Cache

Por defecto usa `file` y es tenant-aware.

```php
cache()->set('k', ['a'=>1], 60);
$data = cache()->get('k');
cache()->delete('k');
```

Con redis, configura `app/Config/Cache.php` o `.env` (host/port/db/prefix).

---

## Seguridad

### CSRF

En formularios:

```php
<?php helper('security'); ?>
<form method="post">
  <?= csrf_field() ?>
  ...
</form>
```

Para fetch/AJAX, usa el header del helper:

```js
// Ejemplo conceptual: Header name viene de csrf_header()
headers[ "X-CSRF-TOKEN" ] = token;
```

En `<head>` (layout):

```php
<?= csrf_meta() ?>
```

### Rate limiting

Usa el filter `rate` en rutas sensibles:

```php
$routes->post('/auth/login', 'Modules\\Auth\\Controllers\\Auth@loginPost', [
  'rate' => ['key'=>'login', 'max'=>10, 'seconds'=>60],
]);
```

### Security headers

El filter `sec` agrega headers de seguridad globales (recomendado mantenerlo).

---

## Observabilidad

- Filtro `obs` (before/after) recolecta métricas HTTP.
- Endpoint de métricas (por defecto) `/_metrics` (configurable).

Ejemplos de lo que expone:
- `http_requests_total{method=...,path=...,status=...,tenant=...}`
- `http_request_duration_seconds_bucket{...}` / `_sum` / `_count`

---

## Errores comunes / troubleshooting

### 1) `Call to undefined function asset_url()` / `csrf_meta()` / `auth_user()`

Causa: **no se cargó el helper**.

Soluciones:
- Autoload global: `App::$globalHelpers = ['url','security'];`
- o en el controller: `helper(['url','security']);`

Funciones:
- `asset_url()` → `helper('url')`
- `csrf_meta()` / `csrf_field()` / `csrf_token()` / `auth_user()` → `helper('security')`

### 2) `Declaration must be compatible ... Controller::view(...)`

Causa: un método en tu controller usa el nombre `view()` con firma distinta y pisa el método del padre.

Solución:
- Renombra tu método (ej: `viewFile()`), o ajusta firma a:
  `view(string $name, array $data = [], ?string $layout = null): string`

### 3) Problemas con rutas y “/public”

Recomendación:
- En producción, apunta el DocumentRoot a `public/`.
- En local, si accedes por subcarpeta, ajusta `APP_BASE_URL`.


## Licencia

MIT. Ver `LICENSE`.
