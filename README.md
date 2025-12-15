# Proyecto: Reconocimiento de Voz y Reproducción de Audio - Mini Contador

Este proyecto es un **ejemplo minimalista** del sistema de reconocimiento de voz y reproducción de audio que se utiliza en proyectos más grandes como 
"ROBERTA" en su version 5.0 (corre en raspberry o en PC).  
Aquí aprenderás a controlar un **contador numérico** con comandos de voz y reproducir sonidos de confirmación.

El objetivo es mostrar **cómo integrar Vosk (reconocimiento de voz), Tkinter (interfaz) y SimpleAudio (reproducción de WAV)** de manera simple y clara.

---

## 🎯 Funcionalidad principal

- Reconoce comandos de voz:
  - `"sumar"` → incrementa el contador.
  - `"restar"` → decrementa el contador.
  - `"borrar"` o `"reset"` → reinicia el contador a 0.
- Muestra el contador en pantalla de forma **muy grande**, legible desde lejos.
- Reproduce audios WAV asociados a cada comando.
- Permite probar la integración de voz + GUI + audio antes de usarlo en proyectos más grandes.

---

## 📂 Estructura de archivos

```plaintext
/contador-voz/
│
├─ main.py        # Código principal
├─ models/        # Modelos de Vosk (ej. vosk-model-small-es-0.42)
├─ audios/        # Audios WAV: sumar.wav, restar.wav, borrar.wav
└─ README.md      # Este archivo
```

---

## 🛠 Instalación paso a paso (Windows, CMD / PowerShell)

### 1️⃣ Instalar Python

