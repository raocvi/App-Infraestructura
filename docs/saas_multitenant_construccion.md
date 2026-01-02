# SaaS Multitenant para Construcción de Vías e Infraestructura

## Supuestos
- Público objetivo: constructoras de vías e infraestructura civil con frentes de obra simultáneos y contratos con interventoría externa.
- Multi-tenant puro (dato separado por tenant + row-level security). Moneda base configurable por tenant; soporta multimoneda en compras/OC con tipo de cambio almacenado.
- Mobile y desktop con un solo código (Flutter) + web admin React/Next. Offline-first en mobile (incluye compras y registros operativos).
- Integración futura con ERP externo vía API/webhooks; por ahora contabilidad básica (P&L) interna.
- Seguridad corporativa: MFA opcional, IP allowlist opcional, logs de auditoría obligatorios.
- Feature flags por plan: offline, mantenimiento avanzado, analítica avanzada, workflows avanzados.

## A) Documento funcional (PRD)
### Problema
Las constructoras necesitan controlar avances, costos y compras en múltiples frentes/obras, con aprobaciones formales y trazabilidad para interventoría, manteniendo operaciones offline en campo.

### Objetivos
1) Registrar y aprobar avances, insumos, maquinaria y horas diarias con evidencia.  
2) Automatizar compras e inventario end-to-end con notificaciones y SLAs.  
3) Entregar P&L por obra y por equipo en tiempo casi real.  
4) Operar offline en campo y sincronizar de forma segura.

### Usuarios y roles (ejemplos)
- Operador/Conductor: registra uso de equipo y combustible.
- Ingeniero de frente: registra avances, consumos, horas; crea solicitudes.
- Supervisor/Director de obra: aprueba avances y requisiciones.
- Compras: gestiona cotizaciones y OC.
- Almacén: recepciona, transfiere y entrega a frente.
- Interventoría: valida avances y cortes de facturación.
- Gerencia/Finanzas: ve dashboards, P&L y alertas.

### Flujos clave (resumen)
- Avances: Ejecutor registra -> Supervisor revisa -> Interventoría aprueba -> alimenta facturación y P&L.
- Compras: Solicitud -> Aprobaciones -> Compras (RFQ opcional) -> OC -> Recepción -> Inventario -> Entrega a frente -> Consumo -> P&L.
- Mantenimientos: Plan preventivo -> OT -> ejecución -> costos al equipo/obra -> KPIs de disponibilidad.
- Offline mobile: cola local con reintentos, reconciliación y resolución de conflictos (last-writer con merge guiado para cantidades/evidencias).

### Requerimientos por módulo
- **Configuración (tenant/obra)**: RBAC/ABAC, catálogos maestros, costos, notificaciones, workflows, feature flags, políticas de seguridad.
- **Cronograma/Contrato**: carga WBS, cantidades y valores, hitos, cortes configurables; comparativo plan vs real.
- **Avances por frente**: actividades, cantidades, evidencias, geolocalización opcional, control de cambios.
- **Insumos/Inventario operativo**: catálogo, entradas, consumos, kardex por obra/frente, validaciones de stock.
- **Maquinaria/Equipos**: catálogo, horas/km, combustible, consumibles, disponibilidad, fresadora con puntas por m³.
- **Mantenimientos**: plan preventivo, OTs, repuestos/mano de obra, costos y alertas.
- **Horas diarias**: registro por persona/rol/actividad; aprobaciones opcionales; costeo configurable.
- **Facturación por cortes + interventoría**: estados, observaciones, evidencias; integra con contrato y P&L.
- **Reportes/Analítica**: dashboards obra/equipos, curva S, costos vs presupuesto, margen, alertas; export PDF/Excel.
- **Compras + Inventario corporativo (obligatorio)**: ver sección 10 detallada.

