---
id: ART-19
titulo: Validación de entradas
version: 1.0.0
estado: vigente
aplica_a: [backend, frontend-web, movil]
depende_de: [ART-02, ART-10, ART-17]
---

# ART-19 · Validación de entradas

> **Instrucción para agentes IA**
> - Usa los **tipos reutilizables** de §2 en vez de declarar reglas sueltas; si el proyecto necesita uno nuevo, agrégalo a `shared` con su equivalente en el front (ART-30).
> - Cada campo declarado en ART-46 §3 debe mapear a uno de estos tipos o a una regla explícita.
> - Principios P-VA-01..07.

---

## 1. Capas de validación

| Capa | Qué valida | Herramienta |
|---|---|---|
| Nginx | Tamaño de body, métodos, timeouts | ART-34 |
| Esquema de entrada (infraestructura) | Tipo, obligatoriedad, longitud, patrón, rango, campos extra | Pydantic v2 |
| Normalización | Unicode NFC, recorte de espacios, eliminación de caracteres de control | Validador `BeforeValidator` |
| Value objects (dominio) | Invariantes del concepto (ej. NIT con dígito de verificación) | Python puro |
| Caso de uso | Reglas que requieren datos (existencia, unicidad, alcance) | Consultas (ART-15) |
| Base de datos | `NOT NULL`, `CHECK`, `UNIQUE`, FK | Última barrera |

## 2. Tipos reutilizables (`shared/infrastructure/http/types.py`)

```python
import re
import unicodedata
from decimal import Decimal
from typing import Annotated

from pydantic import AfterValidator, BeforeValidator, EmailStr, Field, StringConstraints

_CONTROL = re.compile(r"[\x00-\x08\x0B\x0C\x0E-\x1F\x7F​-‏‪-‮⁦-⁩]")

def normalizar(v: object) -> object:
    if isinstance(v, str):
        v = unicodedata.normalize("NFC", v)
        v = _CONTROL.sub("", v)
        v = v.strip()
    return v

Texto = BeforeValidator(normalizar)
LETRAS = r"A-Za-zÁÉÍÓÚáéíóúÑñÜü"

NombrePersona = Annotated[str, Texto, StringConstraints(min_length=2, max_length=80,
                          pattern=rf"^[{LETRAS}]+(?:[ '\-][{LETRAS}]+)*$")]
RazonSocial = Annotated[str, Texto, StringConstraints(min_length=2, max_length=150,
                        pattern=rf"^[{LETRAS}0-9 .,&'\-()]+$")]
TextoCorto = Annotated[str, Texto, StringConstraints(min_length=1, max_length=200,
                       pattern=rf"^[{LETRAS}0-9 .,;:¿?¡!'\"()\-_/#%@+]+$")]
TextoLargo = Annotated[str, Texto, StringConstraints(min_length=1, max_length=2000)]   # Sin patrón: se escapa al mostrar; NO admite HTML
Codigo = Annotated[str, Texto, StringConstraints(min_length=1, max_length=30, pattern=r"^[A-Z0-9][A-Z0-9\-_]*$")]
Email = Annotated[EmailStr, Field(max_length=254)]
TelefonoCO = Annotated[str, Texto, StringConstraints(pattern=r"^(\+57)?3\d{9}$|^(\+57)?60\d{8}$")]
CedulaCO = Annotated[str, Texto, StringConstraints(pattern=r"^\d{5,10}$")]
NitCO = Annotated[str, Texto, StringConstraints(pattern=r"^\d{6,10}-?\d$")]     # DV se valida en el value object
Cantidad = Annotated[int, Field(ge=1, le=1_000_000)]
Dinero = Annotated[Decimal, Field(ge=0, le=Decimal("9999999999.99"), max_digits=12, decimal_places=2)]
Porcentaje = Annotated[Decimal, Field(ge=0, le=100, max_digits=5, decimal_places=2)]
Busqueda = Annotated[str, Texto, StringConstraints(min_length=2, max_length=100)]
Pagina = Annotated[int, Field(ge=1, le=10_000)]
TamanoPagina = Annotated[int, Field(ge=1, le=100)]
```

