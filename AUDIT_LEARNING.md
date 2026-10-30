# SIGI-IEM / Bitácora Formativa — Memoria de auditoría

Este archivo conserva patrones técnicos reutilizables detectados en pruebas y auditorías.
No reemplaza la verificación del backend vivo ni autoriza a dar un hallazgo por corregido sin prueba posterior.

## BF-AUD-001 — Pérdida de borrador por re-render de una fase

- **Fecha de identificación:** 2026-09-24
- **Módulo:** Convivencia → Verificación
- **Severidad:** Alta
- **Síntoma observable:** el usuario diligencia campos de Verificación y, después de registrar una actuación secundaria (por ejemplo compromiso, evidencia, comunicación o citación), al intentar continuar aparece una validación como “Debe registrar el Plan de Verificación”.
- **Causa raíz:** una acción secundaria vuelve a cargar soportes desde Supabase y re-renderiza la vista antes de garantizar que los valores no guardados del formulario activo hayan sido capturados y persistidos. El re-render reconstruye los textarea con el último borrador disponible en backend, que puede ser anterior o vacío.
- **Patrón de riesgo:** `acción secundaria → lectura backend → re-render → pérdida de DOM no persistido`.
- **Señal temprana:** una función de una fase llama `cargarSoportesConvivencia(...)` y después `obsRenderDetalle(...)` sin capturar/persistir previamente el borrador actual.
- **Regla preventiva:** toda operación que pueda reconstruir la vista debe preservar primero el estado editable de la fase. Cuando exista backend, persistir el borrador antes de la operación y releerlo después.
- **Control incorporado en 10.242:** captura previa al render, autosalvado con debounce y protección de acciones secundarias en Verificación/Decisión/Cierre.
- **Prueba de regresión obligatoria:**
  1. escribir Plan de Verificación y Síntesis;
  2. registrar un compromiso;
  3. registrar una actuación;
  4. adjuntar evidencia o agregar enlace;
  5. registrar comunicación/citación si corresponde;
  6. comprobar que todos los campos siguen visibles;
  7. recargar la página;
  8. comprobar que el borrador se restaura;
  9. avanzar a Decisión y verificar la transición en Supabase.
- **Estado:** corrección en código fuente; despliegue y prueba E2E pendientes.

## Regla de auditoría acumulativa

Cada nuevo fallo debe registrar como mínimo: síntoma, módulo, severidad, causa raíz o hipótesis, evidencia, patrón generalizable, señal temprana, prevención, prueba de regresión y estado de verificación. Un patrón recurrente debe convertirse en control automático o checklist permanente.

## Auditoría 2026-10-26

**Alcance:** revisión de solo lectura de SIGI-IEM 10.346 publicado, PostgreSQL/RPC/RLS/Storage y registros HTTP del 25–26 de octubre. No se realizaron pruebas E2E autenticadas ni cambios de producción. Los estados pendientes no implican fallos reproducidos sobre expedientes reales.

### BF-AUD-002 — Escrituras antiguas eluden concurrencia versionada
- **Estado:** recurrente, confirmado en frontend publicado, definición SQL y tráfico real; corrección pendiente. **Severidad:** alta.
- **Evidencia:** 22 HTTP 200 a `guardar_borrador_valoracion_convivencia` en la ventana auditada; HTML 10.346 invoca funciones antiguas y no referencia las RPC `*_v2`; la RPC antigua escribe sin comprobar ni incrementar `version`; 14 borradores existentes siguen en versión 1.
- **Causa raíz:** migración incompleta de consumidores y coexistencia de rutas de escritura incompatibles.
- **Señal temprana:** llamadas a RPC antiguas después de publicar una nueva; contador de versión inmóvil mientras cambia `updated_at`.
- **Prevención:** inventario de todas las escrituras frontend/servidor, comparación de firmas y privilegios y test de concurrencia con dos sesiones.
- **Regla:** todas las rutas que modifican el mismo borrador deben compartir validación e incremento de versión; retirar acceso antiguo solo tras migrar dependencias.
- **Prueba mínima:** dos sesiones con versiones divergentes; guardar una, intentar guardar la otra y exigir conflicto explícito sin sobrescritura.

### BF-AUD-003 — Validación de evidencias no utilizada por el frontend
- **Estado:** recurrente, integración pendiente. **Severidad:** alta.
- **Evidencia:** HTML 10.346 llama `registrar_anexo_convivencia` y no la Edge Function verificada; la RPC antigua mantiene permiso de ejecución para `authenticated`. Los ocho anexos activos tienen objeto en Storage, pero carecen de hash registrado.
- **Causa raíz:** nueva función segura sin migración del consumidor ni cierre de la vía anterior.
- **Señal temprana:** función nueva desplegada pero sin llamadas en código/tráfico; archivos sin hash.
- **Prevención:** prueba integrada subida → verificación de servidor → registro → descarga autorizada, incluyendo negativos.
- **Regla:** no confirmar una evidencia solo por la respuesta de carga; validar objeto, fase, pertenencia, tamaño, integridad y permisos desde servidor.
- **Prueba mínima:** archivo válido, alterado, inexistente, ruta ajena y usuario no competente en entorno sintético.