### Detalle módulo COMPRAS/INVENTARIO (10.x)
- **Catálogo precargado**: SKU, categoría, unidad, centros de costo permitidos, restricciones, proveedor y lista de precios opcional.
- **Solicitudes (SR/Requisición)**: obra/frente, centro de costo, ítems, cantidades, fechas, motivo, actividad; adjuntos y trazabilidad; reglas por presupuesto/umbral.
- **Aprobaciones configurables**: por monto, tipo de ítem, centro de costo, obra/frente, criticidad; en cascada o paralelo; SLAs y escalamiento.
- **Compras/RFQ**: bandeja de solicitudes aprobadas; cotizaciones a N proveedores; comparación; selección proveedor; generación OC con impuestos, incoterms opcional; versionado de OC.
- **Recepción e inventario**: contra OC, parcial/total, calidad opcional, lote/serie/vencimiento; remisión/factura; ubicación y kardex; transferencias; entrega a frente enlaza con consumo operativo.
- **Notificaciones**: in-app + push (email opcional); eventos clave para solicitantes, aprobadores, compras, almacén, jefes y gerencia.
- **Reportes**: trazabilidad, compras por CC/obra/proveedor, lead time, desviaciones, valorización de inventario (FIFO/Promedio), rotación, faltantes.

### Flujos (ASCII)
```
SR (Borrador) -> Enviada -> En aprobación -> Aprobada
     | Rechazada
     v
En compras -> (RFQ opcional) -> OC emitida -> Confirmada
     -> Parcialmente recibida -> Recibida -> Cerrada/Cancelada
                          |
                     Entregas a frente -> Consumo operativo -> P&L
```

## B) Modelo de datos (ERD simplificado + descripción)
```
Tenant --< Obra --< Frente
Obra --< CronogramaActividad --< Avance
AvanceEvidence (foto/nota/geo) -> Avance

Usuario --< RolUsuario --< RolPermiso -> Permiso

CatalogoItem (SKU, categoria, unidad, cc_permitidos)
Proveedor --< ListaPrecioProveedor --< ListaPrecioItem

SolicitudRequisicion (SR)
SR --< SR_Item -- CatalogoItem
SR --< SR_Aprobacion (estado, aprobador, SLA, escalamiento)

CotizacionRFQ --< RFQ_OfertaProveedor --< RFQ_OfertaItem
SR -> CotizacionRFQ (opcional)

OrdenCompra (OC) --< OC_Item -- CatalogoItem
OC -> Proveedor
OC -> SR (origen)

Recepcion --< Recepcion_Item -- OC_Item
Almacen --< Ubicacion
MovimientoInventario (entrada/salida/transferencia) -> Recepcion_Item?/OC_Item?/EntregaFrente
Transferencia --< Transferencia_Item
EntregaFrente --< Entrega_Item -> MovimientoInventario

Equipo --< UsoEquipo --< ConsumoEquipo (combustible/consumibles)
Equipo --< MantenimientoPlan
OrdenTrabajoManto --< OT_Repuesto -- CatalogoItem
OT -> Equipo

HoraDiaria -> Usuario/Persona, Frente, Actividad

Notificacion, PlantillaNotificacion, ReglaWorkflow, FeatureFlag

FacturacionCorte --< Facturacion_Item (derivado de Avance aprobado)
P&L -> Obra, Equipo (ingresos - costos directos/indirectos)
```

### Entidades clave
- **SolicitudRequisicion (SR)**: id, tenant, obra, frente, cc, estado, creador, fecha requerida, motivo, actividad, adjuntos, presupuesto_ref.
- **SR_Item**: SR id, SKU, cantidad, unidad, precio_estimado opcional, criticidad, requiere_cotizacion.
- **SR_Aprobacion**: SR id, aprobador, nivel, estado, monto, sla_horas, fecha_limite, escalado_a, comentarios.
- **Proveedor**: datos fiscales, contacto, condiciones pago, listas de precio, incoterms opcional.
- **CotizacionRFQ / RFQ_OfertaProveedor / RFQ_OfertaItem**: solicitudes de cotización y ofertas.
- **OrdenCompra (OC)**: numeración, proveedor, moneda, impuestos, incoterms opcional, fechas, estado, version, total, cc/obra/frente, origen SR.
- **OC_Item**: OC id, SKU, cantidad, unidad, precio, impuestos, entrega_parcial_permitida.
- **Recepcion / Recepcion_Item**: contra OC, cantidad recibida, lote/serie, vencimiento, calidad, adjuntos.
- **Almacen / Ubicacion**: por tenant y obra; tipos (central, obra, frente).
- **MovimientoInventario**: tipo (entrada/recepcion, salida/entrega, transferencia), cantidades, ubicación origen/destino, costo unitario (FIFO/Promedio), referencia (OC, Recepción, EntregaFrente).
- **Transferencia / EntregaFrente**: mueven stock entre almacenes y hacia frentes.
- **Kardex**: vista materializada/consulta de movimientos y costos.
- **Notificaciones / Plantillas / ReglaWorkflow**: canales, variables, SLAs, escalamiento.

