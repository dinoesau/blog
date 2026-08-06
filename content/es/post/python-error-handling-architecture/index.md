---
title: "Deja de Validar en Todas Partes: Una Guía Arquitectónica para el Manejo de Errores en Python"
description: "Aprende a construir aplicaciones Python resilientes distinguiendo entre validación y aserciones, implementando el patrón Result y usando NewTypes para hacer que los estados inválidos sean irrepresentables."
date: 2026-04-13
categories:
    - Software Architecture
    - Python
    - Development Patterns
tags:
    - Error Handling
    - Type Safety
    - Functional Programming
    - Pydantic
    - NewType
    - Clean Code
---


> *Un desarrollador junior escribe código que asume que todo saldrá bien (sin comprobaciones).*
> *Un desarrollador intermedio escribe código que asume que todo saldrá mal (comprobaciones defensivas y aserciones por todas partes).*
> *Un desarrollador senior usa el sistema de tipos para hacer imposible el error, eliminando el 80% de esas comprobaciones por completo.*<br>
> — <cite>Proverbio de Ingeniería de Software</cite>

<!--more-->

## TL;DR

* **Valida en los bordes (defensivo):** Espera datos malos del mundo exterior. Manejarlos con elegancia usando el patrón `Ok / Err` (el patrón `Result`) en lugar de lanzar errores impredecibles con `try/except`.
* **Afirma en el núcleo (ofensivo):** Espera datos perfectos internamente. Si tu estado interno es incorrecto, es un bug del sistema. Usa un `assert_ok` personalizado para fallar de inmediato (fail-fast).
* **Parsea, no valides:** No solo compruebes los datos; transfórmalos en el límite en primitivas de dominio tipadas usando `NewType` (p. ej., `ValidatedUserId`, `PositiveAmount`).
* **El Result:** Tus funciones de lógica de negocio central *solo* aceptan tipos de dominio marcados. Esto hace que los estados inválidos sean matemáticamente imposibles de representar, permitiéndote eliminar cientos de líneas de comprobaciones defensivas `if not data`.

---

Si has pasado suficiente tiempo escribiendo servicios web en Python, es probable que hayas escrito una función que se ve exactamente así:

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()


@app.post("/refund")
async def process_refund(req: Request):
    # ❌ El Controlador "Defensivo" de Nivel Intermedio
    try:
        body = await req.json()  # ⚠️ ¡Lanza si el JSON está mal formado!

        # Comprobaciones defensivas por todas partes...
        if not body or not isinstance(body, dict):
            raise ValueError("Invalid payload")
        if "userId" not in body or not isinstance(body["userId"], str):
            raise ValueError("Missing UserId")
        if "amount" not in body or not isinstance(body["amount"], (int, float)) or body["amount"] <= 0:
            raise ValueError("Invalid amount")

        user = await get_user(body["userId"])
        if not user:
            raise ValueError("User not found")  # ¿Error esperado? ¿O bug de la BD?

        # Finalmente... la lógica de negocio real
        await gateway.refund(user.stripe_id, body["amount"])

        return JSONResponse({"message": "Success"}, status_code=200)
    except Exception as e:
        # ⚠️ ¿Es un error 400 de petición incorrecta? ¿O una falla 500 de la BD? Ya no lo sabemos.
        return JSONResponse({"error": str(e)}, status_code=400)
