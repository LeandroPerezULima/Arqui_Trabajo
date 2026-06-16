# Patrón Cloud: Sharding aplicado a la Caché de RutaSegura

**Integrante:** Leandro Nahuel Perez Arteaga
**Módulo individual:** Caché inteligente de reportes viales con Redis (NoSQL clave/valor)
**Tema de refuerzo:** Patrón Cloud *Sharding* — fragmentación de datos para escalado horizontal

> Catálogo de referencia: [Azure Architecture Center — Sharding Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding)
> Categoría: *Data Management* · Atributos de calidad: **Escalabilidad, Disponibilidad**


## 1. Contexto: por qué el módulo necesita sharding

El módulo base de caché usa **un solo nodo Redis** con el patrón Cache-Aside para servir reportes de incidentes viales de Lima en menos de 200 ms.

Un único nodo tiene tres límites:

| Límite | Problema |
|---|---|
| Memoria RAM | Cuántas claves caben en el servidor |
| CPU / red | Cuántas peticiones por segundo aguanta |
| Punto único de fallo | Si cae, cae toda la caché |

Ante un evento masivo en Lima —miles de usuarios consultando incidentes al mismo tiempo— no es viable hacer el servidor infinitamente más grande (**escalado vertical**). El sharding propone otra salida: **escalado horizontal**, agregando más nodos.

---

## 2. ¿Qué es sharding?

**Sharding** (fragmentación) consiste en partir un conjunto de datos grande en fragmentos más pequeños llamados *shards* y poner cada fragmento en un servidor distinto. Cada nodo guarda solo su parte; juntos forman el total.

```
Sin sharding:         Con sharding (N = 3):
                      
┌──────────────┐      ┌─────────┐  ┌─────────┐  ┌─────────┐
│   1 Redis    │      │ Redis 0 │  │ Redis 1 │  │ Redis 2 │
│ todas las    │  →   │ ~1/3 de │  │ ~1/3 de │  │ ~1/3 de │
│ zonas de Lima│      │ las zona│  │ las zona│  │ las zona│
└──────────────┘      └─────────┘  └─────────┘  └─────────┘
   un cuello          la carga se reparte entre los tres
```

---

## 3. Sharding vs Replicación

Es importante no confundir los dos patrones:

| | Replicación | Sharding |
|---|---|---|
| **¿Qué copia?** | Los **mismos** datos en varios servidores | Datos **distintos** en cada servidor |
| **Cada nodo tiene…** | Todo | Una parte |
| **Sirve para…** | Redundancia / disponibilidad | Escalar la capacidad |
| **¿Se combinan?** | Sí — cada shard puede tener réplicas propias | |

---

## 4. La función de enrutamiento

La pieza central del patrón es una regla que responde siempre lo mismo a dos preguntas:

- Cuando **guardo** una clave, ¿en qué nodo la pongo?
- Cuando **leo** esa clave, ¿en qué nodo la busco?

Ambas respuestas deben coincidir siempre, o los datos se "perderían". Esa regla es la **función de enrutamiento**:

```
nodo = hash("reportes:{zona}") % N
```

Tres pasos:

```
"reportes:miraflores"
        │
        ▼
   hashlib.sha1(...)
        │
        ▼
   847...291  (número gigante, único y determinista)
        │
        ▼
   847...291 % 3  =  2   →   nodo-2
```

---

## 5. Qué es N y cómo actúa

`N` es el **número de nodos Redis** que forman el shard. En el código, N no es una variable con ese nombre; vive implícita en la lista de nodos:

```python
NODOS = [
    {"nombre": "nodo-0", "host": "localhost", "port": 6390},
    {"nombre": "nodo-1", "host": "localhost", "port": 6391},
    {"nombre": "nodo-2", "host": "localhost", "port": 6392},
]
# N = len(NODOS) = 3
```

N actúa como el **divisor del módulo**. El módulo (`%`) siempre da un resultado entre `0` y `N-1`, que son exactamente los índices de los nodos disponibles. Con N = 3:

