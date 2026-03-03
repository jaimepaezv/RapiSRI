# Documento de Requerimientos del Sistema (DRS)
## Proyecto: RapiSRI Desktop (Windows 10+)

## 1. Propósito
Definir los requerimientos funcionales y no funcionales para un software de escritorio **offline-first** que cubra:
- Emisión de **Notas de Venta preimpresas** para **RIMPE Negocio Popular**.
- Emisión de **Comprobantes Electrónicos** (factura, retención y complementarios) bajo esquema **SRI Off-line v2.32**.
- Control de secuenciales, archivo, respaldo, reimpresión y auditoría.

## 2. Alcance
### 2.1 Incluye
- Aplicación desktop Windows 10+.
- Base local (SQLite) con backup/exportación.
- Módulos: configuración inicial, clientes, productos/servicios (opcional), emisión de nota de venta, emisión electrónica, documentos, respaldos, configuración.
- Firma digital con certificado `.p12`.

### 2.2 Excluye
- Contabilidad integral, nómina, CRM, inventario avanzado, BI.
- Integraciones bancarias/pasarelas.
- Multi-sucursal concurrente en red.

## 3. Reglas de negocio normativas
1. Si régimen = `RIMPE_EMPRENDEDOR`, solo permite comprobantes electrónicos.
2. Si régimen = `RIMPE_NEGOCIO_POPULAR`, permite nota preimpresa y opcional electrónico.
3. Todo comprobante debe incluir leyenda RIMPE según régimen.
4. Para electrónicos, aplicar esquema Off-line: XML, firma, recepción, autorización y clave de acceso 49 dígitos.

## 4. Roles
### 4.1 Administrador
- Configura empresa, establecimiento/punto, ambiente, certificado.
- Gestiona secuenciales/rangos.
- Anula documentos según política.
- Ejecuta respaldo/exportación.

### 4.2 Operador
- Emite documentos.
- Gestiona clientes.
- Reenvía documentos electrónicos en error.
- No modifica configuración crítica.

## 5. Historias de usuario + criterios de aceptación

### HU-01 Setup inicial obligatorio
**Como** Administrador, **quiero** completar un asistente inicial, **para** habilitar emisión.

**Campos obligatorios**: RUC, razón social, régimen, establecimiento, punto emisión, ambiente, certificado `.p12`, clave cifrada y rutas de salida.

**Aceptación**
- No se puede ingresar al módulo Emitir sin setup completo.
- Si régimen emprendedor, nota de venta queda bloqueada.
- Si régimen negocio popular, nota de venta habilitada.

### HU-02 Gestión de clientes
**Como** Operador, **quiero** registrar clientes, **para** usarlos en emisión.

**Aceptación**
- Valida formato de RUC (13) y cédula (10).
- Soporta consumidor final configurable (por defecto `9999999999999`).

### HU-03 Emisión de Nota de Venta preimpresa
**Como** Operador, **quiero** emitir nota preimpresa, **para** registrar ventas de negocio popular.

**Aceptación**
- Toma siguiente secuencial disponible de rango activo.
- No repite secuenciales por estab/punto/autorización.
- Imprime campos variables alineables por plantilla de impresora.
- Incluye leyenda “Contribuyente RIMPE Negocio Popular”.

### HU-04 Administración de rangos preimpresos
**Como** Administrador, **quiero** definir rangos, **para** controlar numeración autorizada.

**Aceptación**
- Estados: ACTIVO, AGOTADO, DADO_DE_BAJA.
- Si no hay rango activo, bloquea emisión.
- Al llegar a “hasta”, cambia a AGOTADO.

### HU-05 Emisión de factura electrónica
**Como** Operador, **quiero** emitir factura electrónica completa, **para** cumplir obligación tributaria.

**Aceptación**
- Flujo: BORRADOR → FIRMADO → ENVIADO_RECEPCION → RECIBIDO → AUTORIZADO/NO_AUTORIZADO/ERROR.
- Genera clave de acceso de 49 dígitos válida módulo 11.
- Genera XML firmado, consulta autorización, genera RIDE PDF y exportables.
- Tiempo objetivo: emisión completa en ≤ 1 minuto con datos precargados.

### HU-06 Emisión de retención electrónica
**Como** Operador, **quiero** emitir retención asociada a compra, **para** soportar obligaciones de retener.

**Aceptación**
- Permite factura base interna o externa.
- Secuencial independiente por tipo documental.
- Genera XML firmado, autorización y RIDE.

### HU-07 Emisión de NC/ND electrónica
**Como** Operador, **quiero** emitir NC/ND sobre documento base, **para** corregir comprobantes autorizados.

**Aceptación**
- Requiere referencia a factura base.
- No edita documento autorizado; solo corrige.
- NC no puede exceder valor del documento base.

