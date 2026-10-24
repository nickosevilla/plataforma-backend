# Guía: conectarse a PostgreSQL y consultar las tablas

## 1. Datos de conexión

Según el `docker-compose.yml`:

| Parámetro | Valor |
|---|---|
| Contenedor Docker | `plataforma_db` |
| Motor | PostgreSQL 15 |
| Usuario | `admin` |
| Contraseña | `secretpassword` |
| Puerto desde Windows | `5433` |
| Puerto dentro del contenedor | `5432` |
| Volumen de datos | `pg_data` |
| Script de inicialización | `./init.sql` |

> **Importante:** la contraseña está definida en el archivo de configuración compartido. En un entorno real, evita subir credenciales a Git y utiliza variables de entorno o secretos.

## 2. Comprobar que PostgreSQL está funcionando

Abre PowerShell en la carpeta donde está tu `docker-compose.yml` y ejecuta:

```powershell
docker ps
```

Busca el contenedor `plataforma_db`.

Si no está iniciado, levanta el servicio desde esa carpeta:

```powershell
docker compose up -d postgres
```

Comprueba el estado:

```powershell
docker ps --filter "name=plataforma_db"
docker logs plataforma_db --tail 50
```

## 3. Entrar a PostgreSQL desde el contenedor

Esta opción no requiere instalar `psql` en Windows:

```powershell
docker exec -it plataforma_db psql -U admin -d postgres
```

Si aparece un prompt parecido a este, ya estás dentro de PostgreSQL:

```text
postgres=#
```

Los comandos siguientes se escriben **dentro de `psql`**, no en PowerShell.

### Comandos básicos de `psql`

| Comando | Qué hace |
|---|---|
| `\l` | Lista las bases de datos |
| `\c nombre_bd` | Se conecta a una base de datos |
| `\dt` | Lista las tablas del esquema actual |
| `\dt *.*` | Lista tablas de todos los esquemas visibles |
| `\dn` | Lista los esquemas |
| `\d nombre_tabla` | Muestra columnas, tipos e índices de una tabla |
| `\d+ nombre_tabla` | Muestra más detalles de la tabla |
| `\conninfo` | Muestra los datos de la conexión actual |
| `\q` | Sale de `psql` |

## 4. Elegir la base de datos correcta

Primero lista las bases:

```sql
\l
```

En este proyecto podrían existir bases como `iam_db`, `tenant_db`, `billing_db` y `transactions_db`, dependiendo de cómo se haya inicializado PostgreSQL.

Conéctate a la que necesites. Por ejemplo:

```sql
\c transactions_db
```

Después lista las tablas:

```sql
\dt
```

Si no aparece ninguna tabla, revisa todos los esquemas visibles:

```sql
\dt *.*
```

También puedes listar los esquemas:

```sql
\dn
```

## 5. Consultar los registros de una tabla

Supongamos que `\dt` muestra una tabla llamada `transactions`.

Ver su estructura:

```sql
\d transactions
```

Consultar algunos registros:

```sql
SELECT *
FROM transactions
LIMIT 10;
```

Contar registros:

```sql
SELECT COUNT(*)
FROM transactions;
```

Consultar columnas específicas:

```sql
SELECT id, created_at
FROM transactions
LIMIT 10;
```

> Sustituye `transactions`, `id` y `created_at` por nombres que existan realmente en tu base de datos. Puedes obtener los nombres de las columnas con `\d nombre_tabla`.

### Si la tabla está en otro esquema

Por ejemplo, si pertenece al esquema `public`:

```sql
SELECT *
FROM public.transactions
LIMIT 10;
```

## 6. Consultar todas las tablas con SQL

Esta consulta lista las tablas de usuario en los distintos esquemas, excluyendo los esquemas internos de PostgreSQL:

```sql
SELECT
    table_schema,
    table_name
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;
```

Para ver las columnas de las tablas:

```sql
SELECT
    table_schema,
    table_name,
    ordinal_position,
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name, ordinal_position;
```

## 7. Conectarse desde Windows con `psql`

Si tienes instalado el cliente PostgreSQL en Windows, puedes ejecutar desde PowerShell:

```powershell
psql -h localhost -p 5433 -U admin -d postgres
```

Te pedirá la contraseña. Introduce la definida en tu configuración.

Para conectarte directamente a una base específica, cambia `postgres` por su nombre:

```powershell
psql -h localhost -p 5433 -U admin -d transactions_db
```

Si PowerShell dice que `psql` no se reconoce, utiliza la opción del apartado 3 con `docker exec`.

## 8. Entrar con una interfaz gráfica

Puedes utilizar un cliente como **pgAdmin** o **DBeaver**. Configura la conexión así:

- Host: `localhost`
- Puerto: `5433`
- Base de datos inicial: `postgres` o una base existente
- Usuario: `admin`
- Contraseña: la configurada en Docker

Una vez conectado, abre la base de datos y navega a **Schemas → public → Tables** (la ubicación puede variar según el cliente).

## 9. Problemas frecuentes

### No puedo conectarme

Comprueba que el contenedor está activo:

```powershell
docker ps --filter "name=plataforma_db"
```

Comprueba los registros:

```powershell
docker logs plataforma_db --tail 100
```

### No aparece una base de datos esperada

Lista las bases con `\l`. Ten presente que `POSTGRES_USER` y `POSTGRES_PASSWORD` configuran el usuario inicial; **no crean automáticamente** bases como `iam_db`, `tenant_db`, `billing_db` o `transactions_db`.

### `init.sql` no parece ejecutarse

Los scripts de `/docker-entrypoint-initdb.d/` se ejecutan al inicializar un directorio de datos vacío. Si el volumen `pg_data` ya estaba inicializado, cambiar `init.sql` y reiniciar el contenedor normalmente no vuelve a ejecutar el script.

No elimines el volumen para forzar la inicialización sin respaldar los datos: borrarlo puede eliminar todas las bases almacenadas en él.

### No hay tablas

Verifica que estás en la base correcta:

```sql
SELECT current_database();
\dt *.*
```

Si sigue sin haber tablas, revisa si las migraciones de Drizzle u otro mecanismo de inicialización se han ejecutado.

## 10. Flujo rápido para recordar

Desde PowerShell:

```powershell
docker exec -it plataforma_db psql -U admin -d postgres
```

Dentro de `psql`:

```text
\l
\c transactions_db
\dt
\d nombre_tabla
```

Consulta datos:

```sql
SELECT *
FROM nombre_tabla
LIMIT 10;
```

Sal de `psql`:

```text
\q
```