## C) Arquitectura propuesta
- **Frontend**
  - Mobile + Desktop: **Flutter** (iOS/Android + Windows/Mac/Linux). Ventajas: un código, soporte offline con `drift`/`floor` y isolates; UI nativa y rendimiento alto.
  - Web admin/analytics: **React/Next.js** (SSR opcional para SEO del portal admin, client-side para panel).
- **Backend**
  - **API**: NestJS (TypeScript) con módulos por dominio; GraphQL o REST; validación (class-validator), RLS/tenant guard.
  - **DB**: PostgreSQL (RLS + esquemas por tenant opcional).  
  - **Almacenamiento**: S3 compatible para evidencias y adjuntos.
  - **Colas**: RabbitMQ/Kafka para notificaciones, SLAs y sincronización offline.
  - **Tiempo real**: WebSockets (NestJS Gateway) o SSE para aprobaciones/notificaciones y dashboards.
  - **Workflow Engine**: motor ligero (temporal-lite/camunda-lite) o implementación con colas + reglas (decision tables) para aprobaciones.
  - **Search**: opcional Elastic/OpenSearch para evidencia y texto.
  - **Cache**: Redis para sesiones, rate-limit y vistas agregadas.
- **Offline sync**
  - Cache local (SQLite) + cola de eventos (commands).  
  - Estrategia: `pending_ops` -> reintentos exponenciales -> reconciliación.  
  - Conflictos: merge específico por entidad (cantidades: sumar diferencias si disponible; estados: última versión con auditoría; adjuntos: multiversión).  
  - Compresión + cifrado de payload; firma HMAC y time drift guard.
- **Seguridad**
  - Auth: email+password + SSO (OIDC), MFA opcional.  
  - RBAC/ABAC: roles por tenant/obra/frente; atributos de centro de costo y montos.  
  - IP allowlist opcional, políticas de contraseña, rotación de tokens, logs de auditoría (PostgreSQL + Loki).
- **Observabilidad**
  - Logs estructurados (JSON), métricas Prometheus, trazas OpenTelemetry.  
  - Dashboards Grafana + alertas.
- **Escalabilidad**
  - Multi-tenant: `tenant_id` en todas las tablas + RLS; particiones por tiempo para movimientos y evidencias.  
  - Servicios: auth, configuraciones, operaciones (avances), compras/inventario, mantenimiento, reporting, notificaciones.

## D) Diseño UX/UI
- **Mapa de pantallas Mobile/Tablet (offline-first)**
  - Login + selección de tenant/obra/frente.
  - Dashboard campo (resumen avance, tareas pendientes, stock en frente, alertas).
  - Registro de avance (actividad, cantidades, evidencias, geo).
  - Consumo de insumos (buscar SKU, escaneo QR/Code128 opcional, cantidades).
  - Uso de maquinaria (horas/km, combustible, puntas fresadora).
  - Solicitud de requisición (SR) y seguimiento de estado.
  - Bandeja de aprobaciones con SLAs y comentarios.
  - Recepción rápida en obra (parcial/total) con fotos y lotes.
  - Sincronización: estado de cola y conflictos.
- **Panel Web Admin**
  - Configuración (roles, permisos, catálogos, workflows, feature flags, notificaciones).
  - Cronograma/WBS y cortes de avance.
  - Bandeja de compras (RFQ, OC, comparador de ofertas).
  - Inventario corporativo, transferencias, kardex, calidad.
  - Mantenimientos y órdenes de trabajo.
  - Dashboards:
    - Obra: % avance real vs plan, curva S, ejecutado vs contratado, costos vs presupuesto, margen, alertas.
    - Equipos: utilización, costo/hora, combustible, disponibilidad, mantenimiento.
    - Compras/Inventario: lead time, desviaciones, rotación, faltantes, costos.
    - P&L obra y equipo (ingresos-costos, directos/indirectos).