```
hash(clave) % 3  →  puede dar 0, 1 o 2   (índices de nodo-0, nodo-1, nodo-2)
```

Esto garantiza que cada clave siempre caiga dentro del rango de nodos existentes.

---

## 6. El hash: huella determinista, no azar

El número gigante que produce el hash **no es aleatorio**. Es **determinista**: la misma clave produce siempre el mismo número, en cualquier ejecución y en cualquier computadora.

```python
hashlib.sha1("reportes:miraflores".encode()).hexdigest()
# Siempre produce: el mismo valor hexadecimal, sin excepción
```

Parecé azar porque tiene dos propiedades que lo distribuyen uniformemente:

**Efecto avalancha:** un cambio mínimo en la entrada produce un número totalmente distinto.

```
sha1("reportes:miraflores")  →  7a3f...
sha1("reportes:mirafloras")  →  c01e...  ← una letra cambió, número completamente distinto
```

**Distribución uniforme:** los números se esparcen parejo por todo el rango. Aunque todas las zonas empiecen con "San" (San Isidro, San Borja, San Miguel...), sus hashes quedan dispersos y el `% N` las reparte de forma equilibrada entre nodos.

### ¿Por qué SHA-1 y no el `hash()` de Python?

```python
# hash() de Python: CAMBIA entre ejecuciones (PYTHONHASHSEED aleatorio)
hash("reportes:miraflores")  # hoy: 1234..., mañana: 9876...  ✗

# hashlib.sha1: SIEMPRE el mismo resultado
hashlib.sha1("reportes:miraflores".encode()).hexdigest()  # siempre igual  ✓
```

El `hash()` nativo de Python cambia entre ejecuciones por seguridad. Si lo usáramos, guardaríamos una clave en el nodo 2, reiniciaríamos el sistema, y al buscarla el hash diferente nos mandaría a otro nodo: el dato se "pierde". SHA-1 elimina ese riesgo.

---

## 7. Tres estrategias de reparto

| Estrategia | Fórmula | Ventaja | Desventaja |
|---|---|---|---|
| **Por rango** | A–H → nodo 0, I–P → nodo 1… | Consultas por rango eficientes | Se desbalancea fácil (*hotspot*) |
| **Por hash** | `hash(clave) % N` | Balance casi perfecto | Cambiar N reubica casi todo |
| **Por directorio** | Tabla explícita clave → nodo | Máxima flexibilidad | La tabla es un punto central a mantener |

La demo usa **sharding por hash**, que es la estrategia estándar y la más didáctica para entender el patrón.

---

## 8. El punto débil: re-balanceo

Como N es el divisor del módulo, **cambiar N cambia el nodo de casi todas las claves**. El número de cada clave (su hash) no cambia; lo que cambia es el resto al dividir por un N distinto:

```
847...291 % 3 = 2   →  nodo-2   (con 3 nodos)
847...291 % 4 = 3   →  nodo-3   (con 4 nodos)  ← cambió de nodo
847...291 % 5 = 1   →  nodo-1   (con 5 nodos)  ← cambió otra vez
```

Resultados reales medidos por `rebalanceo.py` con 1000 claves:

| Cambio de nodos | Claves reubicadas | % |
|---|---|---|
| 2 → 3 | 668 / 1000 | **66.8%** |
| 3 → 4 | 726 / 1000 | **72.6%** |
| 4 → 5 | 796 / 1000 | **79.6%** |
| 5 → 6 | 830 / 1000 | **83.0%** |

Pasar de 3 a 4 nodos reubica el **~73% de las claves**. En una caché, eso provoca una tormenta temporal de *cache miss*: casi todo el mundo tiene que ir a PostgreSQL a recargar datos, justo cuando se intenta escalar para aguantar más carga.

---

## 9. La solución: hashing consistente

El **hashing consistente** (*consistent hashing*) resuelve el costo del re-balanceo. En vez de `hash % N`, imagina un **anillo** (un círculo):