1. Ir a [https://www.python.org/downloads/](https://www.python.org/downloads/)  
2. Descargar la versión recomendada (ej. Python 3.11).  
3. Durante la instalación, **marcar la opción "Add Python to PATH"** antes de hacer clic en "Install".  
4. Abrir **CMD** o **PowerShell** y escribir:

```bash
python --version
```

Si aparece la versión de Python, todo está correcto.

---

### 2️⃣ Crear carpeta del proyecto

1. Crear una carpeta llamada `contador-voz`.  
2. Dentro, crear subcarpetas: `models` y `audios`.  
3. Descargar el modelo de Vosk español pequeño (`vosk-model-small-es-0.42`) y descomprimirlo dentro de `models`.

---

### 3️⃣ Instalar dependencias de Python

Abrir CMD o PowerShell y ejecutar:

```bash
pip install vosk
pip install pyaudio
pip install simpleaudio
pip install gtts
```

**Notas importantes:**

- En Windows, PyAudio puede requerir paquetes precompilados si da error. Descargalos de [https://www.lfd.uci.edu/~gohlke/pythonlibs/#pyaudio](https://www.lfd.uci.edu/~gohlke/pythonlibs/#pyaudio).  
- Vosk necesita el modelo dentro de `models`.  
- gTTS se usa para generar audios desde texto.

---

### 4️⃣ Selección de la voz de Windows para generar los audios

Windows tiene voces TTS integradas. Para elegir una voz:

1. Presiona **Win + S** y escribe `Configuración de voz` o `Text to Speech`.  
2. Abre "Configuración de Voz" o "Narrador".  
3. Verás una lista de voces disponibles, como "Microsoft David", "Microsoft Zira" u otras.  
4. Copia el nombre exacto de la voz que quieras usar para generar tus audios.

---

### 5️⃣ Generar audios WAV

En este proyecto usamos la voz **Sabina (español MX)** de Windows para los audios de confirmación de los comandos del contador.

1. Crear un archivo `generar_audios.py` en la carpeta del proyecto:

```python
# generar_audios.py
# Genera archivos WAV de TTS con la voz Sabina (español MX) para el mini contador

import pyttsx3
import os

# Carpeta donde se guardarán los audios
OUTPUT_FOLDER = "audios"
os.makedirs(OUTPUT_FOLDER, exist_ok=True)

# Inicializar pyttsx3
engine = pyttsx3.init()

# Seleccionar la voz Sabina (español MX)
voices = engine.getProperty('voices')
for v in voices:
    if "SABINA" in v.id.upper():
        engine.setProperty('voice', v.id)
        break
else:
    print("¡No se encontró la voz Sabina! Revisá tus voces instaladas.")
    exit(1)

# Ajustes de velocidad y volumen
engine.setProperty('rate', 130)   # velocidad más lenta para claridad
engine.setProperty('volume', 1.0) # volumen máximo (0.0 a 1.0)

# Lista de frases y nombres de archivo
audios = [
    ("sumar", "sumar.wav"),
    ("restar", "restar.wav"),
    ("borrar", "borrar.wav")
]

# Generar audios
for frase, archivo in audios:
    path = os.path.join(OUTPUT_FOLDER, archivo)
    print(f"Generando: {archivo} ...")
    engine.save_to_file(frase, path)

engine.runAndWait()
print("✅ Todos los audios fueron generados en la carpeta 'audios/'")
```

2. Ejecutar desde CMD o PowerShell:

```bash
python generar_audios.py
```

3. Verás en la carpeta `audios/` los archivos WAV listos para usar en el mini contador: `sumar.wav`, `restar.wav` y `borrar.wav`.

> Este método asegura que los audios tengan **la misma voz y estilo** que queremos, con claridad y volumen adecuado para probar los comandos del ejemplo.

---

Con esto, tu mini proyecto queda completamente funcional, con generación de audios realista y lista para probar el reconocimiento de voz y la reproducción de WAV en paralelo.

---

## 🖥 Código principal (main.py)

```python
import tkinter as tk
import threading
import json
import pyaudio
from vosk import Model, KaldiRecognizer
import simpleaudio as sa
import os

# ================= CONFIG =================
MODEL_PATH = "models/vosk-model-small-es-0.42"
CHUNK = 8000
SAMPLE_RATE = 16000
DEVICE_INDEX = None
AUDIO_FOLDER = "audios"

contador = 0

# ================= FUNCIONES DE AUDIO =================
def reproducir_audio(nombre):
    path = os.path.join(AUDIO_FOLDER, nombre)
    if os.path.exists(path):
        try:
            wave_obj = sa.WaveObject.from_wave_file(path)
            wave_obj.play()
        except Exception as e:
            print(f"[AUDIO] Error: {e}")

# ================= FUNCIONES DE VOZ =================
def normalizar_texto(texto):
    return texto.lower().strip()

def detectar_accion(texto):
    t = normalizar_texto(texto)
    if "sumar" in t: return "SUMAR"
    if "restar" in t: return "RESTAR"
    if "borrar" in t or "reset" in t: return "BORRAR"
    return None

# ================= INTERFAZ =================
class ContadorApp:
    def __init__(self, root):
        self.root = root
        root.title("Contador por Voz")
        self.label = tk.Label(root, text=str(contador), font=("Arial", 150, "bold"), fg="red")
        self.label.pack(expand=True, fill="both")

    def actualizar(self):
        self.label.config(text=str(contador))
        self.root.after(200, self.actualizar)

# ================= HILO DE VOZ =================
def hilo_voz():
    global contador
    model = Model(MODEL_PATH)
    rec = KaldiRecognizer(model, SAMPLE_RATE)
    p = pyaudio.PyAudio()
    stream = p.open(format=pyaudio.paInt16, channels=1, rate=SAMPLE_RATE,
                    input=True, frames_per_buffer=CHUNK,
                    input_device_index=DEVICE_INDEX)
    stream.start_stream()
    print("[VOZ] Iniciando reconocimiento de voz...")
    while True:
        data = stream.read(CHUNK, exception_on_overflow=False)
        if rec.AcceptWaveform(data):
            res = json.loads(rec.Result())
            texto = res.get("text","")
            accion = detectar_accion(texto)
            if accion == "SUMAR":
                contador += 1
                reproducir_audio("sumar.wav")
                print(f"[VOZ] Sumar -> {contador}")
            elif accion == "RESTAR":
                contador -= 1
                reproducir_audio("restar.wav")
                print(f"[VOZ] Restar -> {contador}")
            elif accion == "BORRAR":
                contador = 0
                reproducir_audio("borrar.wav")
                print("[VOZ] Borrar -> 0")

# ================= MAIN =================
if __name__ == "__main__":
    root = tk.Tk()
    app = ContadorApp(root)
    app.actualizar()
    hilo = threading.Thread(target=hilo_voz, daemon=True)
    hilo.start()
    root.mainloop()
```

---

## 📝 Uso

1. Abrir **CMD o PowerShell** en la carpeta del proyecto.  
2. Ejecutar:

```bash
python main.py
```

3. Verás un contador grande en pantalla.  
4. Dicta los comandos `"sumar"`, `"restar"` o `"borrar"`.  
5. El número cambiará en pantalla y reproducirá un audio de confirmación.

---

## 📌 Consideraciones

- Mantener el micrófono limpio y sin ruido para mejorar reconocimiento.  
- Los comandos deben ser claros y cortos.  
- Se puede ampliar fácilmente para otros comandos o mostrar varios contadores en pantalla.  
- Funciona en Windows, Linux y MacOS siempre que las dependencias estén instaladas.  
- Si querés que los audios suenen con la voz de Windows, usar **pyttsx3** y seleccionar la voz correcta como se indicó.