```

Este código es agotador de leer. La lógica de negocio central está enterrada bajo una montaña de sentencias defensivas `if/raise`. Además, el bloque `except` no tiene idea de si el error fue un usuario escribiendo mal (400 Bad Request) o la base de datos incendiándose (500 Internal Error).

> **Una nota sobre FastAPI:** Sí, FastAPI puede manejar la validación de Pydantic automáticamente en la firma de la ruta (p. ej., `async def process_refund(payload: RefundInput):`). Lo estamos haciendo manualmente aquí para demostrar claramente el límite arquitectónico entre el framework web, el Parser y el Dominio Central. En una aplicación real probablemente usarías la integración de FastAPI, pero el patrón arquitectónico sigue siendo idéntico.

El manejo de errores no se trata solo de atrapar errores; **se trata de arquitectura de sistemas**.

Para arreglar esto, necesitamos entender la diferencia fundamental entre **Validación** y **Aserción**, adoptar el **patrón Result** y usar **Branded Types** (`NewType`) para empujar los errores a los bordes absolutos de nuestro sistema.

## Regla 1: Aprende la Diferencia Entre Validación y Aserción

La distinción arquitectónica más importante que puedes hacer es entender la diferencia entre validar datos y afirmar estado.

Piensa en tu aplicación como un Club Nocturno exclusivo.

| Característica | Validación | Aserción |
| :--- | :--- | :--- |
| **Ubicación** | La Puerta Principal (Límite de API/IO) | La Sala VIP (Lógica Central) |
| **Expectativa** | Los datos malos son *esperados* | Los datos son *confiables* |
| **Filosofía** | Defensiva (Seguridad de la puerta) | Ofensiva (Guardia de seguridad) |
| **Resultado** | Recuperación elegante (400 Bad Request) | Bloqueo inmediato (500 System Error) |

* **La validación es el guardia en la puerta principal.**
  El guardia *espera* que la gente le dé identificaciones falsas o que sean menores de edad.
  Los rechaza con calma.
  **La validación inspecciona la entrada externa no confiable y se recupera con elegancia.**

* **La aserción es el guardia de seguridad dentro de la Sala VIP.**
  El guardia *espera* que todos en la sala tengan una pulsera VIP.
  Si alguien está en la sala sin una, el sistema del guardia ha fallado fundamentalmente.
  El guardia detiene la música y cierra la fiesta.
  **Las aserciones inspeccionan la lógica interna y fallan rápido.**

Cuando mezclas estas cosas, los sistemas se vuelven frágiles. Si bloqueas la aplicación cuando un usuario escribe un correo incorrecto, tienes una UX terrible. Si intentas "recuperarte con elegancia" cuando una consulta a la base de datos devuelve un estado imposible—como un saldo de cuenta negativo—corrompes tus datos silenciosamente.

## Regla 2: En el Borde, Trata los Errores como Valores

Cuando los datos llegan del mundo exterior (entrada de API, lecturas de BD), no son confiables. Como *esperamos* datos malos, lanzar excepciones es un anti-patrón. `raise` es esencialmente una instrucción `GOTO` oculta que destruye la seguridad de tipos de Python (la excepción en un bloque `except` no tiene información estática de tipos sobre de dónde vino).

En su lugar, usamos el **patrón `Ok / Err`** (el patrón `Result`), muy inspirado en Go y Rust. Tratamos los errores como valores de retorno estándar.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Ok[T]:
    """Un resultado exitoso que envuelve un valor de tipo T."""
    value: T


@dataclass(frozen=True)
class Err[E]:
    """Un resultado fallido que envuelve un error de tipo E."""
    error: E


# Alias de tipo en Python 3.12+ con parámetros genéricos
type Result[T, E] = Ok[T] | Err[E]


# ✅ Envuelve el caótico borde en un Result predecible
async def parse_json(req: Request) -> Result[dict, str]:
    try:
        return Ok(await req.json())
    except Exception:
        return Err("Malformed JSON")
```

Al devolver un `Result`, el verificador de tipos *obliga* al llamador a manejar el fallo antes de que se le permita acceder a `value`. Hemos eliminado la fuga silenciosa del límite.

Uso con pattern matching estructural (Python 3.10+):

```python
match await parse_json(req):
    case Ok(body):
        # aquí body está completamente tipado como `dict`
        ...
    case Err(error):
        return JSONResponse({"error": error}, status_code=400)
```

## Regla 3: Parsea, No Valides

Ahora que hemos analizado el JSON de forma segura, debemos asegurarnos de que tenga la forma correcta. Pero la validación tradicional tiene un defecto masivo: **no deja un recibo.**

Si escribes una función `is_valid_email(input: str) -> bool`, y devuelve `True`, Python aún solo ve un `str`. Si pasas esa cadena por otros cinco archivos, ninguno de esos archivos *sabe* que fue validada. Entonces, los desarrolladores de nivel intermedio re-validan defensivamente la cadena en todas partes.