## E) Reglas de negocio y fórmulas
- **Avance vs cronograma**: `% avance real = (Σ cantidades aprobadas actividad / cantidad planificada) * 100`. Curva S comparando acumulado plan vs real.
- **Costo real por actividad**: `costo_directo = Σ(consumos valorizados FIFO/Promedio + horas * tarifa + equipos * tarifa_hora + servicios externos)`.
- **Corte y valor a facturar**: por corte, sumar cantidades aprobadas * valor contratado por actividad/tarea; IVA/impuestos según contrato.
- **P&L obra**: `ingresos_facturados - costos_directos - costos_indirectos_prorrateados`.
- **P&L equipo**: `ingresos_por_uso (si se cobra) - (combustible + consumibles + mantenimiento + depreciación opcional + operador)`; separable por equipo y agregado.
- **Inventario**: valoración configurable (FIFO/Promedio). Kardex conserva costo unitario por movimiento.
- **Lead time compras**:  
  - `SR->Aprobación`, `Aprobación->OC`, `OC->Recepción`, `Recepción->Entrega frente`.  
  - SLAs disparan notificaciones/escalamientos.

## F) API (REST/JSON ejemplo)
- `POST /auth/login`, `POST /auth/mfa/verify`, `GET /me`.
- `GET/POST /tenants`, `GET/POST /obras`, `GET/POST /frentes`.
- `GET/POST /catalogo/items`, `GET /catalogo/items/:id/pricing`.
- `POST /sr`, `GET /sr`, `PATCH /sr/:id` (estado, comentarios), `POST /sr/:id/submit`.
- `POST /sr/:id/aprobaciones`, `PATCH /sr/:id/aprobaciones/:aprobId` (aprobar/rechazar).
- `POST /rfq`, `POST /rfq/:id/ofertas`, `GET /rfq/:id/comparacion`.
- `POST /oc`, `GET /oc`, `PATCH /oc/:id` (versionado), `POST /oc/:id/enviar`.
- `POST /recepciones`, `POST /recepciones/:id/items`, `POST /transferencias`, `POST /entregas-frente`.
- `POST /inventario/movimientos` (ajustes con auditoría).
- `POST /avances`, `POST /avances/:id/aprobar`, `POST /consumos`.
- `POST /equipos/uso`, `POST /mantenimientos/ots`.
- `GET /dashboard/obra/:id`, `GET /dashboard/equipos`, `GET /reportes/compras`.
- **Webhooks**: `POST /hooks/sr`, `/hooks/oc`, `/hooks/recepcion`, `/hooks/aprobacion`.

## G) Plan de pruebas
- **Unitarias**: reglas de costos, conversiones de unidades, validación de stock, cálculo FIFO/Promedio, SLAs de aprobación.
- **Integración**: flujo SR->Aprobación->OC->Recepción->Entrega->Consumo; sincronización offline/online; workflows con escalamiento.
- **E2E (mobile+web)**:
  - Crear SR con adjuntos offline, sincronizar y mantener trazabilidad.
  - Aprobación en cascada con SLA vencido y escalado.
  - RFQ a dos proveedores y selección automática por mejor TCO.
  - OC parcial con versión nueva y control de cambios.
  - Recepción parcial con lote/serie y calidad; entrega a frente y consumo que descuente stock.
  - Corte de avance alimenta facturación y actualiza P&L.
- **Performance**: carga de dashboards, particionado de movimientos, stress de notificaciones y colas.
- **Seguridad**: RLS por tenant, ABAC por obra/centro de costo, MFA, auditoría de cambios, rate limiting.
- **UX**: usabilidad offline, resolución de conflictos, estados claros y timelines.

## H) Plan DevOps
- **Repos y ramas**: trunk-based + feature flags. Convenciones de migraciones versionadas.
- **CI/CD**: lint + tests + build (Flutter, Next, Nest) + seguridad (SAST, dep audit). Infra como código (Terraform). Despliegue en contenedores (K8s).
- **Entornos**: dev (shared), staging (datos sintéticos), prod (tenant aislado). Blue/green para backend/web; rollout gradual en mobile con feature flags.
- **Infra**: K8s + Postgres HA + S3 + Redis + RabbitMQ/Kafka + Grafana stack + Loki. WAF y CDN opcional para web.
- **Backups**: snapshots Postgres + objetos S3; pruebas de restauración. Políticas de retención y cifrado at-rest/in-transit.
- **Migraciones**: controladas, con feature toggles y scripts de compatibilidad. Índices y particiones para movimientos/evidencias.
- **Observabilidad**: traces (OTel), métricas (Prometheus), logs (Loki), alertas (PagerDuty/Email).

