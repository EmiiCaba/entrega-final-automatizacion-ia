# Ecosistema de Automatización IA · Clasificación de leads con aprobación humana

**Caso:** Clínica Veterinaria y Peluquería Canina (negocio ficticio)
**Herramientas:** n8n · Airtable · OpenAI (gpt-6-luna) · Slack · Gmail
**Entrega Final · Coderhouse**

El sistema recibe cada consulta de un cliente nuevo desde un formulario de Airtable, la clasifica con IA como **VIP** o **Normal**, redacta una respuesta usando solo información validada de la clínica (RAG) y **solo la envía por mail cuando una persona la aprueba en Slack** (Human in the Loop). Cada paso y cada error queda registrado, y un dashboard público muestra los KPIs.

![Diagrama de arquitectura](img/arquitectura.png)

🎥 **Video demo (hasta 3 min):** [https://youtu.be/ySERYyfy0AA](https://youtu.be/ySERYyfy0AA)
📄 **Documentación completa:** [docs/entrega-final.md](docs/entrega-final.md)

---

## Stack (4 categorías)

| Categoría | Herramienta | Función |
|---|---|---|
| Orquestador | n8n | Flujo principal: disparo, validación, IA, aprobación, envío y rutas de error |
| Base de datos | Airtable | Leads, servicios, conocimiento (RAG), errores y KPIs |
| Procesamiento IA | OpenAI · gpt-6-luna | Clasifica el lead y redacta la respuesta en JSON |
| Canal de salida | Slack + Gmail | Slack para la aprobación humana, Gmail para responder al cliente |

## Criterio de clasificación

- **VIP:** 2 o más perros, o pide el *Plan mensual de peluquería*.
- **Normal:** cualquier otro caso.
- La IA devuelve también un puntaje de 1 a 10 y el motivo.

## Cómo funciona

```
Formulario Airtable
  → Trigger · Nuevo lead (cada 1 min, solo registros nuevos)
  → FILTRO · ¿Estado Pendiente?            (anti-bucle)
  → Lead: Procesando
  → Validar datos mínimos                  ── falla → Registrar error
  → Consultar conocimiento (RAG)
  → Armar contexto                         (Code)
  → IA · Clasificar y redactar             ── falla → Registrar error (3 reintentos)
  → ¿Respuesta IA válida?                  ── falla → Registrar error
  → Guardar resultado IA
  → HITL · Pedir aprobación en Slack       (Aprobar / Rechazar)
  → ¿Aprobado?
       ├─ sí → Enviar mail (Gmail) → Estado: Enviado
       └─ no → Estado: Rechazado
```

**Ciclo de estados:** `Pendiente → Procesando → Procesado por IA → Enviado / Rechazado`. Cualquier falla lleva el lead a `Error` y crea un registro en la tabla Errores.

## Base de datos (Airtable)

| Tabla | Contenido | Relación |
|---|---|---|
| Leads | Datos del formulario + resultado de la IA y del humano | Vinculada a Servicios, Errores y Panel KPI |
| Servicios | Baño y corte, Plan mensual, Vacunación, Desparasitación, Consulta clínica | Muchos leads → 1 servicio |
| Errores | Fecha, Nodo, Tipo, Mensaje, Lead | 1 lead → muchos errores |
| Panel KPI | Contadores (Count / Rollup) y fórmulas de KPIs | Muchos leads → 1 panel |
| Conocimiento | Un tema por registro (base RAG) | Leída por n8n |

Los esquemas JSON de cada integración están en la [documentación](docs/entrega-final.md#esquemas-json-de-transferencia).

## Seguridad y resiliencia

- **Minimización:** nombre, email y teléfono nunca llegan a OpenAI.
- **Error handlers:** 4 rutas de error (datos incompletos, falla de API, JSON inválido, falla de Gmail) que registran el error y pasan el lead a `Error`.
- **HITL:** ningún mail sale sin un *Aprobar* explícito en Slack.
- **Credenciales:** guardadas en n8n; el JSON exportado no las incluye.

## Costos

Con gpt-6-luna, procesar **1.000 leads cuesta unos USD 0,33**, un 95% menos que un modelo de gama media. Detalle en la [matriz de costos](docs/entrega-final.md#3-optimización-de-costos).

## Dashboard y enlaces públicos

| Recurso | Enlace |
|---|---|
| Dashboard de KPIs (Shared View) | [Panel KPI](https://airtable.com/appKQlv5Lk7r09PVh/shr6cv2dVvK2CPJd2) |
| Leads (solo lectura, sin email ni teléfono) | [Vista de solo lectura](https://airtable.com/appKQlv5Lk7r09PVh/shrPiTnhaOWvlK00k) |

**KPIs:** tasa de aprobación, volumen de salida (mails enviados), tasa de error y leads VIP.

## Pruebas (6 ejecuciones · 18-oct-2026)

| # | Prueba | Resultado |
|---|---|---|
| 1 | Camino feliz · VIP (Martina, 3 perros) | ✅ VIP → aprobado → mail enviado |
| 2 | Camino feliz · Normal (Juan, 1 perro) | ✅ Normal → aprobado → mail enviado |
| 3 | VIP rechazado (Sofía, plan mensual) | ✅ Rechazado, no sale mail |
| 4 | Sin email (Pedro) | ✅ Error "Datos incompletos", sin llamar a la IA |
| 5 | 0 perros (Lucía) | ✅ Error "Datos incompletos" |
| 6 | Falla de API (Jorge) | ✅ 3 reintentos → Error "Falla API IA" |

Resultado en el dashboard: 6 leads · 66,7% de aprobación · 2 enviados · 2 VIP · 50% de error (esperable: 3 de las 6 pruebas eran caminos de error).

Capturas en [`evidencias/`](evidencias/).

## Cómo importarlo

1. En n8n: **Workflows → Import from File** y elegir [`workflow/workflow.json`](workflow/workflow.json).
2. Configurar las credenciales de **Airtable**, **OpenAI**, **Slack** y **Gmail**.
3. Ajustar los IDs de base/tablas de Airtable y el ID del canal de Slack.
4. Publicar el workflow y cargar un lead desde el formulario.

## Estructura del repositorio

```
├── README.md
├── docs/
│   └── entrega-final.md    # Documentación completa de la entrega
├── workflow/
│   └── workflow.json       # Flujo exportado de n8n (sin credenciales)
├── evidencias/             # Capturas de las ejecuciones
└── img/
    └── arquitectura.png    # Diagrama de arquitectura
```

---

**Autora:** Emilce Paola Caba · Octubre 2026