Para arreglar esto, usamos el paradigma **"Parsea, No Valides"** [^1].
Usamos analizadores de esquemas (como Pydantic [^2]) combinados con **Branded Types** (`NewType`) para hacer que los estados inválidos sean *imposibles de representar*.

```python
import uuid
from dataclasses import dataclass
from typing import NewType

from pydantic import BaseModel, field_validator, ValidationError

# 1. Los Branded Types (Las Pulseras VIP)
#    NewType tiene costo cero en tiempo de ejecución: solo existe para el verificador de tipos.
ValidatedUserId = NewType("ValidatedUserId", str)
PositiveAmount = NewType("PositiveAmount", float)


# 2. El esquema de Pydantic (valida la entrada cruda)
class _RefundInput(BaseModel):
    user_id: str
    amount: float

    @field_validator("user_id")
    @classmethod
    def _must_be_uuid(cls, v: str) -> str:
        uuid.UUID(v)  # lanza ValueError si es inválido
        return v

    @field_validator("amount")
    @classmethod
    def _must_be_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError("Amount must be positive")
        return v


# 3. El Modelo de Dominio de Confianza (solo construible a partir de datos validados)
@dataclass(frozen=True)
class TrustedRefund:
    user_id: ValidatedUserId
    amount: PositiveAmount


# 4. El Constructor Inteligente (El Guardia de la puerta)
#    Esta función no solo comprueba datos; los transforma y los Marca.
def parse_refund_request(data: dict) -> Result[TrustedRefund, list[dict]]:
    try:
        raw = _RefundInput.model_validate(data)  # Parsea y valida
        return Ok(TrustedRefund(
            user_id=ValidatedUserId(raw.user_id),   # Márcalo
            amount=PositiveAmount(raw.amount),       # Márcalo
        ))
    except ValidationError as e:
        return Err(e.errors(include_input=False))
```

**Por qué esto funciona:** Después de que los datos pasan por `parse_refund_request`, el `TrustedRefund.user_id` devuelto está tipado como `ValidatedUserId`, no como `str`. Cualquier función en tu dominio central que acepte `ValidatedUserId` está matemáticamente garantizada a recibir solo un UUID validado. No puedes pasar accidentalmente una cadena cruda y sin validar: el verificador de tipos la rechazará.

## Regla 4: Dentro del Núcleo, Afirma y Bloquea

Una vez que los datos han pasado el Constructor Inteligente, están dentro de nuestro dominio de confianza. Ya no deberíamos manejar errores de validación esperados. Estamos lidiando con invariantes internos del sistema.

Si una suposición es incorrecta aquí, *queremos* bloquear. El `TypeGuard` de Python hace esto poderoso al reducir los tipos de forma permanente para el resto del alcance.

```python
from typing import TypeGuard, TypeVar

T = TypeVar("T")
E = TypeVar("E")


def is_ok(result: Result[T, E]) -> TypeGuard[Ok[T]]:
    """Predicado de reducción de tipos: después de `if is_ok(x)`, x es `Ok[T]`."""
    return isinstance(result, Ok)


class InternalError(Exception):
    """Se lanza cuando se viola un invariante del sistema. Siempre es un 500."""


def assert_ok(result: Result[T, E]) -> Ok[T]:
    """Aserción fail-fast para zonas de confianza.

    A diferencia del `assert` integrado de Python, esto NO se desactiva con -O.
    Si esto se dispara, significa que tu límite de validación tiene un agujero.
    """
    if not isinstance(result, Ok):
        raise InternalError(str(result.error))
    return result
```

**Una nota sobre `assert`:** La sentencia `assert` integrada de Python se elimina cuando el intérprete se ejecuta con `-O` (optimizaciones). Nunca la uses para invariantes del sistema. Usa siempre una función de aserción personalizada como `assert_ok` de arriba que lanza incondicionalmente.

## La Gran Arquitectura: Capas de Confianza

Veamos el código "Antes" del inicio de este artículo, refactorizado en la arquitectura de **Capas de Confianza**.

