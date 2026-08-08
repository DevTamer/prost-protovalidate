# prost-protovalidate

[![Crates.io](https://img.shields.io/crates/v/prost-protovalidate.svg)](https://crates.io/crates/prost-protovalidate)
[![docs.rs](https://img.shields.io/docsrs/prost-protovalidate)](https://docs.rs/prost-protovalidate)
[![Licencia](https://img.shields.io/badge/license-MIT%2FApache--2.0-blue.svg)](LICENSE-MIT)
[![MSRV](https://img.shields.io/badge/MSRV-1.86-blue.svg)](https://blog.rust-lang.org/2025/04/03/Rust-1.86.0.html)

[English](README.md) · **Español**

Validación de mensajes Protocol Buffer utilizando las reglas de [buf.validate](https://github.com/bufbuild/protovalidate) — un motor de ejecución basado en `prost-reflect`, junto con una **ruta de validación en tiempo de compilación a través de [`prost-protovalidate-build`](crates/prost-protovalidate-build/)** que es compatible tanto con los tipos de mensajes de [`prost`](https://crates.io/crates/prost) como de [`buffa`](https://crates.io/crates/buffa).

Por defecto, `prost-protovalidate` evalúa las restricciones de `buf.validate` (incluidas las expresiones de Common Expression Language, CEL) en tiempo de ejecución mediante `prost-reflect`, aplicando reglas directamente desde tu única fuente de verdad: los archivos `.proto`. Para mensajes con reglas estándar (no CEL), `prost-protovalidate-build` genera `impl Validate` en tiempo de compilación, lo que permite que la validación se realice mediante un acceso directo a campos monomorfizados: la ruta más rápida en tiempo de ejecución, sin reflexión, sin intérprete CEL en el camino crítico. En el backend de buffa, los mensajes con CEL también pueden ser redirigidos al motor de ejecución (`runtime_bridge`), proporcionando a los tipos de buffa la superficie **completa** de reglas de `buf.validate` desde una sola biblioteca.

## Características principales

- **Validación en tiempo de compilación (ruta más rápida en tiempo de ejecución)**: [`prost-protovalidate-build`](crates/prost-protovalidate-build/) genera `impl Validate` para mensajes con reglas estándar únicamente; sin transcodificación de `prost-reflect`, sin intérprete CEL, con acceso directo a campos mediante código monomorfizado. Combinado con `default-features = false` en `prost-protovalidate`, todo el subárbol de dependencias `cel` / `chrono` / `pastey` / `thiserror` 1.x desaparece de la compilación. Esto permite intercambiar tamaño binario y tiempo de compilación por velocidad en el camino crítico. Los mensajes con CEL recurren automáticamente al `Validator` de tiempo de ejecución (con un diagnóstico `cargo:warning=`; nunca se omiten en silencio).
- **Validación en tiempo de ejecución con CEL**: Aprovecha `prost-reflect` para inspeccionar dinámicamente los campos de los mensajes y evaluar reglas de validación complejas, incluidas expresiones CEL arbitrarias, en tiempo de ejecución.
- **Evaluación de CEL**: Admite por completo la compilación y evaluación de condiciones de Common Expression Language (CEL) para restricciones cruzadas o complejas.
- **El proto es la única fuente de verdad**: Define reglas de validación dentro de los archivos `.proto` utilizando `buf.validate` y aplícalas automáticamente en Rust.
- **Una biblioteca para `prost` y `buffa`**: Las mismas reglas de `buf.validate`, el mismo resultado de violaciones, independientemente de qué runtime de protobuf genere tus tipos. Usa `Builder::backend(Backend::Buffa)` para buffa (`buffa-build` o un envoltorio como `connectrpc-build`).
- **Dos motores, una fuente de reglas**: El `Validator` de tiempo de ejecución y el generador en tiempo de compilación consumen los IDs y mensajes de reglas canónicos en `prost-protovalidate-types::rules_meta`, por lo que no pueden divergir; un barrido de paridad impulsado por descriptores asegura violaciones idénticas en ambos motores.
- **Conformidad verificada por harness**: El motor de tiempo de ejecución pasa la suite de conformidad completa de protovalidate upstream: **2872/2872** pruebas (protovalidate v1.2.2, 0 fallos esperados).
- **Enrutamiento fail-closed o de cobertura total**: `fail_on_runtime_only(true)` convierte cualquier regla que el generador no pueda cubrir en un error de compilación definitivo (para servicios que no incluyen ningún validador en tiempo de ejecución); en buffa, `runtime_bridge(true)` delega esos mensajes al motor de ejecución embebido para cobertura completa de CEL.
- **Tres niveles de huella**: completo (`cel`, predeterminado) ⊃ `reflect` (`Validator` de tiempo de ejecución sin CEL) ⊃ slim (`default-features = false`; validación solo en tiempo de compilación, sin `prost-reflect` en el grafo de dependencias).
- **Integración gRPC**: Característica opcional `tonic` que mapea `ValidationError` a `tonic::Status`, y `tonic-types` adjunta detalles a nivel de campo de `google.rpc.BadRequest`.
- **Soporte para Edición 2023**: Normaliza descriptores de Edición 2023 (incluida la codificación de grupos `DELIMITED`) para que `prost-reflect` 0.16 los maneje correctamente.
- **Crates modulares**: Separación clara entre tipos protobuf puros (`prost-protovalidate-types`) y el motor de validación de tiempo de ejecución (`prost-protovalidate`).

## Crates

| Crate                                                          | Propósito                                                         | Sección de Cargo          |
| -------------------------------------------------------------- | --------------------------------------------------------------- | ---------------------- |
| [prost-protovalidate-types](crates/prost-protovalidate-types/) | Tipos proto de `buf.validate` con soporte para prost y prost-reflect; aloja `rules_meta` (IDs y mensajes de reglas canónicos para ambos motores) | `[dependencies]`       |
| [prost-protovalidate](crates/prost-protovalidate/)             | Motor de validación de tiempo de ejecución (análisis de CEL, evaluación de restricciones)  | `[dependencies]`       |
| [prost-protovalidate-build](crates/prost-protovalidate-build/) | Generador de código en tiempo de compilación para validación                      | `[build-dependencies]` |

## Inicio rápido

Agrega las dependencias a tu `Cargo.toml`:

```toml
[dependencies]
prost = "0.14"
prost-protovalidate = "0.6"
```

### Flags de características

| Característica       | Predeterminada         | Descripción                                                                                                                                                            |
| ------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cel`         | Sí              | Evaluación de expresiones CEL y soporte para tiempo `chrono` (implica `reflect`)                                                                                          |
| `reflect`     | Sí (vía `cel`)  | Reflexión en tiempo de ejecución: el `Validator` impulsado por descriptores, filtros de validación, hidratación de la ruta de regla en `Violation` y el módulo `bridge` usado por el código generado con `runtime_bridge` |
| `tonic`       | No              | Integración gRPC: `From<ValidationError> for tonic::Status` y un trait de extensión `ValidateRequest` (`req.validate_inner()?` en manejadores)                       |
| `tonic-types` | No              | Implica `tonic`. Adjunta detalles de `google.rpc.BadRequest` a los estados de fallo de validación para que los clientes puedan analizar errores a nivel de campo sin tener que extraerlos de la cadena del mensaje |

Para mantener el `Validator` de tiempo de ejecución pero deshabilitar CEL (elimina `cel`, `chrono`, `pastey` y las dependencias transitivas de `thiserror` 1.x):

```toml
[dependencies]
prost-protovalidate = { version = "0.6", default-features = false, features = ["reflect"] }
```

Sin `cel`, las reglas estándar (comprobaciones de rango, restricciones de cadena, validadores de formato, etc.) funcionan normalmente. Los mensajes con expresiones CEL o reglas CEL predefinidas producirán un `CompilationError` en tiempo de validación.

Para validación solo en tiempo de compilación (el `impl Validate` generado por `prost-protovalidate-build`), elimina todas las características — la compilación slim mantiene solo el trait `Validate`, `Violation`/`ValidationError` y los ayudantes `validators`, **sin `prost-reflect` en el grafo de dependencias**:

```toml
[dependencies]
prost-protovalidate = { version = "0.6", default-features = false }
```

### Uso

Anota tus archivos `.proto` con reglas de `buf.validate`:

```protobuf
syntax = "proto3";

import "buf/validate/validate.proto";

message CreateUserRequest {
  string name = 1 [(buf.validate.field).string.min_len = 1];
  int32 age = 2 [(buf.validate.field).int32.gte = 18];
}
```

En tu código Rust, ejecuta el validador contra un mensaje generado por `prost` usando el atajo `validate` proporcionado:

```rust
use prost_protovalidate::validate;

let request = CreateUserRequest {
    name: "Alice".to_string(),
    age: 25,
};

match validate(&request) {
    Ok(_) => println!("Validation passed!"),
    Err(e) => println!("Validation failed: {}", e),
}
```

O guarda una instancia de `Validator` si necesitas realizar múltiples validaciones eficientemente:

```rust
use prost_protovalidate::Validator;

// Crear un Validator analiza y almacena en caché las reglas internamente
let validator = Validator::new();
validator.validate(&request)?;
```

## Compatibilidad

| prost-protovalidate | prost | prost-reflect | MSRV |
| ------------------- | ----- | ------------- | ---- |
| 0.6.x               | 0.14  | 0.16          | 1.86 |
| 0.5.x               | 0.14  | 0.16          | 1.86 |
| 0.4.x               | 0.14  | 0.16          | 1.86 |

## Modos de validación

Tres modos de validación cubren diferentes casos de uso:

### Validación en tiempo de compilación (más rápida en tiempo de ejecución, sin CEL)

Para mensajes con **solo** reglas estándar (sin CEL). Los validadores se generan en tiempo de compilación y se ejecutan mediante acceso directo a campos monomorfizados: **sin transcodificación de `prost-reflect`, sin intérprete CEL, sin despacho dinámico** en el camino crítico. Combinado con `default-features = false` en `prost-protovalidate`, todo el subárbol `cel` / `chrono` / `pastey` / `thiserror` 1.x desaparece de la compilación. Requiere `prost-protovalidate-build` en `[build-dependencies]`.

El código generado apunta a tipos de mensajes `prost` por defecto; usa `Builder::backend(Backend::Buffa)` cuando tus tipos provienen de [buffa](https://crates.io/crates/buffa) (`buffa-build` o un envoltorio como `connectrpc-build`). Con `Builder::fail_on_runtime_only(true)`, cualquier mensaje que el generador no pueda cubrir hace fallar la compilación en lugar de recurrir al validador de tiempo de ejecución — para consumidores que no incluyen ningún validador en tiempo de ejecución.

En el backend de buffa, `Builder::runtime_bridge(true)` es la tercera opción: mensajes que el generador no puede cubrir (reglas CEL, formas no admitidas) reciben un `impl Validate` generado que los delega a un motor de ejecución embebido, de modo que **cada** mensaje obtiene una llamada uniforme a `msg.validate()`, reglas estándar en línea y todo lo demás a través del mismo `Validator` probado en conformidad. Los mensajes redirigidos requieren `prost-protovalidate` con `reflect`/`cel` habilitados y se ejecutan a velocidad de intérprete; mantén `fail_on_runtime_only` cuando quieras la garantía slim, libre de CEL.

```rust
use prost_protovalidate::Validate;

msg.validate()?;
```

### Validación en tiempo de ejecución con CEL (conformidad completa)

Para mensajes con expresiones CEL o reglas mixtas. Usa el despacho dinámico de `prost-reflect` y el intérprete CEL en tiempo de ejecución.

```rust
use prost_protovalidate::Validator;

let validator = Validator::new();
validator.validate(&msg)?;
```

### Validación en tiempo de ejecución sin CEL (ligera)

Para servicios que no usan CEL. Deshabilitar la característica `cel` elimina `cel`, `chrono`, `pastey` y las dependencias transitivas de `thiserror` 1.x. Combina de forma natural con los validadores en tiempo de compilación descritos arriba para un árbol de dependencias libre de CEL de principio a fin.

```toml
[dependencies]
prost-protovalidate = { version = "0.6", default-features = false }
```

Las reglas estándar funcionan normalmente; los mensajes con reglas CEL producen un `CompilationError`.

### ¿Qué modo para qué mensaje?

| Reglas proto                                    | ¿Trait `Validate` generado? | Cómo validar                          |
| ---------------------------------------------- | --------------------------- | ---------------------------------------- |
| Solo estándar (min_len, gte, email, etc.)      | Sí                         | `msg.validate()` (en tiempo de compilación, más rápido) |
| Timestamp relativo al tiempo actual (lt_now, gt_now, within)| Sí (lee `SystemTime::now()`) | `msg.validate()` (en tiempo de compilación, más rápido) |
| `repeated.unique` en float/double              | Sí (bits IEEE-754 canónicos) | `msg.validate()` (en tiempo de compilación, más rápido) |
| CEL solo (expresiones)                         | No¹                         | `validator.validate(&msg)` (tiempo de ejecución)     |
| Mixto (estándar + CEL)                         | No¹                         | `validator.validate(&msg)` (tiempo de ejecución)     |
| Dependencias anidadas solo en tiempo de ejecución | No¹                         | `validator.validate(&msg)` (tiempo de ejecución)     |
| Sin reglas                                       | No                          | —                                        |

¹ En el backend de buffa con `runtime_bridge(true)`, estos mensajes **sí** reciben un `impl Validate` generado que los delega al motor de ejecución embebido, y así `msg.validate()` funciona de manera uniforme en todo el esquema.

Para mensajes con reglas de timestamp relativas al tiempo actual, la implementación generada de `Validate::validate(&self)` lee `SystemTime::now()` una vez por llamada — idéntico al comportamiento predeterminado del `Validator` de tiempo de ejecución. Las pruebas que necesiten un reloj determinista deben usar el `Validator` de tiempo de ejecución y sobrescribir `now_fn` en `ValidatorOptions`; la firma del trait no tiene dónde inyectar una fuente de reloj.

### Integración gRPC (tonic)

Habilita la característica `tonic` para mapear errores de validación a estados gRPC:

```toml
[dependencies]
prost-protovalidate = { version = "0.6", features = ["tonic"] }
tonic = "0.14"
```

En tu manejador de servicio:

```rust,ignore
use prost_protovalidate::tonic::ValidateRequest;

#[tonic::async_trait]
impl my_proto::greeter_server::Greeter for GreeterImpl {
    async fn say_hello(
        &self,
        req: tonic::Request<my_proto::HelloRequest>,
    ) -> Result<tonic::Response<my_proto::HelloReply>, tonic::Status> {
        req.validate_inner()?;
        // ... cuerpo del manejador ...
    }
}
```

`ValidateRequest` está implementado para manejadores unarios y de streaming de servidor (`tonic::Request<T>`). Para manejadores de streaming de cliente y bidireccionales (`tonic::Request<tonic::Streaming<T>>`), valida cada mensaje dentro del bucle de mensajes: `msg.validate().map_err(tonic::Status::from)?`.

`ValidationError` mapea a `Code::InvalidArgument`. `CompilationError` / `RuntimeError` (y las variantes unificadas `Error::Compilation` / `Error::Runtime`) mapean a `Code::Internal` con un **mensaje fijo, genérico** — las cadenas `cause` subyacentes pueden contener nombres de campos proto y detalles internos de CEL que no deben ser expuestos a clientes no confiables. Registra el error original del lado del servidor antes de convertir si necesitas la causa completa.

Habilitar la característica adicional `tonic-types` adjunta un detalle de `google.rpc.BadRequest` a los estados de fallo de validación, con un `FieldViolation` por violación, para que los clientes puedan analizar errores a nivel de campo de forma programática:

```toml
[dependencies]
prost-protovalidate = { version = "0.6", features = ["tonic-types"] }
```

## Generación de código en tiempo de compilación

Agrega `prost-protovalidate-build` a tus dependencias de compilación:

```toml
[dependencies]
prost = "0.14"
prost-protovalidate = "0.6"

[build-dependencies]
prost-build = "0.14"
prost-protovalidate-build = "0.6"
```

En tu `build.rs`:

```rust,no_run
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let descriptor_path = std::path::PathBuf::from(std::env::var("OUT_DIR")?)
        .join("file_descriptor_set.bin");

    prost_build::Config::new()
        .file_descriptor_set_path(&descriptor_path)
        .compile_protos(&["proto/service.proto"], &["proto/"])?;

    prost_protovalidate_build::Builder::new()
        .file_descriptor_set_path(&descriptor_path)?
        .compile()?;

    Ok(())
}
```

Incluye los validadores generados junto con tu código generado por prost:

```rust,ignore
include!(concat!(env!("OUT_DIR"), "/validate_impl.rs"));
```

Los mensajes que el generador no puede cubrir (reglas CEL o reglas CEL predefinidas, y mensajes que dependen transitivamente de validación anidada solo en tiempo de ejecución) se enrutan mediante uno de tres modos:

- **Predeterminado** — omitido con un `cargo:warning` explicando por qué; valídalos con el `Validator` de tiempo de ejecución.
- **`fail_on_runtime_only(true)`** — en su lugar, la compilación falla, de forma que una regla nunca pueda omitirse en silencio (para consumidores que no incluyen ningún validador en tiempo de ejecución).
- **`runtime_bridge(true)`** (backend buffa) — un `impl Validate` generado los delega a un motor de ejecución embebido, dando a todo el esquema una llamada uniforme a `msg.validate()`, incluido el soporte completo para CEL.

## Desarrollo

Compilar desde el código fuente requiere `protoc` (el script de compilación de `prost-protovalidate-types` compila el esquema `buf.validate` empaquetado). Instálalo con tu gestor de paquetes o establece la variable de entorno `PROTOC` a la ruta del binario.

```bash
make pre-commit     # verificación de formato + lint + pruebas (ejecutar antes de confirmar los cambios)
make conformance    # suite de conformidad (requiere Go)
```

## Conformidad

Conformidad completa con la suite de pruebas de bufbuild protovalidate (v1.2.2): **2872/2872 pruebas superadas** (0 fallos esperados).

La conformidad usa un harness upstream con versión fijada de `github.com/bufbuild/protovalidate/tools/protovalidate-conformance`.

Las versiones fijadas están definidas en el `Makefile` de la raíz del repositorio:

- `PROTOVALIDATE_TOOLS_VERSION`
- `PROTOVALIDATE_SCHEMA_REF`

### Plan de actualización

1. Incrementa `PROTOVALIDATE_TOOLS_VERSION` y `PROTOVALIDATE_SCHEMA_REF` en `Makefile`.
2. Sincroniza `validate.proto` usando el flujo de trabajo en `crates/prost-protovalidate-types/SYNC.md`.
3. Ejecuta la suite de conformidad y actualiza `expected_failures.yaml` con cualquier nuevo fallo:
   - `make conformance`
4. Ejecuta pruebas de regresión:
   - `cargo test --all-features`
5. Confirma en un mismo commit el incremento de versión y los cambios en `expected_failures.yaml`.

## Licencia

[MIT](LICENSE-MIT) O [Apache-2.0](LICENSE-APACHE)
