# Asignación de Pacientes a Doctores y Salas con SMA + CP-SAT + LLM

Este repositorio contiene un prototipo de planificación hospitalaria que integra:

* **Sistemas Multi-Agente (SMA)**
* **Programación Restrictiva (CP-SAT con OR-Tools)**
* **Un modelo de lenguaje (LLM, Gemini)** para explicar la solución en lenguaje natural

El objetivo es asignar pacientes a doctores y salas en distintos horarios, respetando las restricciones de recursos y priorizando a los pacientes más urgentes.

---

## 1. Descripción general

El sistema simula un entorno hospitalario ambulatorio en un solo día, con:

* **Pacientes**:

  * Especialidad requerida
  * Nivel de urgencia (1–3)
  * Horarios posibles / preferidos
* **Doctores**:

  * Especialidades que atienden
  * Disponibilidad por slots de tiempo
  * Capacidad máxima diaria (número de pacientes)
* **Salas**:

  * Tipo de sala
  * Disponibilidad por slots
  * Capacidad por slot (consultorios estándar: 1 paciente)

La solución se compone de tres capas:

1. **Capa Multi-Agente**

   * Agentes: `PatientAgent`, `DoctorAgent`, `RoomAgent` y `CoordinatorAgent`.
   * Interactúan mediante mensajes en un entorno compartido (`HospitalEnv`).
   * Los pacientes solicitan atención; doctores y salas reportan disponibilidad; el coordinador centraliza la información.

2. **Capa de Programación Restrictiva (CP-SAT, OR-Tools)**

   * El coordinador construye un modelo con variables binarias `x[p,d,s,t]`.
   * Se imponen restricciones de compatibilidad, disponibilidad y capacidad.
   * La función objetivo prioriza urgencias altas y, en menor medida, horarios más tempranos.

3. **Capa de Explicación con LLM (Gemini)**

   * A partir de la solución, se genera un resumen estructurado de las asignaciones.
   * Este resumen se envía a la API de Gemini, que devuelve:

     * `explanation`: explicación corta
     * `strengths`: puntos fuertes del plan
     * `limitations`: posibles mejoras

---

## 2. Requisitos

* **Python** 3.9+ (recomendado)
* Librerías principales:

  * `ortools`
  * `google-generativeai`
* Entorno sugerido:

  * Jupyter Notebook **o**
  * VS Code / entorno Python estándar

Instalación de dependencias (ejemplo):

```bash
pip install ortools google-generativeai
```

---

## 3. Estructura del repositorio

> Ajusta los nombres si usas otros archivos.

```text
.
├── notebook.ipynb          # Notebook principal con todo el experimento
├── src/
│   ├── agents.py           # Definición de agentes y entorno HospitalEnv
│   ├── cp_model.py         # Construcción y resolución del modelo CP-SAT
│   ├── run_simulation.py   # Script para lanzar la simulación (opcional)
│   └── llm_explainer.py    # Llamada a Gemini y formateo de la explicación
└── README.md
```

También puedes tener todo junto en un único notebook y usar este README solo como documentación conceptual.

---

## 4. Cómo ejecutar el prototipo

### 4.1. Configurar la API de Gemini

**No** subas tu API key al repositorio.

Opciones sugeridas:

1. **Variable de entorno**

```bash
export GEMINI_API_KEY="TU_API_KEY_AQUI"
```

Luego, en Python:

```python
import os
GEMINI_KEY = os.getenv("GEMINI_API_KEY")
```

2. **Archivo local (no versionado)**

* Crea un archivo de texto, por ejemplo: `api_key_gemini.txt`
* Añádelo a `.gitignore`
* Léelo desde el código:

```python
import os
GEMINI_KEY = os.getenv("GEMINI_API_KEY")
if not GEMINI_KEY:
    key_path = r"ruta/local/api_key_gemini.txt"
    if os.path.exists(key_path):
        with open(key_path, "r") as f:
            GEMINI_KEY = f.read().strip()
```

### 4.2. Ejecutar la simulación

En el notebook:

1. Ejecutar las celdas que definen:

   * `HospitalEnv`
   * Los agentes (`PatientAgent`, `DoctorAgent`, `RoomAgent`, `CoordinatorAgent`)
   * El modelo CP-SAT
2. Instanciar los pacientes, doctores y salas de prueba.
3. Ejecutar el ciclo de fases:

   * `announce` → pacientes, doctores, salas
   * `collect` y `optimize` → coordinador
   * `dispatch` → coordinador
   * `react` → pacientes
4. Inspeccionar:

   * `env.assignments`
   * Estado final de cada paciente
   * Horarios por doctor

Ejemplo de salida de asignaciones:

P1 (urg 3, Cirugía)       -> D3 en S1 a las 08:00
P2 (urg 2, Traumatología) -> D2 en S1 a las 10:00
P3 (urg 1, Cardiología)   -> D1 en S2 a las 08:00
P4 (urg 3, Neurología)    -> D3 en S2 a las 13:00
P5 (urg 1, Traumatología) -> D2 en S2 a las 09:00
P6 (urg 2, Cirugía)       -> D1 en S2 a las 11:00

### 4.3. Ejecutar la explicación con LLM

1. Asegúrate de tener disponible `solution_summary` (lista de diccionarios con las asignaciones).
2. Ejecuta la celda / script que llama a Gemini (por ejemplo, `llm_explainer.py`).

La salida se imprime en forma legible:

=== Explicación de la solución ===
...

=== Puntos fuertes de la planificación ===

* ...

=== Limitaciones / Posibles mejoras ===

* ...

---

## 5. Detalles del modelo CP-SAT

* **Variable principal**: `x[p,d,s,t]` ∈ {0,1}
* **Restricciones principales**:

  * Compatibilidad especialidad paciente–doctor
  * Disponibilidad de doctor, sala y paciente en el slot `t`
  * Un paciente se atiende a lo sumo una vez
  * Un doctor atiende a lo sumo un paciente por slot
  * Capacidad diaria máxima por doctor
  * Capacidad por sala y slot
* **Función objetivo (versión implementada)**:

  * Maximizar la suma de `score * x[p,d,s,t]` para todas las combinaciones posibles
  * `score = alpha * urgencia_p - beta * t`
  * `alpha` grande (prioriza urgencia), `beta` pequeña (prefiere horarios más tempranos)

---

## 6. Posibles extensiones

Algunas ideas para seguir ampliando el proyecto:

* Modelar atenciones que duren varios slots consecutivos.
* Evitar huecos grandes en las agendas de los doctores mediante nuevas restricciones.
* Balancear mejor la carga entre salas.
* Introducir replanificación dinámica cuando entran nuevos pacientes o se cancelan citas.
* Usar el LLM como interfaz en lenguaje natural para ajustar parámetros:

  * “prioriza Neurología en la tarde”
  * “minimiza los huecos en la agenda del doctor D3”.

---

## 7. Licencia

Me olvide de poner cuando cree el repo

---

## 8. Autoría

Trabajo desarrollado como parte de un curso de Tópicos de Inteligencia Artificial.

* Autora: *panconlocrito*.
