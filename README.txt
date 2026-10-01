Tablero Inteligente de Seguridad Quirúrgica

MVP de hackatón: tablero digital de una cirugía en curso con checklist de 3 fases, 7 hitos
de tiempo (sin salida a recuperación), tecnovigilancia de equipos e insumos, anestesia
con varios tipos, alertas por reglas (modal al quedar 2 unidades y al agotarse), drenes,
novedades, trazabilidad (quién registra vs. quién es responsable) y Wally, el agente de voz
con palabra de activación.



No es una IA médica autónoma. Wally solo traduce frases en acciones, pide confirmación y
responde hablando. Las validaciones y alertas son reglas deterministas y auditables.

Documentación:





Manual de usuario — operación en quirófano, Wally, alertas y pantallas.



Manual de sistema — arquitectura, modelo de datos, API y despliegue.



Arranque rápido (Windows)

Requisitos: Python 3.11+, Node 20+, Google Chrome (para el reconocimiento de voz).

.\iniciar.ps1          # instala lo necesario la primera vez y abre http://localhost:5173
.\iniciar.ps1 -Reset   # igual, pero reinicia los datos de la demo

Manual:

# Backend  ->  http://127.0.0.1:8000/docs  (Swagger)
cd backend
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
.venv\Scripts\python -m app.seed            # reinicia la semilla (un comando)
.venv\Scripts\python -m uvicorn app.main:app --port 8000

# Frontend ->  http://localhost:5173
cd frontend
npm install
npm run dev

Pruebas: cd backend; .venv\Scripts\python -m pytest -q

También hay un botón Reiniciar demo en la cabecera (POST /api/demo/reset).

Base de datos: SQLite en backend/data/tablero.db. Al arrancar, si la BD está vacía o su
esquema no coincide con los modelos (tabla o columna faltante), se recrea y se vuelve a sembrar.

Flujo temporal (7 hitos)





Ingreso del paciente



Tecnovigilancia (automático al completar equipos + insumos)



Verificación de seguridad del paciente (automático)



Verificación de anestesia (automático)



Inicio de anestesia



Inicio de cirugía



Fin de cirugía

No existe el hito «Salida a recuperación». El orden es obligatorio; un salto requiere justificación
y queda en trazabilidad.

Wally (agente de voz)

Escucha de forma continua, pero no ejecuta hasta oír la palabra Wally (acepta transcripciones
cercanas: wali, guali, oye Wally…).

ESCUCHANDO → «Wally» → ACTIVADO («Te escucho.») → comando → PROCESANDO
         → CONFIRMANDO si aplica → ejecución → RESPONDIENDO → ESPERANDO «Wally»

Por voz, un comando sin activación se ignora. El campo de texto no exige «Wally».
Confirmaciones de sí/no expiran a los 60 s. Toda acción pasa por app.services.

Usuario de la plataforma

En el MVP solo el auxiliar de enfermería opera la plataforma. En cada acción indica
quién la realizó (instrumentador, anestesiólogo, cirujano u otro tipo configurable).
La trazabilidad guarda ambos: usuario_registro y rol_responsable.

Insumos y equipos





Equipos (torre, electrobisturí, monitor…): se verifican en tecnovigilancia (pendiente /
verificado / con falla). No tienen cantidades.



Insumos (gasas, suturas…): el auxiliar elige cuáles incluir; los demás se pueden excluir.
Empiezan en 0; luego se registra la cantidad inicial. El backend calcula
disponible = inicial − utilizado.



Alerta preventiva (R9) al quedar exactamente 2 (1 = estado crítico):
«Alerta preventiva: quedan 2 unidades de gasas.»



Alerta de agotamiento (R6) al llegar a 0: «Alerta: gasas agotadas.»



Estructura

backend/app/
  models.py           tablas SQLModel (equipos, insumos, anestesias, hitos, trazabilidad…)
  catalogos.py        checklist, catálogos, tipos de persona y reglas R1–R10
  services.py         lógica de negocio única + registrar_accion
  config_services.py  reglas, tipos de persona, catálogos y procedimientos
  rules.py            motor de alertas (supresión, modal, se resuelven solas)
  agent/              Wally: activación, intérprete, confirmaciones
  stats.py            KPIs
  seed.py             demo + 20 históricas
  main.py             endpoints FastAPI
frontend/src/
  App.jsx             auxiliar como usuario, polling, modal global de alertas
  useVoz.js           Web Speech API continua (es-CO)
  components/         Quirofano, AgenteVoz, AlertaModal, NuevaCirugia, Gestion, Trazabilidad



Notas





El reconocimiento de voz de Chrome necesita internet; sin red, use el campo de texto o las frases.



Los registros no se borran: las correcciones guardan el valor anterior y el nuevo.



Las confirmaciones y la ventana de activación de Wally viven en memoria del servidor.

