---
title: "Deja de Validar en Todas Partes: Una Guía Arquitectónica para el Manejo de Errores en TypeScript"
description: "Aprende a construir aplicaciones TypeScript resilientes distinguiendo entre validación y aserciones, implementando el patrón Result y usando Branded Types para hacer que los estados inválidos sean irrepresentables."
date: 2026-04-13
categories:
    - Software Architecture
    - TypeScript
    - Development Patterns
tags:
    - Error Handling
    - Type Safety
    - Functional Programming
    - Zod
    - Branded Types
    - Clean Code
---


> *Un desarrollador junior escribe código que asume que todo saldrá bien (sin comprobaciones).*
> *Un desarrollador intermedio escribe código que asume que todo saldrá mal (comprobaciones defensivas y aserciones por todas partes).*
> *Un desarrollador senior usa el sistema de tipos para hacer imposible el error, eliminando el 80% de esas comprobaciones por completo.*<br>
> — <cite>Proverbio de Ingeniería de Software</cite>

<!--more-->

## 📌 TL;DR

* **Valida en los bordes (defensivo):** Espera datos malos del mundo exterior. Manejarlos con elegancia usando el patrón `val / ok` (el patrón `Result`) en lugar de lanzar errores impredecibles con `try/catch`.
* **Afirma en el núcleo (ofensivo):** Espera datos perfectos internamente. Si tu estado interno es incorrecto, es un bug del sistema. Usa la palabra clave `asserts` de TypeScript para fallar de inmediato (fail-fast).
* **Parsea, no valides:** No solo compruebes los datos; transfórmalos en el límite en **Branded Types** (p. ej., `EmailAddress`, `ValidatedUserId`).
* **El Result:** Tus funciones de lógica de negocio central *solo* aceptan Branded Types. Esto hace que los estados inválidos sean matemáticamente imposibles de representar, permitiéndote eliminar cientos de líneas de comprobaciones defensivas `if (!data)`.

---

Si has pasado suficiente tiempo escribiendo Node.js o TypeScript, es probable que hayas escrito una función exactamente como esta:

```typescript
// ❌ El Controlador "Defensivo" de Nivel Intermedio
async function processRefund(req: Request, res: Response) {
  try {
    const body = await req.json(); // ⚠️ ¡Lanza si el JSON está mal formado!

    // Comprobaciones defensivas por todas partes...
    if (!body || typeof body !== 'object') {
      throw new Error("Invalid payload");
    }
    if (!body.userId || typeof body.userId !== 'string') {
      throw new Error("Missing UserId");
    }
    if (typeof body.amount !== 'number' || body.amount <= 0) {
      throw new Error("Invalid amount");
    }

    const user = await db.getUser(body.userId);
    if (!user) {
      throw new Error("User not found"); // ¿Error esperado? ¿O bug de la BD?
    }

    // Finalmente... la lógica de negocio real
    await gateway.refund(user.stripeId, body.amount);

    return res.status(200).send("Success");
  } catch (error) {
    // ⚠️ ¿Es un error 400 de petición incorrecta? ¿O una falla 500 de la BD? Ya no lo sabemos.
    return res.status(400).send({ error: error.message });
  }
}
```

Este código es agotador de leer. La lógica de negocio central está enterrada bajo una montaña de sentencias defensivas `if/throw`. Además, el bloque `catch` no tiene idea de si el error fue un usuario escribiendo mal (400 Bad Request) o la base de datos incendiándose (500 Internal Error).

El manejo de errores no se trata solo de atrapar errores; **se trata de arquitectura de sistemas**.

Para arreglar esto, necesitamos entender la diferencia fundamental entre **Validación** y **Aserción**, adoptar el **patrón Result** y usar **Branded Types** para empujar los errores a los bordes absolutos de nuestro sistema.

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

Cuando los datos llegan del mundo exterior (entrada de API, lecturas de BD), no son confiables. Como *esperamos* datos malos, lanzar excepciones es un anti-patrón. `throw` es esencialmente una instrucción `GOTO` oculta que destruye la seguridad de tipos de TypeScript (el error en un bloque catch siempre está tipado como `unknown`).

En su lugar, usamos el **patrón `val / ok`** (el patrón `Result`), muy inspirado en Go y Rust. Tratamos los errores como valores de retorno estándar.

```typescript
// El Plano del Tipo Result
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

// ✅ Envuelve el caótico borde en un Result predecible
async function parseJson(req: Request): Promise<Result<unknown, string>> {
  try {
    return { ok: true, value: await req.json() };
  } catch (err) {
    return { ok: false, error: "Malformed JSON" };
  }
}
```

Al devolver un `Result`, el compilador de TypeScript *obliga* al llamador a manejar el fallo antes de que se le permita acceder a `value`. Hemos eliminado la fuga silenciosa del límite.

## Regla 3: Parsea, No Valides

Ahora que hemos analizado el JSON de forma segura, debemos asegurarnos de que tenga la forma correcta. Pero la validación tradicional tiene un defecto masivo: **no deja un recibo.**

Si escribes una función `isValidEmail(input): boolean`, y devuelve `true`, TypeScript aún solo ve un `string`. Si pasas esa cadena por otros cinco archivos, ninguno de esos archivos *sabe* que fue validada. Entonces, los desarrolladores de nivel intermedio re-validan defensivamente la cadena en todas partes.

