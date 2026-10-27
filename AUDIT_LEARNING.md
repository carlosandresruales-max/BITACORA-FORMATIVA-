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
