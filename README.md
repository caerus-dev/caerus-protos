# Caerus Protocol Buffer Definitions (`caerus-protos`)

[![Protobuf v3](https://img.shields.io/badge/Protobuf-v3-blue.svg)](https://protobuf.dev/)
[![gRPC over HTTP/2](https://img.shields.io/badge/gRPC-HTTP%2F2-green.svg)](https://grpc.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-purple.svg)](LICENSE)

Este repositorio contiene las definiciones canónicas en **Protocol Buffers v3 (`.proto`)** de la API gRPC de alto rendimiento de **Caerus BaaS**.

Si estás construyendo aplicaciones en **Go, Python, Java, Kotlin, Rust, C#, C++ o Elixir**, podés clonar este repositorio o incluirlo como submódulo Git para compilar tus propios clientes gRPC fuertemente tipados sin necesidad de utilizar el SDK de Node.js/TypeScript.

---

## 📂 Estructura del Repositorio

Siguiendo el estándar universal de Google y la [Guía de Estilo de Buf](https://buf.build/docs/style-guide), los paquetes gRPC reflejan exactamente la estructura de directorios:

```text
caerus-protos/
├── README.md
├── buf.yaml
└── caerus/
    ├── sre/
    │   └── v1/
    │       └── sre_service.proto   # Shared Resource Engine (SRE)
    └── dls/
        └── v1/
            └── dls_service.proto   # Distributed Locking Engine (DLS)
```

---

## ⚡ Servicios y Paquetes

### 1. `caerus.sre.v1.SharedResourceEngine`
* **Archivo:** [`caerus/sre/v1/sre_service.proto`](caerus/sre/v1/sre_service.proto)
* **Propósito:** Gestión transaccional de inventario limitado, reservas temporales (holds con TTL), resolución de conflictos (política `FAIL` inmediata o `QUEUE` con asignación automática FIFO) y confirmaciones permanentes.

| RPC | Entrada | Salida | Descripción |
| :--- | :--- | :--- | :--- |
| `CreateResource` | `CreateResourceRequest` | `ResourceResponse` | Da de alta un recurso Unitario o Pooled. |
| `UpdateResource` | `UpdateResourceRequest` | `ResourceResponse` | Modifica stock disponible o metadata de forma atómica. |
| `DeleteResource` | `DeleteResourceRequest` | `Empty` | Elimina un recurso del motor. |
| **`Take`** | `TakeRequest` | `ResourceHolderResponse` | Adquiere un lease temporal atómico (estado `PENDING` o `QUEUED`). |
| **`Confirm`** | `ConfirmRequest` | `ResourceHolderResponse` | Consolida la compra definitivamente y cancela el TTL (`SOLD`). |
| **`Release`** | `ReleaseRequest` | `Empty` | Libera el lease de inmediato devolviendo el cupo al inventario. |
| `Extend` | `ExtendRequest` | `ResourceHolderResponse` | Extiende el TTL de un lease activo. |
| `GetResource` | `GetResourceRequest` | `ResourceResponse` | Consulta el stock y estado de un recurso. |
| `GetResourcesByGroupKey` | `GetResourcesByGroupKeyRequest` | `GetResourcesByGroupKeyResponse` | Lista recursos paginados por clave de grupo. |
| `GetResourceHolder` | `GetResourceHolderRequest` | `ResourceHolderResponse` | Consulta el estado de un lease por su ID. |
| `GetResourceHoldersList` | `GetResourceHoldersListRequest` | `GetResourceHoldersListResponse` | Historial paginado de holders filtrados por estado. |

---

### 2. `caerus.dls.v1.DistributedLockingEngine`
* **Archivo:** [`caerus/dls/v1/dls_service.proto`](caerus/dls/v1/dls_service.proto)
* **Propósito:** Exclusión mutua distribuida, locks de lectura/escritura (`EXCLUSIVE` y `SHARED_READ`), Fencing Tokens monotónicos contra split-brain, transacciones acotadas y prevención de deadlocks.

| RPC | Entrada | Salida | Descripción |
| :--- | :--- | :--- | :--- |
| `BeginTransaction` | `BeginTransactionRequest` | `BeginTransactionResponse` | Inicia una sesión transaccional con timeout. |
| **`AcquireLock`** | `AcquireLockRequest` | **`stream AcquireLockResponse`** | Streaming de contención: emite `QUEUED` y finaliza en `ACQUIRED` o `DENIED`. Retorna `fencing_token`. |
| `RenewTransaction` | `RenewTransactionRequest` | `RenewTransactionResponse` | Extiende el tiempo de vida de la transacción. |
| **`ReleaseLock`** | `ReleaseLockRequest` | `Empty` | Libera un bloqueo específico. |
| `ReleaseTransactionLocks` | `ReleaseTransactionLocksRequest` | `Empty` | Libera en bloque todos los locks de una transacción. |
| `GetLockStatus` | `GetLockStatusRequest` | `GetLockStatusResponse` | Inspección en tiempo real: quién tiene el lock y tamaño de cola. |
| `GetTransactionStatus` | `GetTransactionStatusRequest` | `GetTransactionStatusResponse` | Consulta el estado y los locks asociados a una transacción. |

---

## 🛠️ Cómo Compilar los Protos en tu Lenguaje

### Opción A: Con Buf CLI (Recomendado)
Si tienes instalado [`buf`](https://buf.build/):
```bash
# Validar sintaxis y reglas de compatibilidad
buf lint

# Generar stubs en base a tu buf.gen.yaml
buf generate
```

---

### Opción B: Con `protoc` Tradicional

#### 🐹 Go
```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

protoc -I. \
  --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  caerus/sre/v1/sre_service.proto \
  caerus/dls/v1/dls_service.proto
```

#### 🐍 Python
```bash
pip install grpcio-tools

python -m grpc_tools.protoc -I. \
  --python_out=. \
  --grpc_python_out=. \
  caerus/sre/v1/sre_service.proto \
  caerus/dls/v1/dls_service.proto
```

#### ☕ Java / Kotlin (Gradle)
Agrega a tu `build.gradle.kts`:
```kotlin
plugins {
    id("com.google.protobuf") version "0.9.4"
}

protobuf {
    protoc { artifact = "com.google.protobuf:protoc:3.25.1" }
    plugins {
        id("grpc") { artifact = "io.grpc:protoc-gen-grpc-java:1.60.0" }
    }
    generateProtoTasks {
        all().forEach { task ->
            task.plugins { id("grpc") }
        }
    }
}
```

#### 🦀 Rust (Tonic / Prost)
En tu `build.rs`:
```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure()
        .compile(
            &[
                "caerus/sre/v1/sre_service.proto",
                "caerus/dls/v1/dls_service.proto",
            ],
            &["."],
        )?;
    Ok(())
}
```

#### 🟦 TypeScript / Node.js Dinámico (Sin compilación previa)
```typescript
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

const packageDefinition = protoLoader.loadSync('caerus/sre/v1/sre_service.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const protoDescriptor = grpc.loadPackageDefinition(packageDefinition) as any;
const SreClient = protoDescriptor.caerus.sre.v1.SharedResourceEngine;

const client = new SreClient(
  'api.caerus.dev:443',
  grpc.credentials.createSsl()
);
```

---

## 🔐 Autenticación y Conexión

### Endpoints
* **Producción Cloud:** `api.caerus.dev:443` (Requiere TLS/SSL)
* **Entorno Local (Docker):** `localhost:9090` (Texto plano / Insecure)

### Headers de Autenticación (Metadata gRPC)
Cada llamada RPC debe incluir tu API Key en los metadatos gRPC:

```text
authorization: Bearer caer_live_xxxxxxxxxxxxxxxxxxxxxxxx
# O alternativamente:
x-api-key: caer_live_xxxxxxxxxxxxxxxxxxxxxxxx
```

---

## 📚 Documentación Oficial

Para consultar la documentación interactiva, diagramas de arquitectura y ejemplos de integración:
* **Especificación gRPC Completa:** [https://caerus.dev/docs/proto](https://caerus.dev/docs/proto)
* **Consola y Dashboard:** [https://caerus.dev/dashboard](https://caerus.dev/dashboard)
* **SDK de TypeScript:** [caerus-dev/caerus-sdk-ts](https://github.com/caerus-dev/caerus-sdk-ts)

---

## 📄 Licencia
Este repositorio se distribuye bajo los términos de la licencia [MIT](LICENSE).