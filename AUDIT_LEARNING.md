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