```
              nodo-0
               (0°)
         k3  ·  ·  · k1
       ·                 ·
     ·                     ·
  nodo-2                 nodo-1
  (240°)     k2          (120°)
     ·                     ·
       ·                 ·
         ·  ·  ·  ·  ·
```

- Cada nodo y cada clave se colocan en el anillo según su hash.
- Cada clave pertenece al **primer nodo que encuentra girando en sentido horario**.
- Al **agregar un nodo**, este se inserta en un punto y solo "roba" las claves del tramo entre él y el nodo anterior. Las demás se quedan donde estaban.

Resultado: al escalar, solo se reubica **≈ 1/N** de las claves, en lugar de casi todas.

Esto es lo que usan internamente **Redis Cluster** (con 16 384 hash slots), Cassandra, DynamoDB y Memcached.

---

## 10. Demo en terminal

La demo tiene dos scripts Python que ilustran el patrón y su limitación.

### `sharding_demo.py` 

```python


import argparse
import hashlib
import json
import sys
from datetime import datetime

NODOS = [
    {"nombre": "nodo-0", "host": "localhost", "port": 6390},
    {"nombre": "nodo-1", "host": "localhost", "port": 6391},
    {"nombre": "nodo-2", "host": "localhost", "port": 6392},
]

DATOS_ZONAS = {
    "miraflores": [
        {"tipo": "Accidente", "ubicacion": "Av. Larco", "estado": "Activo"},
        {"tipo": "Vehículo mal estacionado", "ubicacion": "Calle Schell", "estado": "Activo"},
    ],
    "san_isidro": [
        {"tipo": "Congestión Vehicular", "ubicacion": "Av. Javier Prado", "estado": "Activo"},
    ],
    "surco": [
        {"tipo": "Bloqueo vial", "ubicacion": "Av. Primavera", "estado": "Activo"},
    ],
    "barranco": [
        {"tipo": "Accidente", "ubicacion": "Av. Grau", "estado": "Activo"},
    ],
    "san_borja": [
        {"tipo": "Congestión Vehicular", "ubicacion": "Av. Aviación", "estado": "Activo"},
    ],
    "la_molina": [
        {"tipo": "Vehículo mal estacionado", "ubicacion": "Av. La Molina", "estado": "Activo"},
    ],
    "lince": [
        {"tipo": "Bloqueo vial", "ubicacion": "Av. Arequipa", "estado": "Activo"},
    ],
    "chorrillos": [
        {"tipo": "Accidente", "ubicacion": "Av. Defensores", "estado": "Activo"},
    ],
}

TTL_SEGUNDOS = 60


def indice_nodo(clave: str, total_nodos: int) -> int:
    """Devuelve el índice del nodo donde debe vivir la clave.
    Usa SHA-1 (no hash() de Python) porque SHA-1 es estable entre ejecuciones.
    """
    digest = hashlib.sha1(clave.encode("utf-8")).hexdigest()
    return int(digest, 16) % total_nodos


def clave_de(zona: str) -> str:
    return f"reportes:{zona}"


class ShardRouter:
    """Router de shards: abstrae en qué nodo va cada clave."""

    def __init__(self, conexiones):
        self.conexiones = conexiones
        self.total = len(conexiones)  # esto es N

    def _nodo(self, clave: str):
        i = indice_nodo(clave, self.total)
        return i, self.conexiones[i]

    def set(self, zona: str, incidentes: list):
        clave = clave_de(zona)
        i, conn = self._nodo(clave)
        valor = json.dumps(
            {
                "ultima_actualizacion": datetime.now().strftime("%Y-%m-%d %H:%M"),
                "incidentes": incidentes,
            },
            ensure_ascii=False,
        )
        conn.setex(clave, TTL_SEGUNDOS, valor)
        return i

    def get(self, zona: str):
        clave = clave_de(zona)
        i, conn = self._nodo(clave)
        raw = conn.get(clave)
        return i, (json.loads(raw) if raw else None)

    def mapa_distribucion(self, zonas):
        dist = {i: [] for i in range(self.total)}
        for z in zonas:
            i = indice_nodo(clave_de(z), self.total)
            dist[i].append(z)
        return dist


class RedisSimulado:
    """Cliente Redis en memoria para correr la demo sin instalar Redis."""

    def __init__(self, nombre):
        self.nombre = nombre
        self._store = {}

    def setex(self, clave, ttl, valor):
        self._store[clave] = valor

    def get(self, clave):
        return self._store.get(clave)

    def ping(self):
        return True


def conectar(simulado: bool):
    if simulado:
        print(">> Modo SIMULADO: no se requiere Redis instalado.\n")
        return [RedisSimulado(n["nombre"]) for n in NODOS]
    try:
        import redis
    except ImportError:
        print("ERROR: falta el paquete 'redis'. Instala con:  pip install redis")
        sys.exit(1)
    conexiones = []
    for n in NODOS:
        c = redis.Redis(host=n["host"], port=n["port"], decode_responses=True)
        try:
            c.ping()
        except Exception as e:
            print(f"ERROR: no se pudo conectar a {n['nombre']}. ¿Levantaste los Redis? Ver sección 11.")
            print(f"       Alternativa:  python sharding_demo.py --simulado")
            sys.exit(1)
        conexiones.append(c)
    print(">> Modo REAL: conectado a 3 nodos Redis.\n")
    return conexiones


def barra(n, total, ancho=20):
    llenos = int(round(ancho * n / total)) if total else 0
    return "█" * llenos + "·" * (ancho - llenos)


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--simulado", action="store_true")
    args = parser.parse_args()

    conexiones = conectar(args.simulado)
    router = ShardRouter(conexiones)

    print("=" * 60)
    print("  DEMO SHARDING - Caché de reportes viales (RutaSegura)")
    print(f"  Nodos en el shard: {router.total}  (N = {router.total})")
    print("=" * 60)

    print("\n[1] Cacheando zonas — cada zona va a su nodo según el hash:\n")
    for zona, incidentes in DATOS_ZONAS.items():
        i = router.set(zona, incidentes)
        print(f"    reportes:{zona:<14} ->  {NODOS[i]['nombre']}")

    print("\n[2] Distribución de claves entre nodos:\n")
    dist = router.mapa_distribucion(DATOS_ZONAS.keys())
    total = len(DATOS_ZONAS)
    for i in range(router.total):
        zs = dist[i]
        print(f"    {NODOS[i]['nombre']}  [{barra(len(zs), total)}] {len(zs)}")
        for z in zs:
            print(f"        - {z}")

    print("\n[3] Lectura dirigida — la app va directo al nodo correcto:\n")
    for zona in ["miraflores", "san_isidro", "surco"]:
        i, valor = router.get(zona)
        if valor:
            n_inc = len(valor["incidentes"])
            print(f"    GET reportes:{zona:<12} ->  {NODOS[i]['nombre']}  "
                  f"({n_inc} incidente(s), act. {valor['ultima_actualizacion']})")

    print("\n" + "=" * 60)
    print("  Cada nodo guarda solo SU parte de las zonas.")
    print("  La app nunca recorre todos los nodos: va directo al correcto.")
    print("=" * 60 + "\n")


if __name__ == "__main__":
    main()
```

