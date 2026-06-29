# Plan de implementación — Anchor SPEI ↔ CETES (Etherfuse)

> Fuente: documentación de Etherfuse (`docs.etherfuse.com`), investigada 2026-06-28.

## Estado de los issues

| Issue | GitHub | Responsable | Notas |
|-------|--------|-------------|-------|
| A-1 · Backend: API client + `/ramp/assets` | [#220](https://github.com/ericmt-98/micopay-protocol/issues/220) cerrado | **Equipo core** | Requiere API key de Etherfuse — trabajo interno |
| A-2 · Backend: KYC routes | [#221](https://github.com/ericmt-98/micopay-protocol/issues/221) cerrado | **Equipo core** | Requiere API key de Etherfuse — trabajo interno |
| A-3 · Backend: CLABE + quote + order + webhook | [#222](https://github.com/ericmt-98/micopay-protocol/issues/222) cerrado | **Equipo core** | Requiere API key de Etherfuse — trabajo interno |
| A-4 · Frontend: pantalla KYC | [#223](https://github.com/ericmt-98/micopay-protocol/issues/223) abierto | **Drips** | Trabaja contra stubs del backend (ver abajo) |
| A-5 · Frontend: onramp SPEI en CETESScreen | [#224](https://github.com/ericmt-98/micopay-protocol/issues/224) abierto | **Drips** | Trabaja contra stubs del backend (ver abajo) |
| A-6 · Frontend: offramp CETES → SPEI | [#225](https://github.com/ericmt-98/micopay-protocol/issues/225) abierto | **Drips** | Trabaja contra stubs del backend (ver abajo) |

### Estrategia de mocks para Drips (A-4/A-5/A-6)

Los contribuidores de frontend no necesitan la API key de Etherfuse. El equipo core agrega rutas stub en el backend que devuelven respuestas con la forma correcta de JSON (sin llamar a Etherfuse). Cuando el equipo conecte la API real, el frontend no cambia nada — solo el backend.

**Stubs necesarios antes de desbloquear A-4:**
- `POST /defi/kyc/submit` → `{ customerId: "mock-kyc-123", status: "pending" }`
- `POST /defi/kyc/documents` → `{ uploaded: true }`
- `GET /defi/kyc/status` → `{ status: "approved" }` (en sandbox siempre aprueba)

**Stubs necesarios antes de desbloquear A-5/A-6:**
- `POST /defi/bank-account` → `{ bankAccountId: "mock-bank-456", clabe: "<clabe_ingresada>" }`
- `POST /defi/ramp/quote` → `{ quoteId: "mock-q-789", exchangeRate: "17.50", destinationAmount: "0.0571", expiresAt: "<now+2min>" }`
- `POST /defi/ramp/order` → `{ orderId: "mock-o-001", depositClabe: "646180157000000004", depositAmount: "1000.00", depositBankName: "Etherfuse MX", depositAccountHolder: "Etherfuse MX", withdrawAnchorAccount: "GABPM7AXX...", withdrawMemo: "bW9ja21lbW8xMjM0NTY3ODkw", withdrawMemoType: "hash" }`
- `GET /defi/ramp/order/:id` → `{ status: "completed" }` (después de 5s simulados)
- `POST /defi/ramp/order/:id/regenerate_tx` → misma forma que el order original

## Resumen de flujos

Etherfuse expone una API REST propia (no SEP-6/SEP-24) para onramp y offramp SPEI ↔ CETES en Stellar.

**Onramp (SPEI → CETES):**
```
Usuario transfiere MXN via SPEI a CLABE de Etherfuse
  → Etherfuse detecta el pago
  → acredita CETES en la wallet Stellar del usuario
```

**Offramp (CETES → SPEI):**
```
Frontend firma un payment Stellar con keypair del dispositivo
  → CETES enviados a cuenta de Etherfuse (anchor mode, con memo)
  → Etherfuse detecta el pago on-chain
  → envía MXN via SPEI a la CLABE del usuario
```

**Restricciones clave:**
- API key B2B requerido — nunca va al frontend, siempre en el backend
- KYC obligatorio para todos los usuarios (CURP + RFC + selfie + liveness check por SPEI/CNBV)
- El issuer de CETES difiere entre sandbox y producción — siempre leerlo de `/ramp/assets`, nunca hardcodearlo
- Sandbox: `api.sand.etherfuse.com` / Producción: `api.etherfuse.com`
- Cotización caduca en **2 minutos** — la transacción Stellar en **1-2 minutos**

## Diagrama de dependencias

```
[EQUIPO CORE — necesitan API key Etherfuse]
A-1 (API client + assets)
├── A-2 (KYC backend)
└── A-3 (quote/order/webhook)
         │
         │ equipo core agrega stubs
         ▼
[DRIPS — trabajan contra stubs, sin API key]
A-4 (KYC frontend)       ← stubs de A-2
A-5 (onramp SPEI UI)     ← stubs de A-3
A-6 (offramp CETES→SPEI) ← stubs de A-3
```

A-4/A-5/A-6 pueden empezar en paralelo en cuanto el equipo core agregue los stubs.
Cuando A-1/A-2/A-3 estén listos con la API real, el frontend no necesita cambios.

---

## A-1 · Backend: Etherfuse API client + `GET /defi/ramp/assets`

**Complejidad:** media | **Depende de:** nada

### Que existe hoy

`apps/api/src/services/etherfuse.service.ts` llama a `/lookup/bonds/cost/CETES` para tasas. No existe cliente para la API de ramp (`/ramp/*`), ni manejo del API key, ni endpoint de assets.

### Que construir

**1. Variable de entorno**

Agregar a `.env.example` y `render.yaml`:
```
ETHERFUSE_API_KEY=<tu-llave-de-etherfuse>
ETHERFUSE_API_URL=https://api.sand.etherfuse.com   # sandbox
# produccion: https://api.etherfuse.com
```

**2. Cliente HTTP autenticado**

En `apps/api/src/services/etherfuse.service.ts`, agregar una funcion `etherfuseRampClient()` que retorne un `fetch` preconfigurado con:
```
Authorization: Bearer <ETHERFUSE_API_KEY>
Content-Type: application/json
```

**3. Ruta `GET /defi/ramp/assets`**

Hace proxy a `GET /ramp/assets` de Etherfuse. Retorna la lista de activos soportados con su identificador Stellar real (`CODE:ISSUER`).

Respuesta esperada (subset):
```json
[
  {
    "identifier": "CETES:GCRYUGD5NVARGXT56XEZI5CIFCQETYHAPQQTHO2O3IQZTHDH4LATMYWC",
    "symbol": "CETES",
    "network": "stellar",
    "type": "bond"
  }
]
```

### Archivos a tocar

| Archivo | Que cambia |
|---------|-----------|
| `apps/api/src/services/etherfuse.service.ts` | Agregar `etherfuseRampClient()` |
| `apps/api/src/routes/cetes.ts` | Agregar `GET /defi/ramp/assets` |
| `.env.example` | Agregar `ETHERFUSE_API_KEY` y `ETHERFUSE_API_URL` |
| `render.yaml` | Agregar `ETHERFUSE_API_KEY` como `sync: false` |

### Criterio de aceptacion

- [ ] `GET /defi/ramp/assets` retorna la lista de activos con su `identifier` Stellar.
- [ ] El API key se lee de `process.env.ETHERFUSE_API_KEY` — nunca hardcodeado.
- [ ] Si `ETHERFUSE_API_KEY` no esta configurado, la ruta retorna 503 con mensaje claro.
- [ ] `tsc --noEmit` pasa sin errores en `apps/api/`.

---

## A-2 · Backend: KYC routes (submit / documents / status)

**Complejidad:** alta | **Depende de:** A-1

### API de Etherfuse a usar

- `POST /customers` — crear registro de cliente con datos personales
- `POST /kyc/submit` — enviar CURP, RFC, fecha de nacimiento, ocupacion, domicilio
- `POST /kyc/documents` — subir selfie, ID oficial, comprobante de domicilio
- `GET /kyc/status` — consultar estado (pending / approved / rejected)

### Que construir

Tres rutas en `apps/api/src/routes/kyc.ts`:

**`POST /defi/kyc/submit`**
```json
{
  "firstName": "Maria",
  "lastName": "Garcia",
  "email": "maria@example.com",
  "phone": "+521234567890",
  "curp": "GARM990101MDFRC001",
  "rfc": "GARM990101AB9",
  "dateOfBirth": "1999-01-01",
  "occupation": "comerciante",
  "address": {
    "street": "Av. Insurgentes 100",
    "city": "CDMX",
    "region": "CDMX",
    "postalCode": "06600"
  }
}
```
Crea el cliente en Etherfuse y envia la solicitud de KYC. Guarda el `customerId` en `users`.

**`POST /defi/kyc/documents`**
Recibe multipart/form-data con `selfie`, `id_document`, `proof_of_address`. Los reenvía a Etherfuse.

**`GET /defi/kyc/status`**
```json
{ "status": "pending" | "approved" | "rejected", "rejectionReason": "..." }
```

### Archivos a tocar

| Archivo | Que cambia |
|---------|-----------|
| `apps/api/src/routes/kyc.ts` | Nuevo archivo con las 3 rutas |
| `apps/api/src/index.ts` | Registrar `kycRoutes` |
| `micopay/sql/migrations/` | Migracion para columna `kyc_customer_id` en `users` |

### Criterio de aceptacion

- [ ] `POST /defi/kyc/submit` crea el cliente en Etherfuse sandbox y guarda el `customerId`.
- [ ] `POST /defi/kyc/documents` sube los archivos y retorna confirmacion.
- [ ] `GET /defi/kyc/status` retorna el estado real desde Etherfuse (no mock).
- [ ] Las rutas requieren token de usuario valido (middleware de auth existente).
- [ ] En sandbox, un KYC con datos ficticios retorna `approved` automaticamente.
- [ ] `tsc --noEmit` pasa sin errores.

---

## A-3 · Backend: CLABE registro + quote + order + webhook SPEI

**Complejidad:** alta | **Depende de:** A-1

### API de Etherfuse a usar

- `POST /ramp/bank-accounts` — registrar CLABE del usuario
- `POST /ramp/quote` — cotizacion MXN → CETES
- `POST /ramp/order` — crear orden, obtener CLABE de deposito
- `GET /ramp/order/:id` — consultar estado
- Webhook entrante — Etherfuse notifica cuando el SPEI llega y los CETES son acreditados

### Que construir

**`POST /defi/bank-account`**
```json
{ "clabe": "646180157000000004" }
```
Registra la CLABE en Etherfuse y la guarda en `users`.

**`POST /defi/ramp/quote`**
```json
{
  "sourceAsset": "MXN",
  "targetAsset": "CETES:<ISSUER>",
  "sourceAmount": "1000",
  "walletAddress": "<stellar_pubkey_del_usuario>"
}
```
El `targetAsset` se obtiene de `/defi/ramp/assets` — nunca hardcodeado.

Respuesta:
```json
{
  "quoteId": "...",
  "exchangeRate": "17.12",
  "destinationAmount": "0.0584",
  "expiresAt": "2026-06-28T10:30:00Z"
}
```

**`POST /defi/ramp/order`**
```json
{
  "orderId": "...",
  "depositClabe": "646180157000000004",
  "depositAmount": "1000.00",
  "depositBankName": "Etherfuse MX",
  "depositAccountHolder": "Etherfuse MX"
}
```

**`GET /defi/ramp/order/:orderId`**
Proxy a Etherfuse. Retorna `{ status: "pending" | "completed" | "failed" }`.

**`POST /defi/ramp/webhook`** (ruta publica, sin auth de usuario)
Etherfuse llama este endpoint cuando el SPEI es recibido. Verificar firma del webhook con secret de Etherfuse.

### Archivos a tocar

| Archivo | Que cambia |
|---------|-----------|
| `apps/api/src/routes/ramp.ts` | Nuevo archivo con las 5 rutas |
| `apps/api/src/index.ts` | Registrar `rampRoutes` |
| `micopay/sql/migrations/` | Columna `bank_account_id` y `clabe` en `users` |

### Criterio de aceptacion

- [ ] `POST /defi/bank-account` guarda la CLABE y retorna el `bankAccountId` de Etherfuse.
- [ ] `POST /defi/ramp/quote` retorna tasa y monto en CETES con `expiresAt`.
- [ ] `POST /defi/ramp/order` retorna la CLABE de deposito de Etherfuse.
- [ ] `GET /defi/ramp/order/:id` retorna el estado actualizado de la orden.
- [ ] `POST /defi/ramp/webhook` verifica firma y responde 200 (sin crash en body invalido).
- [ ] `tsc --noEmit` pasa sin errores.

---

## A-4 · Frontend: pantalla KYC (CURP / RFC / documentos)

**Complejidad:** alta | **Depende de:** A-2

### Que construir

Nueva pantalla `micopay/frontend/src/pages/KYCScreen.tsx` con dos pasos:

**Paso 1 — Datos personales**
- Nombre completo, fecha de nacimiento, telefono, ocupacion, domicilio
- **CURP** (18 caracteres — validacion de formato regex)
- **RFC** (13 caracteres)

**Paso 2 — Documentos**
- Selfie (camara o galeria)
- Identificacion oficial (INE, pasaporte)
- Comprobante de domicilio

En Android: `input type="file" accept="image/*" capture="user"` dentro de WebView de Capacitor.

**Estado post-envio**
Polling a `GET /defi/kyc/status` cada 5 segundos:
- `pending` → "Verificando identidad..."
- `approved` → navegar al flujo SPEI
- `rejected` → motivo + opcion de reintentar

El estado `approved` se cachea en `secureStorage` con clave `kyc_status`.

**Punto de entrada:** si el usuario toca "Depositar via SPEI" en `CETESScreen` sin KYC aprobado, navegar a `KYCScreen`.

### Archivos a tocar

| Archivo | Que cambia |
|---------|-----------|
| `micopay/frontend/src/pages/KYCScreen.tsx` | Nuevo archivo |
| `micopay/frontend/src/services/api.ts` | Agregar `submitKYC()`, `uploadKYCDocuments()`, `getKYCStatus()` |
| `micopay/frontend/src/App.tsx` | Agregar ruta `/kyc` |

### Criterio de aceptacion

- [ ] CURP y RFC validados con regex antes de enviar.
- [ ] Los documentos se suben correctamente via `POST /defi/kyc/documents`.
- [ ] El polling muestra estado en tiempo real.
- [ ] En sandbox, KYC con datos ficticios se aprueba automaticamente.
- [ ] Estado `approved` cacheado en `secureStorage`.
- [ ] `tsc --noEmit` pasa sin errores.

---

## A-5 · Frontend: flujo SPEI en CETESScreen (quote + CLABE + polling)

**Complejidad:** alta | **Depende de:** A-3 + A-4

### Flujo completo

```
[CETESScreen] → tab "SPEI"
  → verificar kyc_status → si no aprobado → KYCScreen
  → input monto MXN
  → POST /defi/ramp/quote → mostrar cotizacion (tasa + CETES + expira en X min)
  → usuario confirma → POST /defi/ramp/order
  → instrucciones de transferencia:
      CLABE: 646180157000000004  [Copiar] [QR]
      Banco: Etherfuse MX
      Monto exacto: $1,000.00 MXN
  → polling GET /defi/ramp/order/:id cada 5s
  → completed → "CETES acreditados en tu wallet"
```

### Que construir

Tercer tab "SPEI" en `CETESScreen.tsx` con 3 subpasos:

**Subpaso 1 — Cotizacion:** input MXN + llamada a `POST /defi/ramp/quote` + display de CETES a recibir con countdown de expiracion.

**Subpaso 2 — Instrucciones SPEI:** CLABE copiable, QR (`qrcode.react` ya instalado), monto exacto, nombre banco/titular.

**Subpaso 3 — Confirmacion:** polling cada 5s. Cuando `completed`, mostrar CETES acreditados + hash Stellar.

Si la cotizacion expira antes de confirmar: aviso + boton para obtener nueva cotizacion.

Primer uso: modal para registrar CLABE del usuario (`POST /defi/bank-account`), cacheada en `secureStorage`.

### Archivos a tocar

| Archivo | Que cambia |
|---------|-----------|
| `micopay/frontend/src/pages/CETESScreen.tsx` | Agregar tab SPEI con los 3 subpasos |
| `micopay/frontend/src/services/api.ts` | Agregar `getRampQuote()`, `createRampOrder()`, `getRampOrderStatus()`, `registerBankAccount()` |

### Criterio de aceptacion

- [ ] Flujo SPEI inaccesible sin KYC aprobado.
- [ ] Cotizacion muestra CETES a recibir y countdown de expiracion.
- [ ] Instrucciones incluyen CLABE copiable y QR.
- [ ] Polling actualiza estado automaticamente.
- [ ] En sandbox, orden transiciona a `completed` en ~30 segundos.
- [ ] `tsc --noEmit` pasa sin errores.

---

---

## A-6 · Frontend: offramp CETES → MXN via SPEI (anchor mode)

**Complejidad:** alta | **Depende de:** A-3 (quote/order backend), A-4 (KYC)

### Contexto

El offramp es la operación inversa al onramp: el usuario vende CETES y recibe pesos en su cuenta SPEI. Etherfuse soporta dos modos en Stellar:

- **Modo default:** Etherfuse construye la transacción de quema y la devuelve firmada parcialmente.
- **Anchor mode** (`useAnchor: true`): Etherfuse devuelve cuenta destino + memo; el frontend construye y firma el payment con el keypair del dispositivo.

MicoPay usa **anchor mode** porque es no-custodial — el usuario firma con su propia llave.

### Flujo completo

```
[CETESScreen] → tab "Vender" → subtab "Recibir SPEI"
  → verificar kyc_status y CLABE registrada
  → input monto en CETES a vender
  → POST /defi/ramp/quote (type: offramp, sourceAsset: CETES, targetAsset: MXN)
  → mostrar MXN a recibir + tasa + countdown 2 minutos
  → usuario confirma
  → POST /defi/ramp/order (useAnchor: true)
  → recibe { withdrawAnchorAccount, withdrawMemo, withdrawMemoType: "hash" }
  → frontend firma y envía transaccion Stellar:
      payment:
        destination: withdrawAnchorAccount
        asset: CETES:<ISSUER>
        amount: <monto del quote>
        memo: Memo.hash(Buffer.from(withdrawMemo, 'base64'))
  → polling GET /defi/ramp/order/:id cada 5s
  → funded  → "CETES recibidos por Etherfuse, procesando SPEI..."
  → completed → "MXN enviados a tu cuenta SPEI"
  → finalized → operacion cerrada
```

### Timing — restriccion critica

| Etapa | Ventana | Que hacer si expira |
|-------|---------|---------------------|
| Cotizacion | 2 minutos | Nueva llamada a `/ramp/quote` |
| Transaccion Stellar | ~1-2 minutos | `POST /ramp/order/:id/regenerate_tx` → re-firmar |

La UI debe guiar al usuario para confirmar y firmar dentro de los 90 segundos de obtener la orden. Si la transaccion Stellar llega tarde (`tx_too_late`), llamar a `regenerate_tx` y volver a firmar.

### Que construir

**En `stellarRamp.ts`** (nuevo servicio, paralelo a `stellarDex.ts`):

```typescript
export async function sendCETESToEtherfuse(
  cetesAmount: string,
  withdrawAnchorAccount: string,
  withdrawMemo: string,          // base64
): Promise<{ hash: string }> {
  // 1. exportSecretKey() → keypair del dispositivo
  // 2. loadAccount() desde Horizon
  // 3. TransactionBuilder con:
  //      Operation.payment({
  //        destination: withdrawAnchorAccount,
  //        asset: new Asset('CETES', CETES_ISSUER),
  //        amount: cetesAmount,
  //      })
  //      memo: TransactionBuilder.buildIncrement...
  //      Memo.hash(Buffer.from(withdrawMemo, 'base64'))
  // 4. setTimeout(90)  ← ventana mas generosa que el DEX
  // 5. sign(kp) + submitTransaction()
}
```

**En `CETESScreen.tsx`**, dentro del tab "Vender", agregar opcion "Recibir en SPEI" vs "Recibir en wallet":
- "Recibir en wallet" → flujo DEX ya implementado (`sellCETESOnDex`)
- "Recibir en SPEI" → flujo offramp (este issue)

**Nuevas funciones en `api.ts`:**
- `getOfframpQuote(cetesAmount)` → `POST /defi/ramp/quote` con `type: offramp`
- `createOfframpOrder(quoteId)` → `POST /defi/ramp/order` con `useAnchor: true`
- `regenerateOfframpTx(orderId)` → `POST /defi/ramp/order/:id/regenerate_tx`

### Archivos a tocar

| Archivo | Que cambia |
|---------|-----------|
| `micopay/frontend/src/services/stellarRamp.ts` | Nuevo: `sendCETESToEtherfuse()` |
| `micopay/frontend/src/services/api.ts` | Agregar `getOfframpQuote()`, `createOfframpOrder()`, `regenerateOfframpTx()` |
| `micopay/frontend/src/pages/CETESScreen.tsx` | Tab "Vender" con opcion SPEI vs wallet |

### Criterio de aceptacion

- [ ] La opcion "Recibir en SPEI" solo aparece si el usuario tiene KYC aprobado y CLABE registrada.
- [ ] El countdown de 2 minutos es visible durante la confirmacion del quote.
- [ ] Si el quote expira antes de confirmar, la UI invita a obtener uno nuevo sin perder el monto ingresado.
- [ ] La transaccion Stellar incluye el memo correcto (hash, decodificado de base64) — sin memo = refund de Etherfuse.
- [ ] Si la transaccion expira (`tx_too_late`), la UI llama a `regenerate_tx` y re-firma automaticamente.
- [ ] El polling muestra los estados `funded` → `completed` con mensajes en espanol.
- [ ] En sandbox, la orden transiciona a `completed` en ~30 segundos tras el envio on-chain.
- [ ] `tsc --noEmit` pasa sin errores.

---

## Plan de ejecución

### Equipo core (con API key de Etherfuse)
1. Registrar cuenta business en `sandbox.etherfuse.com` → obtener API key
2. Implementar A-1 (cliente + `/ramp/assets`) — base de todo
3. Implementar A-2 y A-3 en paralelo
4. **Mientras tanto:** agregar stubs para desbloquear A-4/A-5/A-6 a Drips

### Drips (sin API key, trabajando con stubs)
- A-4, A-5, A-6 pueden empezar en cuanto los stubs estén en el backend
- Cuando el equipo conecte la API real (A-2/A-3), el frontend no necesita cambios
- Para desbloquear un issue: quitar el label `wave:blocked` y avisar en el issue

### Orden sugerido
```
Semana 1: equipo core → A-1 + stubs para A-4/A-5/A-6
Semana 1-2: Drips → A-4/A-5/A-6 en paralelo contra stubs
Semana 2-3: equipo core → A-2 + A-3 (reemplaza stubs con API real)
```