### BF-AUD-004 — Datos no confiables interpolados en HTML
- **Estado:** riesgo estático confirmado, explotación no demostrada; corrección pendiente. **Severidad:** alta preventiva.
- **Evidencia:** `verifRenderEvidencia` construye HTML con nombre de archivo y enlaces sin escape visible; `obsRenderDetalleBase` asigna HTML generado a `innerHTML` con campos variables.
- **Causa raíz probable:** composición de HTML mediante concatenación sin codificación contextual uniforme.
- **Señal temprana:** interpolaciones de nombres, descripciones o URL dentro de `innerHTML`, `href` o atributos.
- **Prevención:** usar `textContent`, nodos DOM y validación de protocolos; análisis estático de sinks.
- **Regla:** datos de expedientes y evidencias nunca deben interpretarse como marcado ejecutable.
- **Prueba mínima:** cadenas inocuas con etiquetas y comillas deben renderizarse como texto; bloquear esquemas de enlace inseguros.

### BF-AUD-005 — Lectura directa de tabla sin privilegio
- **Estado:** recurrente, corrección pendiente. **Severidad:** media-alta.
- **Evidencia:** HTML 10.346 consulta `convivencia_criterios_clasificacion` directamente; `authenticated` carece de SELECT directo; existe RPC autorizada `obtener_criterios_clasificacion_convivencia`.
- **Causa raíz:** consumidor no migrado a interfaz de lectura autorizada.
- **Señal temprana:** `sb.from(...)` sobre tabla protegida pese a existir RPC.
- **Prevención:** matriz de permisos por rol y prueba de carga de clasificación desde frontend.
- **Regla:** consultas protegidas deben usar el contrato de acceso autorizado; nunca ampliar SELECT para ocultar un fallo de integración.
- **Prueba mínima:** rol competente consulta correctamente y rol ajeno recibe denegación controlada.

### Seguimiento obligatorio de BF-AUD-001
- **Estado:** pendiente de certificación E2E. Existen cinco borradores de Verificación; cuatro contienen Plan y tres Síntesis/Informe, lo que no demuestra pérdida de datos.
- **Regla de regresión:** antes de re-render por compromiso, actuación, evidencia, modal, comunicación o citación, capturar/persistir borrador; después restaurar y comparar los seis campos, recargar y avanzar. No declarar corregido sin prueba autenticada.

### Actividad y cobertura
- **Ventana:** 2026-10-25 00:55 a 2026-10-26 00:55 (America/Bogota).
- **API:** 1.978 HTTP 200 y 1 HTTP 204; **Edge Functions:** 1.441 HTTP 200 y 1 HTTP 204; 1.440 ejecuciones de recordatorios. Sin 4xx/5xx observados en esas fuentes, pero esto no certifica módulos sin prueba.
- **Cambios de datos agregados:** 7 fuentes documentales y 1 remisión creadas durante la ventana. No se inspeccionaron datos personales.
- **Pruebas:** inspección estática, SQL y tráfico PASS para existencia/configuración; E2E autenticado NO EJECUTADO por ausencia de sesiones de prueba.

## Auditoría 2026-10-27

**Alcance y versión:** revisión de solo lectura de Supabase `SIGI-IEM-DEV`, frontend publicado 10.346 (HTTP 200, sin despliegues posteriores al 20-oct), código de RPC y registros de 2026-10-26 00:57:48 a 2026-10-27 00:57:48 (America/Bogota). No se realizaron E2E autenticados ni escrituras en expedientes. No se declara ningún hallazgo corregido sin retest.

### BF-AUD-006 — Validación de documento con HTTP 500
- **Estado:** nuevo evento observado; causa raíz pendiente. **Severidad:** alta operativa en Asistencia / gestión de personal.
- **Evidencia:** 1 respuesta HTTP 500 en `/rest/v1/rpc/asistencia_personal_validar_documento` (26-oct 15:25:23 UTC). La función existe; el análisis de privilegios indica que `authenticated` no tiene EXECUTE directo. No está demostrado que la denegación sea la causa del 500.
- **Patrón:** error interno en función de validación de identidad/documento; posible diferencia entre canal autorizado y consumidor.
- **Señal temprana:** HTTP 5xx en validación y discrepancia entre permisos de RPC y ruta de invocación.
- **Prevención:** probar RPC desde el mismo canal y rol usados por la interfaz; inspeccionar error PostgreSQL/Edge con correlación de request, sin exponer documentos.
- **Regla:** ninguna validación de documento debe fallar silenciosamente ni exponer identificadores; errores deben diferenciar formato, permisos y fallo interno.
- **Prueba mínima:** documentos sintéticos válidos, no encontrados, duplicados, atípicos y usuario sin autorización; verificar mensajes y ausencia de 5xx.