---

### `rebalanceo.py` — Muestra lo que pasa al cambiar N

```python
"""

Cuando cambia N, la fórmula hash % N cambia para casi todas las claves:
la caché se "desordena" y provoca una tormenta de cache miss.

Este script lo cuantifica con 1000 claves de muestra.

Ejecución:  python rebalanceo.py
No requiere Redis.
"""

import hashlib


def indice_nodo(clave: str, total_nodos: int) -> int:
    digest = hashlib.sha1(clave.encode("utf-8")).hexdigest()
    return int(digest, 16) % total_nodos


def generar_zonas(cantidad: int):
    # Claves sintéticas para tener una muestra estadísticamente representativa.
    # Con solo 8 zonas reales el porcentaje variaría demasiado entre casos.
    return [f"reportes:zona_{i:04d}" for i in range(cantidad)]


def porcentaje_reubicado(claves, n_antes, n_despues):
    """
    Para cada clave, calcula su nodo con n_antes y con n_despues.
    Si son distintos, la clave se "mueve" de nodo al escalar.
    """
    movidas = 0
    for c in claves:
        if indice_nodo(c, n_antes) != indice_nodo(c, n_despues):
            movidas += 1
    return movidas, len(claves), 100.0 * movidas / len(claves)


def main():
    claves = generar_zonas(1000)

    print("=" * 64)
    print("  Sharding por hash módulo N: costo de cambiar el número de nodos")
    print(f"  Muestra: {len(claves)} claves")
    print("=" * 64)
    print()
    print(f"  {'Cambio de nodos':<22}{'Claves reubicadas':<20}{'%':>8}")
    print("  " + "-" * 50)

    for n in range(2, 6):
        movidas, total, pct = porcentaje_reubicado(claves, n, n + 1)
        print(f"  {f'{n} -> {n+1}':<22}{f'{movidas} / {total}':<20}{pct:>7.1f}%")

    print()
    print("  Interpretación:")
    print("  - El número de cada clave (su hash) no cambia nunca.")
    print("  - Lo que cambia es el divisor N, y con él el resto del módulo.")
    print("  - En una caché real, cada clave reubicada produce un cache miss.")
    print()
    print("  Solución: HASHING CONSISTENTE (consistent hashing).")
    print("  Al añadir un nodo solo reubica ~1/N claves, no casi todas.")
    print("  Es lo que usan Redis Cluster, Cassandra, DynamoDB y Memcached.")
    print("=" * 64)


if __name__ == "__main__":
    main()
```