```python
import logging
from typing import TypeVar, TypeGuard

logger = logging.getLogger(__name__)


# --- Infraestructura compartida ---

T = TypeVar("T")
E = TypeVar("E")


def is_ok(result: Result[T, E]) -> TypeGuard[Ok[T]]:
    return isinstance(result, Ok)


def assert_ok(result: Result[T, E]) -> Ok[T]:
    if not isinstance(result, Ok):
        raise InternalError(str(result.error))
    return result


def log_critical_bug(error: Exception) -> None:
    logger.critical("System invariant violated", exc_info=error)


# ----------------------------------------------------------------
# --- 1. EL NÚCLEO DEL DOMINIO (¡Cero comprobaciones!)        ---
#     Como exigimos Branded Types, es matemáticamente            ---
#     imposible llamar a esta función con datos sucios.          ---
# ----------------------------------------------------------------

async def execute_refund(user_id: ValidatedUserId, amount: PositiveAmount) -> None:
    # get_user también ha sido refactorizado para devolver Result[User, DbError]
    # — las lecturas de BD están en el "borde" también, así que reciben el mismo tratamiento.
    db_result = await get_user(user_id)

    # ASERCIÓN: Asumimos que la BD funciona y que el usuario existe.
    # Si no, nuestro sistema está roto. ¡Fail-fast!
    ok_user = assert_ok(db_result)

    # El verificador de tipos garantiza que ok_user.value es nuestro objeto User.
    await gateway.refund(ok_user.value.stripe_id, amount)


# ----------------------------------------------------------------
# --- 2. EL CONTROLADOR DEL BORDE                             ---
# ----------------------------------------------------------------

@app.post("/refund")
async def process_refund_route(req: Request):
    # Capa 1: Analiza el caótico mundo exterior de forma segura
    json_result = await parse_json(req)
    if not is_ok(json_result):
        return JSONResponse({"error": json_result.error}, status_code=400)

    # Capa 2: Constructor Inteligente (Valida y Marca)
    payload_result = parse_refund_request(json_result.value)
    if not is_ok(payload_result):
        return JSONResponse({"error": payload_result.error}, status_code=400)

    # Capa 3: Ejecuta la Lógica Central
    try:
        # Pydantic garantiza que payload_result.value contiene nuestros Branded Types!
        refund = payload_result.value
        await execute_refund(refund.user_id, refund.amount)
        return JSONResponse({"message": "Success"}, status_code=200)
    except InternalError as error:
        # Si llegamos aquí, una ASERCIÓN interna se disparó.
        # Esto es un bug real. ¡Avísale al desarrollador!
        log_critical_bug(error)
        return JSONResponse({"error": "Internal Server Error"}, status_code=500)
```

## La Conclusión

Mira la función `execute_refund` de arriba.
Es completamente pura.
No hay sentencias `if` comprobando cadenas vacías.
No hay comprobaciones `isinstance`.
**Tu carga cognitiva baja a cero.**

Para dejar de pelear contra Python y empezar a aprovecharlo como una herramienta arquitectónica, memoriza este paradigma:

1. **La Validación es Defensiva.** Esperas que el mundo exterior sea desordenado. Usa el patrón `Ok / Err` para analizar datos, recuperarte con elegancia y devolver errores de nivel 400 amigables para el usuario.
2. **La Aserción es Ofensiva.** Esperas que tu lógica interna sea impecable. Usa `assert_ok` para fallar rápido, bloquear y devolver errores de nivel 500 cuando se rompen los invariantes del sistema.
3. **Codifica la Confianza en Tipos.** Empuja la validación a los bordes, usa Constructores Inteligentes para crear Branded Types (`NewType`), haz que los estados inválidos sean irrepresentables y elimina el resto de tus comprobaciones en tiempo de ejecución.

La corrección es tan importante que cualquier violación de la lógica interna es un bug.
<mark>Construye límites estrictos, confía en tus tipos y mira cómo tu código se vuelve infinitamente más resiliente.</mark>

[^1]: Este concepto fue popularizado por Alexis King en su ensayo seminal [Parse, Don't Validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/).
[^2]: [Pydantic](https://docs.pydantic.dev/) es la librería de facto en Python para validación de datos usando anotaciones de tipos. Es el equivalente en el ecosistema Python de Zod.