### BF-AUD-007 — Errores 502/503 en núcleo de Asistencia
- **Estado:** nuevo evento observado; causa pendiente. **Severidad:** alta operativa.
- **Evidencia:** `asistencia-docente-v17-core` devolvió HTTP 502 (26-oct 15:01:23 UTC) y HTTP 503 (26-oct 18:01:33 UTC); además 8 HTTP 409 en el mismo núcleo. No se demuestra pérdida de marcaciones ni que 409 sea un fallo.
- **Patrón:** error transitorio del servicio o dependencia durante operación sensible, con estado de marcación incierto.
- **Señal temprana:** 5xx, reintentos repetidos, botón que no termina y divergencia entre respuesta y marcación registrada.
- **Prevención:** idempotencia por request_id, consulta del resultado tras timeout, límites de reintentos y alertas por 5xx.
- **Regla:** ante respuesta incierta no repetir una marcación sin verificar primero su estado real.
- **Prueba mínima:** simular 502/503 y desconexión en entorno aislado, verificar recuperación y exactamente una marcación efectiva.

### BF-AUD-008 — Estado de clasificación marcado como cargado antes de completar la lectura
- **Estado:** nuevo hallazgo estático, no reproducido E2E. **Severidad:** media-alta.
- **Evidencia:** en frontend 10.346, `loadGeneralPersisted(c)` establece `c._sigi260GeneralLoaded=true` antes de `sb.from('convivencia_criterios_clasificacion')`; si la consulta devuelve error, retorna sin restablecer la bandera. La consulta directa a tabla protegida ya figura en BF-AUD-005.
- **Patrón:** marca de carga exitosa establecida antes de conocer el resultado; fallo temporal bloquea reintentos.
- **Señal temprana:** carga de clasificación incompleta sin error visible, ausencia de nuevas consultas al reabrir.
- **Prevención:** asignar loaded=true únicamente después de respuesta válida, o usar estado `loading/loaded/error` y reintento controlado.
- **Regla:** nunca marcar datos remotos como cargados si su consulta falló.
- **Prueba mínima:** provocar rechazo de lectura, corregir el canal autorizado, reabrir y verificar recuperación sin recargar toda la app.

### Seguimiento de hallazgos recurrentes
- **BF-AUD-001 (re-render):** cinco borradores de Verificación; cuatro con Plan y tres con Síntesis/Informe. No se observó escritura de borradores en esta ventana; la conservación de los seis campos después de compromisos, actuaciones, evidencias y modales sigue sin E2E.
- **BF-AUD-002 (concurrencia):** frontend 10.346 sigue llamando RPC antiguas; no aparecen referencias a las RPC v2 en el HTML. Catorce borradores en versión 1; cero modificados en la ventana. Pendiente.
- **BF-AUD-003 (evidencias):** ocho anexos activos, ocho sin hash; frontend continúa llamando RPC antigua. Pendiente.
- **BF-AUD-004 (HTML):** interpolaciones sin escape visible en `verifRenderEvidencia`; sin prueba de explotación. Pendiente.
- **BF-AUD-005 (clasificación):** lectura directa de tabla protegida y bandera de carga prematura; pendiente.
- **RAI:** trece rutas activas; nueve `verification_policy=NULL`. El frontend utiliza `REQUIRED_SAFE_DEFAULT` para rutas específicas sin política, por lo que no se presume omisión automática de Verificación. Revisar políticas y recorridos.
- **Asistencia:** 19 HTTP 400 en salida manual, 8 en marcación y 4 en estado kiosco; distinguir rechazo legítimo de defecto funcional. 
- **Seguridad:** 40 avisos informativos RLS sin políticas (21 esquema candidato y 19 public), 9 avisos de funciones privilegiadas para anon y 292 para authenticated. Son señales de revisión, no vulnerabilidades demostradas.
- **Datos agregados:** 597 migraciones, 160 fuentes documentales, 16 remisiones, 0 solicitudes de firma; sin cambios respecto al cierre anterior. 
- **Cobertura:** frontend HTTP 200 y SQL/RPC inspeccionados; pruebas de componentes de ejecución, integración autenticada y E2E por rol NO EJECUTADAS por ausencia de sesiones y entorno sintético. Ningún hallazgo se declara corregido hoy.