---

### `docker-compose.yml` — tres nodos Redis para el modo real

```yaml
services:
  redis-nodo-0:
    image: redis:7-alpine
    container_name: rutasegura-redis-0
    ports:
      - "6390:6379"

  redis-nodo-1:
    image: redis:7-alpine
    container_name: rutasegura-redis-1
    ports:
      - "6391:6379"

  redis-nodo-2:
    image: redis:7-alpine
    container_name: rutasegura-redis-2
    ports:
      - "6392:6379"
```

---

## 11. Requisitos y ejecución

### Requisitos

```
redis>=5.0.0
```

Solo necesario para el **modo real**. El modo simulado no requiere ningún paquete externo.

### Opción A — Sin instalar nada (modo simulado)

Ideal para revisar el código o exponer sin riesgo. Usa un cliente Redis en memoria.

```bash
python sharding_demo.py --simulado
python rebalanceo.py
```

### Opción B — Con Redis real

**1. Levantar los tres nodos Redis:**

```bash
docker compose up -d
```

**2. Instalar la dependencia de Python:**

```bash
pip install redis>=5.0.0
```

**3. Correr la demo principal:**

```bash
python sharding_demo.py
```

**4. Correr el script de re-balanceo:**

```bash
python rebalanceo.py
```

**5. Apagar los nodos al terminar:**

```bash
docker compose down
```

---

## Relación con el módulo individual

| Módulo base (Cache-Aside + Redis) | Este refuerzo (Sharding) |
|---|---|
| Un solo nodo Redis | Varios nodos Redis |
| Resuelve la **velocidad** de lectura | Resuelve el **escalado horizontal** |
| Patrón Cache-Aside | Patrón Sharding |
| `reportes:{zona}` → Redis único | `reportes:{zona}` → nodo según hash |

El módulo base resuelve la latencia; el sharding resuelve la capacidad. Cuando el tráfico supera lo que un nodo puede manejar —un evento masivo en Lima, por ejemplo—, el sharding permite distribuir la carga entre varios nodos sin cambiar la lógica de la aplicación: solo aumenta N.

---

[⬅️ Anterior](../4.4/4.4.md) | [🏠 Home](../../README.md) | [Siguiente ➡️](../4.6/4.6.md)