### HU-08 Motor único de secuenciales
**Como** Sistema, **quiero** controlar secuenciales por tipo/serie/ambiente, **para** evitar cruces y duplicados.

**Aceptación**
- Consume secuencial al emitir (no en borrador).
- Si falla envío/firma, conserva secuencial con estado ERROR/NO_AUTORIZADO.

### HU-09 Gestión documental
**Como** Usuario, **quiero** buscar y operar documentos, **para** reimprimir/exportar/reintentar.

**Aceptación**
- Filtros por fecha, tipo, estado, cliente, secuencial y clave.
- Acciones: ver detalle, reimprimir, exportar, reintentar, ver motivos SRI.
- Búsqueda por secuencial/clave en ≤ 5 segundos.

### HU-10 Respaldo y archivado
**Como** Administrador, **quiero** respaldar base+archivos, **para** proteger información fiscal.

**Aceptación**
- Estructura de carpetas por `XML/RIDE/LOGS` y año/mes.
- Backup ZIP manual y opcional al cierre.
- Sin borrado automático; retención sugerida ≥ 7 años.

## 6. Modelo de estados (electrónicos)
- `BORRADOR`
- `FIRMADO`
- `ENVIADO_RECEPCION`
- `RECIBIDO`
- `AUTORIZADO`
- `NO_AUTORIZADO`
- `ERROR`

## 7. Interfaces internas (servicios)
1. Motor de secuenciales.
2. Motor XML por tipo documental.
3. Motor de firma digital (`.p12`).
4. Conector SRI (recepción y autorización).
5. Motor RIDE (PDF).
6. Servicio de auditoría/log por documento.

## 8. Contratos funcionales (endpoints internos sugeridos)
> Nota: Son endpoints locales de aplicación (API interna), no WS públicos SRI.

### 8.1 Setup/configuración
- `POST /api/setup/initialize`
- `GET /api/setup/status`
- `PUT /api/config/company`
- `PUT /api/config/certificate`

### 8.2 Catálogos
- `POST /api/customers`
- `GET /api/customers`
- `PUT /api/customers/{id}`
- `POST /api/items` (opcional catálogo)

### 8.3 Rangos y secuenciales
- `POST /api/preprint-ranges`
- `GET /api/preprint-ranges`
- `POST /api/sequences/next`

### 8.4 Emisión documental
- `POST /api/sales-notes/draft`
- `POST /api/sales-notes/issue`
- `POST /api/electronic-invoices/draft`
- `POST /api/electronic-invoices/issue`
- `POST /api/withholdings/issue`
- `POST /api/credit-notes/issue`
- `POST /api/debit-notes/issue`

### 8.5 Ciclo electrónico
- `POST /api/electronic/{id}/sign`
- `POST /api/electronic/{id}/send-reception`
- `POST /api/electronic/{id}/check-authorization`
- `POST /api/electronic/{id}/retry`

### 8.6 Documentos y archivos
- `GET /api/documents`
- `GET /api/documents/{id}`
- `GET /api/documents/{id}/exports`
- `GET /api/documents/{id}/ride`
- `POST /api/backups/generate`

## 9. Validaciones transversales
- Totales con precisión de 2 decimales.
- No emitir sin cliente, ítems ni configuración crítica.
- Documento AUTORIZADO es inmutable.
- Leyenda RIMPE obligatoria en impresión/RIDE.

## 10. Requisitos no funcionales
- Instalador único para Windows (MSI o equivalente).
- Emisión local nota preimpresa < 3 segundos.
- Generación + firma electrónica < 5 segundos (sin latencia SRI).
- Contraseñas cifradas (certificado y usuarios).
- Trazabilidad de acciones por usuario y fecha/hora.

## 11. Priorización MVP (6 semanas)

### Semana 1–2 (MVP núcleo local)
- Setup Wizard.
- Clientes.
- Rangos preimpresos.
- Nota de Venta + impresión/calibración.
- Motor de secuenciales base.

### Semana 3–4 (MVP electrónico factura)
- Factura electrónica completa (XML, firma, recepción, autorización).
- RIDE PDF y exportables.
- Listado de documentos + filtros + reintentos.

### Semana 5 (Retención)
- Retención electrónica con factura base interna/externa.
- Estados, exportación y RIDE.

### Semana 6 (NC/ND + cierre)
- NC/ND electrónica.
- Respaldo ZIP.
- Reporte simple de ventas día/mes (CSV/PDF).
- Hardening, QA funcional y checklist de salida.

## 12. Definition of Done (DoD)
1. Factura electrónica en pruebas llega a `AUTORIZADO`.
2. Retención electrónica en pruebas llega a `AUTORIZADO`.
3. Nota de venta usa rango, secuencial único e impresión alineable.
4. Documentos persisten XML/PDF en estructura año/mes.
5. Reporte de ventas día/mes exportable.