### Contraseñas
```python
Password = Annotated[str, StringConstraints(min_length=12, max_length=128)]
```
- Mínimo 12, máximo 128 (el máximo evita abuso de CPU en el hash).
- No se exige composición rara; se rechazan contraseñas comunes (lista local de las 100 mil más usadas) y las que contienen el correo o el nombre.
- Nunca se normalizan ni recortan: se usan tal cual.

### Texto libre y HTML
- Por defecto **no se acepta HTML**. Los campos de texto se guardan como texto y el front los muestra escapados (React lo hace por defecto).
- Si un campo debe aceptar formato enriquecido: Markdown con un subconjunto permitido, o HTML sanitizado en el servidor con `nh3` (lista blanca de etiquetas) y de nuevo con DOMPurify al mostrar. Requiere ADR.

## 3. Archivos (P-VA-04)

```python
ALLOWED = {"application/pdf": ".pdf", "image/png": ".png", "image/jpeg": ".jpg"}
MAX_BYTES = 5 * 1024 * 1024

async def validar_archivo(upload: UploadFile) -> ArchivoValidado:
    head = await upload.read(4096)
    mime = magic.from_buffer(head, mime=True)
    if mime not in ALLOWED:
        raise ArchivoTipoNoPermitido()
    size = len(head)
    chunks = [head]
    while chunk := await upload.read(64 * 1024):
        size += len(chunk)
        if size > MAX_BYTES:
            raise ArchivoDemasiadoGrande()
        chunks.append(chunk)
    contenido = b"".join(chunks)
    return ArchivoValidado(contenido=contenido, mime=mime,
                           nombre_almacenado=f"{uuid7()}{ALLOWED[mime]}",
                           nombre_original=sanitizar_nombre(upload.filename))
```
Reglas adicionales:
- Imágenes: re-codificar (Pillow) para eliminar metadatos EXIF y contenido incrustado; límite de dimensiones (anti "bombas" de descompresión).
- PDF: no se renderiza en el servidor sin aislamiento; se sirve con `Content-Disposition: attachment` y `X-Content-Type-Options: nosniff`.
- ZIP u Office: evitar; si se requieren, límite de tamaño descomprimido y antivirus (ClamAV) por ADR.
- Los archivos se sirven por un endpoint que verifica permiso y alcance, nunca por ruta pública.

## 4. Parámetros de query y path
- Path: tipos estrictos (`UUID`); un id mal formado → 422, no 500.
- Orden: `Literal[...]` con los campos permitidos (ART-14 §4.4).
- Filtros: modelo Pydantic dedicado (`FiltroFacturasIn`) con `extra="forbid"` usando `Annotated[FiltroFacturasIn, Query()]`.
- Listas en query (`?estado=A&estado=B`): máximo de elementos.

## 5. Headers y webhooks
- Headers usados por la app (`Idempotency-Key`, `If-Match`, `X-Request-ID`) se validan con formato y longitud.
- Webhooks entrantes: verificar firma HMAC con comparación de tiempo constante, timestamp con ventana ≤ 5 min (anti-replay), luego validar el body con esquema.

## 6. WebSocket
Cada mensaje entrante: tamaño máximo antes de parsear → `model_validate_json` con modelo discriminado por `type` → rate por conexión (ART-10 §6.9).

## 7. Expresiones regulares seguras
- Sin cuantificadores anidados ni alternancias ambiguas (riesgo ReDoS).
- Siempre ancladas `^...$`.
- La longitud máxima se valida **antes** del patrón (Pydantic aplica `max_length` antes de `pattern`).
- Probar cada regex nueva con entradas largas (10 KB) en las pruebas adversarias.

## 8. Mensajes de validación
Ver ART-17 §5: `field` + `rule`, sin el valor recibido. El front muestra mensajes propios por regla (ART-30).