## I) MVP y roadmap
- **MVP (8-10 semanas)**
  - Tenants/obras/frentes, RBAC básico, catálogos, cronograma WBS, registro de avances con evidencias, consumos, maquinaria básica, horas diarias, SR + aprobaciones + OC básica, recepción y entregas a frente, notificaciones in-app/push, dashboards iniciales, P&L básico (obra/equipo), offline para avances/consumos/SR.
- **v1**
  - RFQ comparador completo, mantenimiento preventivo + OT, versionado de OC, calidad en recepción, feature flags, curva S, lead time compras, export PDF/Excel, ABAC por montos/CC, MFA, observabilidad completa, backup automatizado.
- **v2**
  - Analítica avanzada (predicciones de consumo/overrun), optimización de flota, integraciones ERP, pricing dinámico por proveedor, workflows visuales, buscador full-text, reportes custom, soporte WhatsApp/SMS, depreciación contable y costos indirectos avanzados.

## Ejemplos concretos
- **Solicitud de Requisición (SR)**: Obra “Autopista Norte”, Frente “KM12”, CC “Obra”, ítem: “AC-001 Asfalto caliente”, cantidad 120 t, fecha requerida 2024-10-05, motivo “Recarpeteo tramo 2”, actividad WBS “AC-1.2 Pavimentación”.
- **Aprobación**: Nivel 1 Ingeniero de obra (límite 10k), Nivel 2 Director (límite 50k); SLA 24h cada uno; escalamiento a Gerente si vence.
- **Orden de Compra (OC)**: OC-2024-045, proveedor “Asfaltos Andinos”, moneda USD, incoterm FOB opcional, 120 t a 95 USD/t, IVA 12%, entrega en obra KM12 el 2024-10-07.
- **Recepción parcial**: 60 t recibidas con lote L-778, calidad OK, factura adjunta; estado OC “Parcialmente recibida”; kardex registra entrada con costo 95 USD/t.
- **Entrega a frente**: 40 t al Frente KM12; movimiento de salida; saldo 20 t en obra; consumo de 35 t en actividad AC-1.2.
- **Impacto en P&L**: costo directo actividad = 35 t * 95 USD/t; P&L obra reduce margen; P&L equipo si se imputan costos de uso de fresadora asociada.

## Diagramas ASCII
### Flujo SR->OC->Recepción->Entrega->Consumo
```
[SR Borrador]
     |
     v
[SR Enviada] -> [Aprobaciones SLA/escalamiento] -> [Aprobada]
     |
     v
[Compras/RFQ] -> [OC emitida] -> [OC confirmada]
                              |             |
                         [Recepción parcial]|
                              |             v
                              v        [OC recibida]
                        [Entrada almacén]
                              |
                        [Transferencia]
                              |
                        [Entrega a frente]
                              |
                        [Consumo operativo]
                              |
                        [P&L / Reportes]
```

### ERD simplificado (principal)
```
Tenant--<Obra--<Frente
Obra--<CronogramaActividad--<Avance--<AvanceEvidence

Usuario--<RolUsuario--<RolPermiso->Permiso

CatalogoItem--<SR_Item>--SolicitudRequisicion--<SR_Aprobacion
SolicitudRequisicion--(opcional)->CotizacionRFQ--<RFQ_OfertaProveedor--<RFQ_OfertaItem
SolicitudRequisicion--<OrdenCompra--<OC_Item
OrdenCompra--<Recepcion--<Recepcion_Item
Recepcion_Item--<MovimientoInventario>--Ubicacion--<Almacen
MovimientoInventario--<Transferencia/EntregaFrente

Equipo--<UsoEquipo--<ConsumoEquipo
Equipo--<OrdenTrabajoManto--<OT_Repuesto (CatalogoItem)

FacturacionCorte--<Facturacion_Item
P&L (vista) -> Obra, Equipo
Notificacion/Plantilla/ReglaWorkflow
```

---
Documento en español, listo para revisar con stakeholders y alimentar diseño de dominio, backlog y especificaciones técnicas.