Para arreglar esto, usamos el paradigma **"Parsea, No Valides"** [^1].
Usamos analizadores de esquemas (como Zod [^2]) combinados con **Branded Types** para hacer que los estados inválidos sean *imposibles de representar*.

```typescript
import { z } from "zod";

// 1. Los Branded Types (Las Pulseras VIP)
type ValidatedUserId = string & { readonly __brand: unique symbol };
type PositiveAmount = number & { readonly __brand: unique symbol };

// 2. El Constructor Inteligente (El Guardia de la puerta)
// Este esquema no solo comprueba datos; los transforma y los Marca.
const RefundSchema = z.object({
  userId: z.string().uuid().transform(val => val as ValidatedUserId),
  amount: z.number().positive().transform(val => val as PositiveAmount)
});

// Un helper para convertir Zod en nuestro patrón Result
function parseRefundRequest(data: unknown): Result<z.infer<typeof RefundSchema>, string> {
  const parsed = RefundSchema.safeParse(data);
  if (!parsed.success) return { ok: false, error: parsed.error.message };

  return { ok: true, value: parsed.data };
}
```

## Regla 4: Dentro del Núcleo, Afirma y Bloquea

Una vez que los datos han pasado el Constructor Inteligente, están dentro de nuestro dominio de confianza. Ya no deberíamos manejar errores de validación esperados. Estamos lidiando con invariantes internos del sistema.

Si una suposición es incorrecta aquí, *queremos* bloquear. TypeScript hace esto poderoso con la palabra clave `asserts`, que reduce los tipos de forma permanente para el resto del alcance.

```typescript
// Una utilidad para conectar los Result de vuelta a las aserciones en zonas de confianza
function assertOk<T>(result: Result<T, Error>): asserts result is { ok: true; value: T } {
  if (!result.ok) {
    // ¡BLOQUEO! El guardia VIP detiene la fiesta.
    throw result.error;
  }
}
```

## La Gran Arquitectura: Capas de Confianza

Veamos el código "Antes" del inicio de este artículo, refactorizado en la arquitectura de **Capas de Confianza**.

```typescript
// --- 1. EL NÚCLEO DEL DOMINIO (¡Cero comprobaciones de validación!) ---
// Como exigimos Branded Types, es matemáticamente imposible
// llamar a esta función con datos sucios.
async function executeRefund(userId: ValidatedUserId, amount: PositiveAmount) {
  const dbResult = await db.getUser(userId);

  // ASERCIÓN: Asumimos que la BD funciona y que el usuario existe.
  // Si no, nuestro sistema está roto. ¡Fail-fast!
  assertOk(dbResult);

  // TypeScript garantiza que dbResult.value es nuestro objeto User.
  await gateway.refund(dbResult.value.stripeId, amount);
}

// --- 2. EL CONTROLADOR DEL BORDE ---
async function processRefundRoute(req: Request, res: Response) {
  // Capa 1: Analiza el caótico mundo exterior de forma segura
  const jsonResult = await parseJson(req);
  if (!jsonResult.ok) {
    return res.status(400).send({ error: jsonResult.error });
  }

  // Capa 2: Constructor Inteligente (Valida y Marca)
  const payloadResult = parseRefundRequest(jsonResult.value);
  if (!payloadResult.ok) {
    return res.status(400).send({ error: payloadResult.error });
  }

  // Capa 3: Ejecuta la Lógica Central
  try {
    // TS sabe que payloadResult.value contiene nuestros Branded Types!
    await executeRefund(payloadResult.value.userId, payloadResult.value.amount);
    return res.status(200).send("Success");
  } catch (error) {
    // Si llegamos aquí, una ASERCIÓN interna se disparó.
    // Esto es un bug real. ¡Avísale al desarrollador!
    logCriticalBug(error);
    return res.status(500).send("Internal Server Error");
  }
}
```

## La Conclusión

Mira la función `executeRefund` de arriba.
Es completamente pura.
No hay sentencias `if` comprobando cadenas vacías.
No hay comprobaciones `typeof`.
**Tu carga cognitiva baja a cero.**

Para dejar de pelear contra TypeScript y empezar a aprovecharlo como una herramienta arquitectónica, memoriza este paradigma:

1. **La Validación es Defensiva.** Esperas que el mundo exterior sea desordenado. Usa el patrón `val / ok` para analizar datos, recuperarte con elegancia y devolver errores de nivel 400 amigables para el usuario.
2. **La Aserción es Ofensiva.** Esperas que tu lógica interna sea impecable. Usa `asserts` para fallar rápido, bloquear y devolver errores de nivel 500 cuando se rompen los invariantes del sistema.
3. **Codifica la Confianza en Tipos.** Empuja la validación a los bordes, usa Constructores Inteligentes para crear Branded Types, haz que los estados inválidos sean irrepresentables y elimina el resto de tus comprobaciones en tiempo de ejecución.

La corrección es tan importante que cualquier violación de la lógica interna es un bug.
<mark>Construye límites estrictos, confía en tus tipos y mira cómo tu código se vuelve infinitamente más resiliente.</mark>

[^1]: Este concepto fue popularizado por Alexis King en su ensayo seminal [Parse, Don't Validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/).
[^2]: [Zod](https://zod.dev/) es una librería de declaración y validación de esquemas enfocada en TypeScript.