## Auditoría 2026-10-28

**Alcance:** consulta directa y no destructiva de Vercel, Supabase (PostgreSQL, RPC, Edge Functions, Storage y logs) y repositorio. Ventana de logs 2026-10-27 01:04 a 2026-10-28 01:04 America/Bogota. Despliegue SIGI-IEM 10.346 sigue READY (20-oct), sin nuevas migraciones. No se ejecutaron pruebas E2E autenticadas por ausencia de sesiones y entorno sintético. No se corrigieron expedientes ni funciones.

### BF-AUD-009 — Consulta de marcación por requestId sin vínculo con dispositivo
- **Estado:** nuevo riesgo de autorización confirmado por inspección estática; acceso indebido no explotado. **Severidad:** alta.
- **Evidencia:** Edge Function \`asistencia-docente-v17-core\` versión 2 tiene \`verify_jwt=false\` y usa credencial de servicio. En \`verify(body)\`, si se aporta \`requestId\`, llama \`asistencia_verificar_request\` sin exigir \`deviceToken\`. La RPC interna \`verificar_request_marcacion\` busca solo por UUID y devuelve \`personalId\`, tipo, hora y estado de jornada. La validación de Origin/CORS no sustituye la autorización del sujeto.
- **Causa raíz:** recuperación idempotente basada en identificador de solicitud sin comprobación de pertenencia al dispositivo/usuario.
- **Señal temprana:** endpoint de lectura con rol privilegiado que recibe únicamente un identificador de registro y devuelve datos personales.
- **Prevención:** matriz de autorización por recurso; pruebas negativas de token ausente/ajeno y requestId ajeno.
- **Regla:** cada consulta de resultado debe demostrar pertenencia al actor, no solo conocer el identificador; conservar idempotencia sin divulgar datos ajenos.
- **Prueba mínima:** en entorno aislado, comprobar que el dispositivo propietario recupera su resultado y otro dispositivo no recibe datos, aun con el requestId correcto.

### BF-AUD-010 — Modalidad QR indicada sin comprobación del token QR
- **Estado:** nuevo riesgo estático; depende del requisito funcional del QR. **Severidad:** media-alta, alta si QR acredita presencia.
- **Evidencia:** \`asistencia-docente-v17-core\` recibe \`markQr\`, pero solo utiliza \`!!body.qrToken\` para \`qrPresent\` en metadatos; no verifica el valor, firma, vigencia ni asociación del QR. La función SQL de marcación no contiene referencias a \`qrToken\` ni \`qrPresent\`.
- **Causa raíz probable:** modo QR representado como etiqueta de método y no como comprobación de credencial.
- **Señal temprana:** parámetro \`qrToken\` usado solo como booleano; ausencia de verificación del código.
- **Prevención:** contrato formal del QR (si es requisito de presencia), validación criptográfica y caducidad del token en servidor; prueba negativa con token inválido.
- **Regla:** no presentar una marcación como verificada por QR si el servidor no valida efectivamente ese QR.
- **Prueba mínima:** token QR válido, ausente, alterado, vencido y reutilizado; comprobar rechazo seguro según el protocolo.

### BF-AUD-011 — Códigos 409 de Asistencia requieren diagnóstico por código interno
- **Estado:** recurrente, nueva desagregación verificable. **Severidad:** media-alta operativa.
- **Evidencia:** 15 HTTP 409 en \`asistencia-docente-v17-core\`; logs internos: 7 \`GPS_PRECISION_TOO_LOW\`, 5 \`OUTSIDE_GEOFENCE\` y 3 \`OFFLINE_STATE_CONFLICT\`. La función mapea múltiples validaciones al mismo HTTP 409. No se observaron HTTP 5xx en esta ventana.
- **Causa raíz:** distintas causas de rechazo agrupadas bajo un mismo estado HTTP; no se ha demostrado error de geocerca ni GPS.
- **Señal temprana:** aumento de 409 sin clasificación por \`code\`; usuarios que reintentan sin comprender motivo.
- **Prevención:** tableros por código, pruebas de ubicación con precisión insuficiente, geocerca y colas offline, consulta de estado antes de reintentar.
- **Regla:** nunca clasificar un 409 como duplicado sin leer el código de negocio; no registrar marcación rechazada como exitosa.
- **Prueba mínima:** simular cada condición, confirmar mensaje correcto, preservación de solicitud offline y ausencia de doble marcación.

### Seguimiento de patrones recurrentes y cobertura
- **BF-AUD-001:** cinco borradores de Verificación; cuatro con Plan y tres con Síntesis/Informe, seis claves presentes en los cinco. La pérdida por re-render sigue sin E2E. Preservar antes de compromisos, actuaciones, evidencias y modales; comprobar tras recarga.
- **BF-AUD-002:** 14 borradores, ninguno con versión superior a 1; funciones antiguas aún ejecutables y sin incremento de versión. Sin escrituras de borrador en la ventana. No declarar resuelto.
- **BF-AUD-003:** ocho anexos activos, ocho objetos localizados en Storage, ocho sin hash. Registro verificado pendiente.
- **BF-AUD-004/005/008:** riesgos de HTML, consulta directa de clasificación y bandera de carga prematura pendientes por ausencia de nueva publicación y retest.
- **API:** 2776 HTTP 200, 2 HTTP 201, 4 HTTP 204, 23 HTTP 400 y 7 HTTP 401. Los 400 corresponden a 15 marcaciones y 8 consultas de estado kiosco; los 401 a cinco consultas de coordinador académico, una de remisiones y una de periodos. No se determinó si las 401 son expiración de sesión, denegación correcta o regresión.
- **Edge:** 1710 HTTP 200, 250 HTTP 204, 15 HTTP 409. Los tres HTTP 5xx de la ventana anterior no se repitieron, pero no se declaran corregidos.
- **Postgres:** 23 ERROR P0001 y 8 ERROR 42501; correlación causal individual no establecida.
- **Datos agregados:** 160 fuentes documentales, 16 remisiones, 13 rutas activas (nueve con política NULL), 0 solicitudes de firma, 40 avisos RLS sin políticas. Sin cambios.
- **Pruebas:** PASS de inspección de despliegue, existencia de RPC y conciliación básica Storage; FAIL estático de controles de autorización en recuperación por requestId y de verificación de QR; componentes interactivos, integración autenticada y E2E NO EJECUTADOS. No hubo cambios de producción.


## Auditoría 2026-10-29

**Alcance:** inspección de solo lectura de despliegue Vercel, frontend HTML publicado, funciones y permisos PostgreSQL, tablas de borradores/actas/evidencias, Edge Function de Asistencia y logs Supabase. Ventana 2026-10-28 01:01:07 a 2026-10-29 01:01:07 (America/Bogota). Versión publicada 10.346 (despliegue 20-oct), backend ACTIVE_HEALTHY, 597 migraciones. Sin E2E autenticadas por falta de sesiones aisladas; no se alteraron expedientes ni código productivo.

### BF-AUD-012 — Rechazos repetidos de dispositivo no vinculado

- **Estado:** nuevo patrón de frecuencia; causa individual y regresión no demostradas. **Severidad:** media-alta en Asistencia.
- **Evidencia:** 18 eventos \`ATTENDANCE_ERROR\` con código \`DEVICE_NOT_LINKED\` en logs de Edge Functions durante la ventana; 18 HTTP 400 en \`asistencia_contexto_inicio\`. La coincidencia de totales no establece correspondencia uno a uno. El código de Asistencia devuelve el código de error explícito. No se inspeccionaron identificadores personales.
- **Causa raíz posible:** desvinculación real, token/caché desincronizado, reinstalación, cambio de navegador o problema de persistencia del token; requiere correlación por identificador técnico anonimizado.
- **Señal temprana:** aumento de \`DEVICE_NOT_LINKED\` tras actualizar PWA, reiniciar sesión o cambiar dispositivo.
- **Prevención:** prueba de vinculación/revinculación, persistencia de token, actualización de PWA y validación de un solo dispositivo, con autorización administrativa.
- **Regla:** ante \`DEVICE_NOT_LINKED\`, diferenciar seguridad legítima de pérdida involuntaria de vínculo; no revincular automáticamente ni omitir controles.
- **Prueba mínima:** vincular dispositivo sintético, marcar, recargar, actualizar PWA y reiniciar; confirmar vínculo persistente. Revocar y verificar denegación y flujo de recuperación autorizado.

### BF-AUD-013 — Conflictos offline y reintentos repetidos

- **Estado:** recurrente, pendiente de reproducción controlada. **Severidad:** alta para integridad de Asistencia.
- **Evidencia:** 30 HTTP 409 en \`asistencia-docente-v17-core\`; logs de aplicación con 5 \`OFFLINE_STATE_CONFLICT\` y 1 \`OFFLINE_CAPTURE_EXPIRED\`. También 5 \`OUTSIDE_GEOFENCE\` y 2 \`GPS_PRECISION_TOO_LOW\`; los totales de eventos de aplicación no son equivalentes a solicitudes HTTP únicas.
- **Causa posible:** estado local desactualizado, reintentos de una misma solicitud, expiración de captura o validaciones de ubicación; no se demostró duplicación efectiva.
- **Señal temprana:** múltiples 409 para el mismo intento, cola offline que no se resuelve, estado incierto y botón que invita a remarcar.
- **Prevención:** request_id idempotente, verificación del resultado real, cola persistente y distinción entre rechazo legítimo y conflicto recuperable.
- **Regla:** nunca crear otra marcación para resolver un conflicto sin consultar primero el resultado del intento original.
- **Prueba mínima:** dos reintentos offline del mismo intento, cambio de estado entre captura y envío, captura vencida; exigir una sola marcación válida y mensajes comprensibles.

### BF-AUD-014 — Códigos HTTP 400 agrupados ocultan módulos afectados

- **Estado:** nueva señal operativa, fallo funcional no confirmado. **Severidad:** media.
- **Evidencia:** 38 HTTP 400 en API: 18 en \`asistencia_contexto_inicio\`, 14 en \`asistencia_registrar_marcacion\`, 6 en \`asistencia_apps_script_estado_kiosco\`. PostgreSQL registró 38 errores \`P0001\`; no se estableció correlación individual.
- **Señal temprana:** incremento de 400 en contexto inicial o kiosco tras una actualización, incluso sin 5xx.
- **Prevención:** tablero de rechazos por RPC, código funcional y tipo de dispositivo, sin registrar documentos ni tokens.
- **Regla:** clasificar cada 400 como rechazo esperado, error de datos o defecto; no contar todo 400 como fallo ni todo 200 como éxito E2E.
- **Prueba mínima:** contexto de dispositivo vinculado/no vinculado, marcación válida/inválida y estado de kiosco, comparando mensajes y registros de backend.

### Seguimiento de riesgos previos y regresión obligatoria

- **BF-AUD-009:** persiste en código de Edge Function \`asistencia-docente-v17-core\` versión 2: \`verify(requestId)\` llama \`asistencia_verificar_request\` sin requerir pertenencia por token. No se explotó ni corrigió. Prioridad alta de seguridad.
- **BF-AUD-010:** \`qrToken\` sigue siendo indicador de presencia en \`markQr\`, sin verificación de autenticidad visible en ese recorrido. Revisar contrato del QR. Pendiente.
- **BF-AUD-001:** frontend 10.346 mantiene campos \`verifPlan\` y \`verifInforme\`; cinco borradores de Verificación contienen ambas claves, cuatro Plan con texto y tres Informe con texto. Cero borradores modificados en la ventana. Sin E2E de compromiso/actuación/evidencia/modal + recarga. **No declarar corregido.**
- **BF-AUD-002:** frontend publicado contiene referencias a \`guardar_borrador_fase_convivencia\` y \`guardar_borrador_valoracion_convivencia\`, sin \`*_v2\`; ambas RPC antiguas siguen ejecutables por \`authenticated\`. Catorce borradores, ninguno con versión >1. Pendiente.
- **BF-AUD-003:** frontend continúa con \`registrar_anexo_convivencia\` y sin referencia a Edge Function verificada. Ocho anexos ACTIVO, ocho objetos correspondientes en Storage, ocho sin hash SHA-256. Pendiente.
- **BF-AUD-004/005/008:** interpolaciones HTML, lectura directa de clasificación y bandera de carga prematura continúan como riesgos del frontend 10.346, sin retest autenticado.
- **RAI/actas:** 13 rutas activas, 9 sin política explícita; 160 fuentes documentales, 16 remisiones, 0 solicitudes de firma. No se observaron nuevas escrituras de borradores; actas, firma, permisos y transiciones siguen sin E2E.
- **Cobertura:** HTTP de frontend 200; logs API 2414×200, 2×201, 4×204, 38×400; Edge 1700×200, 205×204, 30×409; cero 5xx en ventana. Los 5xx anteriores no se consideran corregidos sin prueba causal. Vercel no entregó logs históricos por retención del plan. No hubo cambios en producción.

**Próximo control obligatorio:** pruebas por rol y entorno sintético de preservación de los seis campos de Verificación antes/después de registrar compromiso, actuación, evidencia, modal, recarga y transición; autorización de \`verify(requestId)\`; autenticidad QR; vínculo del dispositivo; reintentos offline; migración RPC v2 y evidencias verificadas.


## Auditoría 2026-10-30

**Alcance:** auditoría de solo lectura de Bitácora Formativa, Vercel y Supabase \`SIGI-IEM-DEV\`, ventana 2026-10-29 01:04–2026-10-30 01:04 (America/Bogota). SIGI-IEM permanece en despliegue 10.346 del 20-oct; Asistencia recibió nueva publicación el 29-oct 18:31 local. Backend ACTIVE_HEALTHY; 597 migraciones registradas. No hubo pruebas E2E autenticadas ni cambios de expedientes, permisos o producción por esta auditoría.

### BF-AUD-015 — Prueba de notificaciones push falla con error genérico
- **Estado:** NUEVO, fallo de servicio observado, causa técnica final no determinada. **Severidad:** alta operativa.
- **Evidencia:** 10 HTTP 400 de \`asistencia-recordatorios-api\` v2 entre 2026-10-29 23:27 y 23:32 UTC; 10 logs \`SMART_REMINDER_API_ERROR Received unexpected response code\`. La función \`sendTest\` llama \`webpush.sendNotification\`; su catch devuelve el mismo HTTP 400 genérico sin status del proveedor. Se registraron errores incluso después de la última publicación de Asistencia (23:31:24 UTC).
- **Hipótesis:** rechazo del proveedor push, suscripción caducada, VAPID o configuración; no se demuestra cuál.
- **Señal temprana:** botón de prueba sin notificación y múltiples HTTP 400 con mensaje genérico.
- **Prevención/regla:** conservar status y tipo de error del proveedor en logs técnicos anonimizados; diferenciar suscripción inválida, credenciales, permisos del navegador y falla transitoria. Nunca declarar entrega por HTTP 200 de cron.
- **Prueba mínima:** con suscripción sintética y dispositivo de prueba, solicitar push y confirmar recepción; probar 404/410, VAPID inválido, permiso denegado, desconexión y reintento idempotente.

### BF-AUD-016 — Resolver alerta no conserva actor y cron puede reabrirla
- **Estado:** NUEVO, incompatibilidad de lógica confirmada por SQL; re-apertura efectiva no reproducida. **Severidad:** alta en trazabilidad.
- **Evidencia:** \`asistencia_alerta_marcacion_resolver\` escribe \`estado='RESUELTA'\` y \`resuelta_at\`, pero \`p_actor\` aparece únicamente en la respuesta JSON, sin guardarse. No hay triggers de auditoría en \`asistencia_docente_candidate.alertas_marcacion\`. \`asistencia_alertas_marcacion_sincronizar\` ejecuta \`ON CONFLICT ... DO UPDATE SET estado='PENDIENTE', resuelta_at=null\` cuando persiste el motivo de alerta. Cron v5 ejecuta sincronización periódica.
- **Señal temprana:** alertas resueltas que reaparecen o sin identidad/motivo del cierre.
- **Prevención/regla:** toda resolución administrativa debe registrar actor, fecha, motivo y evento; la reconciliación automática debe distinguir nueva incidencia de alerta ya resuelta y no borrar trazabilidad.
- **Prueba mínima:** alerta sintética → resolver con actor autorizado → ejecutar sincronización → verificar estado y auditoría conforme a regla institucional; repetir si aparece un nuevo evento real.

### BF-AUD-017 — Desajuste temporal de modalidad AUTO y restricción SQL
- **Estado:** NUEVO evento, **corrección estructural verificada**, prueba E2E pendiente. **Severidad:** media.
- **Evidencia:** PostgreSQL SQLSTATE 23514 el 29-oct 22:48:13 UTC por \`asistencia_recordatorios_modo_check\`. A las 22:48:37 UTC se registró DDL de sustitución de restricción. Consulta actual confirma que acepta \`SEMANA\`, \`DIA\` y \`AUTO\`; existen 2 suscripciones activas con modo AUTO.
- **Causa:** desalineación temporal entre nuevo contrato de frontend/API y restricción de base de datos.
- **Señal temprana:** errores 23514 tras introducir nuevo valor enumerado.
- **Prevención/regla:** desplegar migración compatible antes de activar el nuevo consumidor; comprobar constraints y payloads.
- **Prueba mínima:** alta, actualización, desactivación y reactivación de suscripción sintética con modo AUTO; recarga y verificación de persistencia. No considerar funcionalidad completa corregida sin retest.

### BF-AUD-018 — Estado de envío y cobertura de recordatorios insuficientemente certificados
- **Estado:** NUEVO riesgo estático y brecha operativa; falla de actualización no reproducida. **Severidad:** media-alta.
- **Evidencia:** cron v5 envía webpush y llama \`markSent\`, que no inspecciona \`error\` de \`update\`; \`release\` tampoco comprueba errores de \`delete\`. Existen 13 registros \`SENT\`, 0 \`PENDING\`, 2 suscripciones activas y 37 alertas \`PENDIENTE\` de nivel 3 del 29-oct (19 ENTRADA y 18 SALIDA). \`SENT\` significa registro de aceptación/envío, no recepción visible.
- **Señal temprana:** push reportado como enviado sin recibirse, reclamaciones estancadas o ausencia de suscripciones frente a alertas.
- **Prevención/regla:** comprobar resultado de todas las escrituras de estado, reconciliar reclamaciones pendientes y distinguir envío aceptado de notificación mostrada. Medir cobertura de usuarios habilitados sin exponer datos personales.
- **Prueba mínima:** simular fallo de UPDATE después de push, recuperación y reintento sin duplicación; probar suscripciones caducadas, alta de nuevos dispositivos y confirmación de recepción.

### BF-AUD-019 — Cambios directos de esquema sin migración registrada
- **Estado:** NUEVO, deriva de esquema confirmada en ventana. **Severidad:** media-alta para reproducibilidad y reversión.
- **Evidencia:** seis entradas PostgreSQL con \`CREATE OR REPLACE FUNCTION\` entre 29-oct 22:48 y 23:02 UTC, además de \`ALTER TABLE asistencia_recordatorios\`; contador de \`supabase_migrations.schema_migrations\` sigue en 597, última versión \`20261021120742\`.
- **Causa probable:** ajustes manuales aplicados fuera del historial de migraciones.
- **Señal temprana:** objetos y definiciones modificados sin aumento del contador o sin archivo versionado.
- **Prevención/regla:** reconciliar el estado vivo con migraciones idempotentes versionadas, conservar diffs, rollback y pruebas; nunca reejecutar DDL a ciegas sobre producción.
- **Prueba mínima:** recrear esquema en entorno aislado desde migraciones y comparar definiciones de funciones, constraints, GRANT y RLS.

### Seguimiento recurrente, estado y pruebas
- **BF-AUD-001:** cinco borradores de Verificación, cuatro con \`verifPlan\` no vacío y tres con \`verifInforme\`; se comprobaron seis claves distintas. Ningún borrador modificado en 24 h. Sin prueba de compromiso/actuación/evidencia/modal + recarga. **Pendiente de E2E, no corregido.**
- **BF-AUD-002:** 14 borradores, cero con versión >1. RPC antiguas \`guardar_borrador_fase_convivencia\` y \`guardar_borrador_valoracion_convivencia\` siguen con EXECUTE de authenticated, sin referencias de versión en sus definiciones; v2 también disponibles. **Pendiente.**
- **BF-AUD-003:** ocho anexos activos, ruta de registro antiguo en frontend 10.346 y falta de hash documentada anteriormente. Sin nueva prueba de descarga autenticada ni conciliación criptográfica. **Pendiente.**
- **BF-AUD-004/005/008:** HTML no confiable, clasificación directa y marca de carga prematura; frontend SIGI 10.346 no se ha actualizado. **Pendientes, sin retest interactivo.**
- **BF-AUD-009/010:** Edge \`asistencia-docente-v17-core\` permanece versión 2; riesgos de consulta por requestId y validación QR sin corrección certificada. **Pendientes.**
- **BF-AUD-011/013:** ocho HTTP 409 de núcleo de Asistencia; ocho eventos \`ATTENDANCE_ERROR\` (4 \`OFFLINE_CAPTURE_EXPIRED\`, 3 \`OUTSIDE_GEOFENCE\`, 1 \`GPS_PRECISION_TOO_LOW\`). No se observó \`DEVICE_NOT_LINKED\` ni \`OFFLINE_STATE_CONFLICT\` en esta ventana; no implica corrección.
- **HTTP:** API 3012×200, 3×201, 11×204, 12×400 (8 marcación, 4 kiosco); Edge 1594×200, 174×204, 10×400 (recordatorios), 8×409. Sin 5xx observados. PostgreSQL: 12×P0001, 1×23514.
- **Seguridad:** 41 avisos \`rls_enabled_no_policy\` (antes 40), incluida tabla de envíos de recordatorios con RLS y sin SELECT para anon/auth; revisar sin abrir permisos indiscriminadamente. Funciones de alertas listar/resolver/sincronizar ejecutables solo por service_role.
- **Actas/RAI:** 13 rutas activas, 9 políticas de Verificación NULL; no hubo E2E de actas, remisiones, firmas, Storage, Auth o roles. Los datos agregados históricos de documentos no fueron recalculados.
- **Pruebas ejecutadas:** PASS de inspección de despliegues, restricciones actuales, permisos RLS de recordatorios, contadores agregados y firmas de RPC; FAIL estático de trazabilidad de resolución y control de errores de estado de envío; FAIL operativo de pruebas push; NO EJECUTADAS pruebas de componentes en navegador, integración autenticada y E2E por falta de sesiones/entorno aislado.
- **Regla prioritaria:** preservar/persistir el borrador de fase antes de cualquier re-render; verificar después que \`verifPlan\`, \`verifInforme\` y los otros cuatro campos siguen cargados. No dar por corregido sin E2E autenticada.
