# Taller de GNURadio: Procesamiento y Análisis de Señales

Bienvenido a la documentación interactiva del taller extra de GNURadio enfocado en la recepción y procesamiento de señales mediante hardware RTL-SDR. Este repositorio sirve como índice y guía técnica para comprender los flujos de trabajo (*flowgraphs*) diseñados.

A continuación, se detallan los distintos módulos, explicando los fundamentos teóricos de procesamiento digital de señales (DSP) y la justificación técnica de cada bloque utilizado.

---

## 1. Modulación de Amplitud (AM)

Este módulo simula la transmisión de una señal de audio (mensaje) montada sobre una onda de radiofrecuencia (portadora). El objetivo es comprender cómo variamos la amplitud de la portadora en función del mensaje sin perder información.

### Fundamentos Teóricos

La Modulación de Amplitud (AM) estándar se define matemáticamente en el dominio del tiempo como:

$$s(t) = [A_c + m(t)] \cdot \cos(2\pi f_c t)$$

Donde:
* $A_c$: Amplitud de la componente de corriente continua (DC) o portadora pura.
* $m(t)$: Señal moduladora (el mensaje/audio).
* $f_c$: Frecuencia de la portadora.

Para evitar la **sobremodulación** (que causa inversión de fase y distorsión en la recepción), la envolvente $[A_c + m(t)]$ debe ser siempre positiva. Esto se controla mediante el índice de modulación.

### Conceptos Clave del Diagrama

#### A. Frecuencia de Muestreo (Sample Rate)
En el mundo digital (DSP), no tenemos señales continuas, sino puntos discretos. La variable `samp_rate` es crítica. Según el **Teorema de Nyquist-Shannon**, para reconstruir una señal sin errores, la frecuencia de muestreo $f_s$ debe ser:

$$f_s > 2 \cdot f_{max}$$

Si configuramos un `samp_rate` muy bajo en el simulador, verás el efecto de **Aliasing**: la señal se "dobla" y aparecen frecuencias fantasmas que no existen en la realidad.

#### B. Acondicionamiento de la Señal
En GNURadio, para lograr una AM estándar (y no una DSB-SC o Doble Banda Lateral con Portadora Suprimida), debemos manipular el mensaje antes de multiplicarlo.

1.  **Multiply Const ($k_a$):** Controla el índice de modulación. Si $k_a > 1$, la señal cruza por cero y se produce sobremodulación.
2.  **Add Const (+1):** Este bloque es fundamental. Sumar una constante añade un **offset DC**. Esto "levanta" la señal del mensaje sobre el eje X, asegurando que, al multiplicarla por la portadora, la envolvente no se invierta.

### Desglose de Bloques (Flowgraph)

| Bloque | Función Técnica | ¿Por qué lo usamos? |
| :--- | :--- | :--- |
| **Variable (`samp_rate`)** | Define la velocidad del reloj del sistema. | Sincroniza todos los bloques. Debe ser suficiente para cubrir el ancho de banda total ($f_c + f_m$). |
| **Signal Source (Mensaje)** | Genera una onda coseno de baja frecuencia ($f_m \approx 4\text{kHz}$). | Simula la voz o música que queremos transmitir. |
| **Multiply Const** | Escala la amplitud del mensaje por un factor $k_a$. | Permite ajustar la profundidad de modulación. |
| **Add Const** | Suma un valor escalar (generalmente 1) a la señal. | Inserta la portadora en la señal base. Matemáticamente: $1 + k_a \cdot \cos(\dots)$. |
| **Signal Source (Portadora)** | Genera una onda de alta frecuencia ($f_c \approx 20\text{kHz}$ o más). | Es el "vehículo" de transporte que viaja por el aire. |
| **Multiply** | Multiplica señal acondicionada $\times$ portadora. | Realiza la modulación propiamente dicha (traslación en frecuencia). |
| **QT GUI Range** | Deslizadores gráficos. | Permite modificar $f_c$, $f_m$ y $k_a$ en tiempo real para observar los cambios en el espectro. |

### ➡️ Simulación Interactiva

Accede al simulador web para experimentar con el Teorema de Nyquist, ver el efecto de la sobremodulación en el dominio del tiempo y analizar el espectro de frecuencia resultante.

[**Ver Simulación: Modulación AM**](https://dra-clea.github.io/TallerGRadioRTL/modam.html)

---
## 2. Fundamentos: Generación y Superposición
Antes de procesar señales de radio complejas, es fundamental entender cómo se crean y combinan las señales en GNURadio. Este módulo explica el **Principio de Superposición** y el manejo de archivos.

### Conceptos Clave
* **Superposición:** En un medio lineal, la señal resultante en cualquier punto es la suma algebraica de las señales individuales: $y(t) = y_1(t) + y_2(t)$. Esto permite simular interferencias constructivas y destructivas.
* **Throttle (Cuello de Botella):** En una simulación que no involucra hardware físico (como una radio SDR o tarjeta de audio), el ordenador procesará los datos tan rápido como pueda la CPU (millones de muestras por segundo). El bloque *Throttle* frena el flujo para igualar el `samp_rate` definido, permitiendo visualizar la señal en tiempo real sin congelar la interfaz.

### Desglose de Bloques

| Bloque | Función Técnica | ¿Por qué lo usamos? |
| :--- | :--- | :--- |
| **Signal Source** | Generador de forma de onda (Seno, Coseno, etc.). | Crea señales de prueba puras con frecuencia y amplitud conocidas. |
| **Add** | Sumador aritmético punto a punto. | Simula la mezcla de señales en el aire o en un cable. |
| **Throttle** | Limitador de flujo de datos. | **Obligatorio** en simulaciones puras para evitar el uso del 100% de CPU. |
| **File Sink** | Escritura en disco (Binary float 32-bit). | Guarda la señal "cruda" para ser procesada posteriormente o analizada en otro software (Matlab/Python). |

### ➡️ Simulación Interactiva
Prueba a sumar dos frecuencias cercanas (ej. 1000Hz y 1100Hz) para ver el fenómeno de "batido" en el tiempo.

[**Ver Simulación: Generador de Ondas**](https://dra-clea.github.io/TallerGRadioRTL/dam-1.html)

---

## 3. Receptor AM (Parte 1): Traslación y Decimación
Este módulo inicia la cadena de recepción. Partimos de una grabación de radio real capturada con un SDR (`am_usrp710.dat`) que contiene mucho ruido y señales en frecuencias que no nos interesan.

### Fundamentos Teóricos
* **Mezcla de Frecuencia (Traslación):** La señal de interés (voz) está en una frecuencia alta (ej. 80kHz relativos). Para trabajar con ella, debemos "bajarla" a banda base (0 Hz). Esto se logra multiplicando la señal de entrada $x(t)$ por un oscilador local complejo $e^{-j\omega_c t}$.
    $$x_{base}(t) = x_{rf}(t) \cdot e^{-j 2\pi f_c t}$$
* **Decimación:** La grabación original tiene una tasa de muestreo ($f_s$) de **256 kHz**. Procesar tantos datos es ineficiente para una señal de audio que solo necesita ~4 kHz de ancho de banda. La decimación consiste en quedarse con 1 de cada $N$ muestras.
    * Entrada: 256 kHz.
    * Decimación $N=4$.
    * Salida: $256 / 4 = 64 \text{ kHz}$.

### Desglose de Bloques

| Bloque | Función Técnica | ¿Por qué lo usamos? |
| :--- | :--- | :--- |
| **File Source** | Lectura de archivo binario complejo. | Inyecta los datos IQ grabados previamente con el SDR. |
| **Signal Source** | Generador de portadora negativa (-80kHz). | Crea la señal necesaria para cancelar la frecuencia de la emisora original y moverla a 0Hz. |
| **Multiply** | Mezclador (Mixer). | Ejecuta la traslación de frecuencia matemática. |
| **Low Pass Filter** | Filtro Pasa Bajos + Decimador. | **Doble función:** 1. Elimina frecuencias fuera del canal de audio (anti-aliasing). 2. Reduce la velocidad de muestreo (Divide por 4). |

### ➡️ Simulación Interactiva
Visualiza cómo la señal se mueve en el espectro al cambiar la frecuencia del oscilador local y observa el efecto del filtrado.

[**Ver Simulación: Demodulación y Decimación**](https://dra-clea.github.io/TallerGRadioRTL/dam-2.html)

---

## 4. Receptor AM (Parte 2): Audio y Resampleo
Etapa final del receptor. Convertimos la señal de radiofrecuencia (ya filtrada y centrada) en sonido audible por los altavoces del ordenador.

### Fundamentos Teóricos
* **Demodulación AM (Detector de Envolvente):** En AM, la información está en la amplitud. Matemáticamente, si tenemos una señal compleja $z(t) = I + jQ$, la amplitud instantánea se obtiene calculando su magnitud:
    $$|z(t)| = \sqrt{I^2 + Q^2}$$
* **Eliminación de DC:** La señal demodulada incluye la portadora (offset DC). Si enviamos esto al altavoz, el cono se desplazaría hacia afuera constantemente, calentando la bobina y distorsionando el audio. Usamos un filtro para eliminar la componente de 0Hz.
* **Resampleo Racional:** Las tarjetas de sonido estándar funcionan a **48 kHz** (o 44.1 kHz). Nosotros venimos de **64 kHz** (del paso anterior). Necesitamos convertir matemáticamente la tasa de muestreo:
    $$64.000 \times \frac{3}{4} = 48.000 \text{ Hz}$$

### Desglose de Bloques

| Bloque | Función Técnica | ¿Por qué lo usamos? |
| :--- | :--- | :--- |
| **Complex to Mag** | Cálculo de módulo $|z|$. | Extrae la envolvente de la señal (el audio + portadora). Es el demodulador AM propiamente dicho. |
| **DC Blocker** | Filtro de rechazo de banda (Notch en 0Hz). | Elimina la componente continua (DC) para centrar la señal de audio en 0. |
| **Multiply Const** | Ganancia escalar. | Funciona como control de **Volumen**. |
| **Rational Resampler** | Interpolación (3) y Decimación (4). | Adapta la velocidad de datos de 64k a 48k para que la tarjeta de sonido pueda reproducirla correctamente. |
| **Audio Sink** | Salida a Hardware. | Envía el flujo de datos final a los altavoces del sistema. |

### ➡️ Simulación Interactiva
Experimenta con el bloque DC Blocker para ver cómo afecta a la forma de onda y entiende visualmente la diferencia entre la señal de radio (espectro) y la de audio (tiempo).

[**Ver Simulación: Receptor AM Completo**](https://dra-clea.github.io/TallerGRadioRTL/dam-3.html)

---
## 5. Banda Lateral Única (SSB): Eficiencia Espectral

La Modulación de Amplitud (AM) es ineficiente: gasta potencia en la portadora (que no lleva información) y transmite la misma información duplicada en dos bandas laterales. **SSB (Single Sideband)** elimina la portadora y una de las bandas, concentrando toda la potencia en la voz y ocupando la mitad de ancho de banda.

Para lograr esto digitalmente sin filtros analógicos de cristal costosos, utilizamos el **Método de Fase** y la **Transformada de Hilbert**.

### 5.1. El Método de Hilbert (Generación de Señal Analítica)

En el mundo real, las señales son reales ($V(t)$). En el espectro, esto implica simetría: frecuencias positivas y negativas. Para eliminar una banda lateral matemáticamente, necesitamos convertir nuestra señal real en una **Señal Analítica** (Compleja).

La Transformada de Hilbert genera una copia de la señal original desfasada exactamente $90^\circ$ ($\frac{\pi}{2}$ radianes).

* **Señal Real ($I$):** $m(t)$
* **Señal Imaginaria ($Q$):** $\hat{m}(t)$ (Transformada de Hilbert)

$$z(t) = m(t) + j\cdot\hat{m}(t)$$

Al combinar estas dos, eliminamos las frecuencias negativas del espectro, lo que es el primer paso para la SSB.

[**➡️ Simulación 1: Visualizar el Desfase de Hilbert**](https://dra-clea.github.io/TallerGRadioRTL/ssb-1.html)
*(Observa en el osciloscopio cómo la señal azul siempre persigue a la naranja con un retardo exacto de 1/4 de ciclo)*.

---

### 5.2. El Modulador de Fase (Hartley)

Una vez tenemos la señal compleja de audio ($I$ y $Q$) y una portadora compleja, podemos seleccionar matemáticamente qué banda queremos transmitir (Superior o Inferior) sin usar filtros, simplemente sumando o restando fases.

**Matemática del Modulador:**
Para obtener la Banda Lateral Superior (USB) o Inferior (LSB), realizamos la mezcla compleja:

* **USB (Upper Sideband):** $\text{Re}\{ (m(t) + j\hat{m}(t)) \cdot e^{j\omega_c t} \} = m(t)\cos(\omega_c t) - \hat{m}(t)\sin(\omega_c t)$
* **LSB (Lower Sideband):** $m(t)\cos(\omega_c t) + \hat{m}(t)\sin(\omega_c t)$

**Explicación del Diagrama:**
1.  **Audio Source:** Genera la voz.
2.  **Hilbert:** Crea los componentes $I$ (Real) y $Q$ (Imaginaría/Desfasada).
3.  **Multiply:** Multiplica $I_{audio} \times I_{portadora}$ y $Q_{audio} \times Q_{portadora}$.
4.  **Add/Sub:**
    * Si **Restamos** los resultados $\rightarrow$ Las frecuencias bajas se cancelan $\rightarrow$ Queda **USB**.
    * Si **Sumamos** los resultados $\rightarrow$ Las frecuencias altas se cancelan $\rightarrow$ Queda **LSB**.

[**➡️ Simulación 2: Modulador SSB Interactivo**](https://dra-clea.github.io/TallerGRadioRTL/ssb-2.html)
*(Mueve el slider de frecuencia de audio y observa cómo el pico en el espectro salta de un lado a otro de la portadora fantasma)*.

---

### 5.3. Sistema Completo: Entradas Reales y Archivos IQ

Este diagrama muestra cómo preparar un transmisor SSB real. En GNU Radio, podemos alternar entre un tono puro, un micrófono (tu voz en tiempo real) o un archivo `.wav`.

**Puntos Críticos del Flowgraph:**
* **Selector de Entrada:** En GNU Radio, si conectas múltiples fuentes a un punto, **solo una debe estar habilitada** a la vez (Block Enabled/Disabled), o las muestras se mezclarán causando errores de reloj.
* **Generación de Archivo IQ (`.dat`):** Para transmitir esta señal con un hardware SDR (como HackRF o PlutoSDR), necesitamos un archivo complejo.
    * La salida del modulador SSB es **Flotante (Real)**.
    * Para guardarla en formato IQ, usamos un bloque `Float to Complex`. Conectamos la señal SSB a la entrada **Real** y una fuente constante de **0** a la entrada **Imaginaria**.
    * Esto crea un archivo compatible con transmisores SDR.

[**➡️ Simulación 3: Sistema Transmisor con Micrófono/Wav**](https://dra-clea.github.io/TallerGRadioRTL/ssb-3.html)

---

### 5.4. Recepción 1: "Frequency Xlating FIR Filter"

Al recibir señales de radio con un SDR, a menudo capturamos un ancho de banda grande (ej. 256 kHz) a una frecuencia muy alta. Sin embargo, nuestra tarjeta de audio solo soporta 48 kHz.

El bloque **Frequency Xlating FIR Filter** es una "navaja suiza" que realiza tres operaciones costosas en un solo paso eficiente:

1.  **Traslación de Frecuencia (Mixing):** Multiplica la señal de entrada por $e^{-j\omega_{c}t}$ para mover la estación de radio que queremos escuchar (que está en, por ejemplo, 51.5 kHz) al centro (0 Hz).
2.  **Filtrado Pasa Bajos (FIR):** Aplica un filtro digital para eliminar todas las demás estaciones y el ruido, dejando solo el ancho de banda de la voz (3 kHz).
3.  **Decimación:** Reduce la tasa de muestreo. Si entramos con 256k y decimamos por 8:
    $$256.000 / 8 = 32.000 \text{ Hz}$$
    Ahora la señal es compatible con el sistema de audio.

[**➡️ Simulación 4: Sintonizador y Decimador**](https://dra-clea.github.io/TallerGRadioRTL/ssb-4.html)
*(Usa el slider de "Tuning" para arrastrar la señal roja hacia el centro del filtro verde. Solo escucharás algo cuando la señal esté dentro del filtro)*.

---

### 5.5. Recepción 2: Demodulador Weaver

El método Weaver es una técnica avanzada para demodular SSB que evita algunos problemas de fase del método de Hartley.

**El Problema:** Una señal SSB de voz no tiene portadora y su energía no está centrada en 0 Hz, sino desplazada (la voz va de 300 Hz a 3000 Hz). Si la bajamos directamente a 0 Hz, la parte baja del espectro se solapará con la negativa.

**La Solución Weaver:**
1.  **Conversión I/Q:** Separamos la señal en ramas Real e Imaginaria.
2.  **Osciladores Locales (1.5 kHz):** En lugar de bajar la señal exactamente a 0 Hz, utilizamos osciladores seno y coseno a **1.5 kHz** (la mitad del ancho de banda de la voz: $3kHz / 2$).
3.  **Resultado:** Esto centra perfectamente el espectro de voz en la banda base de audio, permitiendo escucharla con claridad.

Este diagrama final conecta todo: entrada decimada $\rightarrow$ separación I/Q $\rightarrow$ mezcla Weaver $\rightarrow$ Audio Sink.

[**➡️ Simulación 5: Demodulador Weaver Completo**](https://dra-clea.github.io/TallerGRadioRTL/ssb-5.html)
*(Experimenta con los botones LSB/USB para ver cómo la matemática invierte el espectro y recupera el audio original)*.

---
## 6. Modulación Digital BPSK (Binary Phase Shift Keying)

Hasta ahora hemos trabajado con audio analógico. Ahora entramos en el mundo digital. La **BPSK** es la forma más robusta de modulación digital: transmitimos bits (0 y 1) invirtiendo la fase de la portadora $180^\circ$.

$$s(t) = d(t) \cdot A_c \cos(2\pi f_c t)$$

Donde $d(t)$ vale $+1$ para el bit '1' y $-1$ para el bit '0'.

---

### 6.1. Transmisor Básico: El problema de la Onda Cuadrada
En el mundo digital teórico, los bits son cuadrados perfectos (cambian de 0 a 1 instantáneamente). Sin embargo, en el mundo físico (radiofrecuencia), **transmitir ondas cuadradas es ilegal e ineficiente**.

#### ¿Por qué necesitamos Interpolar y Filtrar?
1.  **Ancho de Banda Infinito:** Matemáticamente (Fourier), un cambio brusco de voltaje (una esquina de un cuadrado) requiere infinitas frecuencias armónicas. Esto ensuciaría todo el espectro radioeléctrico.
2.  **Interpolación:** Una fuente de datos digital genera un punto matemático cada cierto tiempo. Para convertirlo en una "onda" que ocupe tiempo, debemos repetir ese punto muchas veces (interpolar).
3.  **Filtrado (Pulse Shaping):** Debemos "lijar" las esquinas del pulso cuadrado para hacerlo suave. Esto limita el ancho de banda y concentra la energía donde la necesitamos.

#### Desglose de Bloques

| Bloque | Función Técnica | ¿Por qué lo usamos? |
| :--- | :--- | :--- |
| **GLFSR Source** | Generador de bits pseudo-aleatorios. | Simula un flujo de datos (internet, texto, imagen) para pruebas. |
| **Interpolating FIR Filter** | Repetición de muestras. | Convierte 1 bit en 100 muestras iguales. Transforma un punto en un "ladrillo" cuadrado. |
| **Low Pass Filter** | Suavizado. | Elimina las frecuencias altas de las esquinas del cuadrado. Convierte el "ladrillo" en una "montaña" suave. |
| **Multiply** | Modulador BPSK. | Sube la señal banda base a la frecuencia portadora ($f_c$). |

### ➡️ Simulación Interactiva
Observa la transformación de los bits: de puntos invisibles a cuadrados rojos, y finalmente a la onda verde suave lista para transmisión.

[**Ver Simulación: Transmisor BPSK Básico**](https://dra-clea.github.io/TallerGRadioRTL/bpsk-1.html)

---

### 6.2. Sincronización y Retardo (Delay)
Al procesar señales digitales, los filtros (como el Low Pass Filter) introducen un retraso matemático inevitable llamado **Retardo de Grupo**. La señal "entra" al filtro y tarda unos milisegundos en "salir" procesada.

Para comparar la señal original (bits cuadrados) con la señal filtrada (onda suave) en una gráfica superpuesta, necesitamos **retrasar artificialmente** la señal original para que ambas se alineen en el tiempo.

* **Bloque Delay:** No modifica la señal, solo la retiene en memoria $N$ muestras para compensar la latencia del filtro paralelo.

[**➡️ Simulación: Compensación de Retardo**](https://dra-clea.github.io/TallerGRadioRTL/bpsk-2.html)

---

### 6.3. El Estándar Profesional: Root Raised Cosine (RRC)
En sistemas de telecomunicaciones reales (4G, Wi-Fi, Satélite), no usamos un filtro Pasa Bajos genérico. Usamos el filtro **Coseno Alzado (Raised Cosine)**.

Este filtro está diseñado matemáticamente para cumplir el **Criterio de Nyquist para ISI nula**. Esto significa que, aunque la onda oscile, garantiza que cruzará por cero exactamente en el momento en que ocurre el siguiente símbolo, evitando que un bit interfiera con el siguiente (Inter-Symbol Interference).

En la práctica, dividimos este filtro en dos partes iguales:
1.  **Root Raised Cosine (TX):** En el transmisor.
2.  **Root Raised Cosine (RX):** En el receptor.
$$RRC_{TX} \cdot RRC_{RX} = \text{Raised Cosine Completo}$$

Esto maximiza la relación señal-ruido (SNR) al funcionar como un *filtro adaptado*.

[**➡️ Simulación: Filtro RRC**](https://dra-clea.github.io/TallerGRadioRTL/bpsk-3.html)

---

### 6.4. Receptor y Diagrama de Ojo
Para recuperar los datos, realizamos el proceso inverso: bajamos la señal de frecuencia y analizamos su calidad.

#### Conceptos del Receptor
* **Demodulación (-Freq):** Para bajar una señal que está en $+16\text{kHz}$ a banda base ($0\text{Hz}$), la multiplicamos por una señal compleja de $-16\text{kHz}$.
* **Diagrama de Ojo (Eye Diagram):** Es la herramienta fundamental para analizar la calidad de una señal digital. Superpone todos los bits recibidos en una sola ventana de tiempo.
    * **Ojo Abierto:** Las líneas se separan claramente arriba y abajo. Indica buena calidad y fácil distinción entre '1' y '0'.
    * **Ojo Cerrado:** Las líneas se cruzan en el centro debido al ruido o mala sincronización. Indica errores de bit (BER).

#### Desglose de Bloques

| Bloque | Función Técnica | ¿Por qué lo usamos? |
| :--- | :--- | :--- |
| **Signal Source (-16k)** | Oscilador Local inverso. | Genera la frecuencia negativa para cancelar la portadora y bajar a banda base. |
| **Complex to Real** | Descarte de Q. | En BPSK, la información solo vive en el eje Real (I). Descartamos la parte Imaginaria (Q) que solo contiene ruido. |
| **QT GUI Eye Diagram** | Osciloscopio de persistencia. | Visualiza la "apertura" del ojo para evaluar si la comunicación es fiable frente al ruido. |

### ➡️ Simulación Interactiva
Prueba a subir el nivel de ruido en la simulación y observa cómo el "ojo" se cierra, haciendo imposible distinguir los datos.

[**Ver Simulación: Receptor y Diagrama de Ojo**](https://dra-clea.github.io/TallerGRadioRTL/bpsk-4.html)

---
## 7. FSK (Frequency Shift Keying) con VCO

En este módulo construiremos un módem completo para **RTTY (Radioteletipo)**. A diferencia del BPSK donde cambiábamos la fase, aquí cambiaremos la frecuencia.

* **Bit 0 (Space):** Se transmite a **2125 Hz**.
* **Bit 1 (Mark):** Se transmite a **2295 Hz**.

El reto es: ¿Cómo obligamos a un oscilador a generar estas frecuencias exactas usando solo matemáticas?

---

### 7.1. Preparación de Datos: De Bits a Flotantes
Antes de modular, necesitamos acondicionar los datos. Un ordenador genera bits instantáneos, pero una señal de radio necesita "tiempo" para existir.

#### Desglose de Bloques
* **Random Source & Unpack:** Generamos bytes (0-255) y los descomponemos en bits individuales (0 o 1).
* **Repeat (Interpolación):** Este es clave. Un bit matemático no tiene duración. Para transmitirlo a una velocidad de audio (Sample Rate 48kHz), debemos repetirlo muchas veces.
    * Si queremos una velocidad lenta (típica de RTTY), repetimos cada bit **1056 veces**.
    * Esto crea un "pulso" largo que dura lo suficiente para ser escuchado.
* **UChar to Float:** Convertimos los datos de tipo Byte (morado) a Float (naranja) porque el VCO funciona con números decimales (Voltaje).

[**➡️ Simulación 1: Tubería de Datos**](https://dra-clea.github.io/TallerGRadioRTL/fsk-1.html)

---

### 7.2. El Corazón Matemático: VCO y Voltajes
Aquí es donde ocurre la magia. Usamos un **VCO (Voltage Controlled Oscillator)**. Este bloque no recibe una frecuencia, recibe un **Voltaje** y lo convierte en frecuencia.

#### La Constante del VCO (K)
En GNU Radio, la sensibilidad del VCO se define en radianes. En tu diagrama es **15.708k**.
Para saber cuántos Hercios genera por cada Voltio, aplicamos:

$$K_{VCO} (Hz/V) = \frac{\text{Sensitivity}}{2\pi} = \frac{15708}{6.283} \approx 2500 \text{ Hz/Volt}$$

Esto significa: **Por cada 1 Voltio que entra, el VCO emite 2500 Hz.** 
#### Cálculo de los Voltajes de Control
Necesitamos generar 2125 Hz y 2295 Hz. Usando la constante $K = 2500$:

1.  **Voltaje para "Space" (0):**
    $$V_{space} = \frac{2125 \text{ Hz}}{2500 \text{ Hz/V}} = \mathbf{0.850 \text{ V}}$$
    Este es nuestro "suelo" o base.

2.  **Voltaje para "Mark" (1):**
    $$V_{mark} = \frac{2295 \text{ Hz}}{2500 \text{ Hz/V}} = \mathbf{0.918 \text{ V}}$$

3.  **La Diferencia (Delta):**
    $$0.918 \text{ V} - 0.850 \text{ V} = \mathbf{0.068 \text{ V}}$$

#### Implementación en Bloques
* **Add Const (0.850):** Sumamos 0.850 a todo. Si entra un '0', sale 0.850V (2125 Hz).
* **Multiply Const (0.068):** Si entra un '1', lo multiplicamos por 0.068. Al sumarle el 0.850 base, obtenemos $0.068 + 0.850 = 0.918V$ (2295 Hz).

[**➡️ Simulación 2: Matemáticas VCO interactivas**](https://dra-clea.github.io/TallerGRadioRTL/fsk-2.html)

---

### 7.3. Receptor 1: Traslación de Frecuencia (Xlating)
En el receptor, queremos aislar estas dos frecuencias. Para facilitar la demodulación, centramos la señal en 0 Hz (Banda Base).

* **Frecuencia Central:** El punto medio entre Mark y Space es:
    $$\frac{2125 + 2295}{2} = \mathbf{2210 \text{ Hz (2.21k)}}$$
* **Xlating FIR Filter:** Este bloque sintoniza en 2.21k y lo mueve a 0 Hz.
    * El **Space (2125)** se mueve a $-85 \text{ Hz}$.
    * El **Mark (2295)** se mueve a $+85 \text{ Hz}$.

[**➡️ Simulación 3: Filtro Traductor**](https://dra-clea.github.io/TallerGRadioRTL/fsk-3.html)

---

### 7.4. Receptor 2: Demodulación por Cuadratura
Una vez centrada la señal, usamos un **Quadrature Demod**. Este bloque calcula la velocidad de cambio de fase (frecuencia instantánea).

* **Gain (Ganancia):** La salida cruda del demodulador es pequeña. Para que nos de exactamente "1.0" o "-1.0" y poder decidir si es un bit 1 o 0, aplicamos una ganancia basada en la desviación de frecuencia ($Deviation = 85 \text{ Hz}$).
    $$Gain = \frac{SampRate}{2\pi \cdot Deviation}$$
* **Binary Slicer:** Toma la decisión final. Si la señal es positiva ($>0$), saca un byte **1**. Si es negativa ($<0$), saca un byte **0**.
* **Verificación:** Usamos un bloque **Subtract** restando la señal recibida de la original (retardada por el bloque **Delay** para compensar el tiempo de viaje). Si la línea resultante es plana en 0, la transmisión es perfecta.

[**➡️ Simulación 4: Demodulador y Verificación de Errores**](https://dra-clea.github.io/TallerGRadioRTL/fsk-4.html)

---
## 8. Conceptos Avanzados y Herramientas Extra

Esta sección aborda conceptos transversales esenciales para el diseño de flujos en GNU Radio, desde la manipulación correcta de tipos de datos hasta la creación de bloques personalizados y el manejo de audio.

---

### 8.1. Matemáticas vs. Conversión de Tipos
Un error común es confundir bloques que **transforman** valores con bloques que **desempaquetan** tipos de datos.

* **Transcendental (Math):** Son bloques que aplican funciones trigonométricas o matemáticas complejas. Transforman el valor numérico.
    * Ejemplo: Entrada `0` $\rightarrow$ Bloque Cos $\rightarrow$ Salida `1.0`.
    * Analogía: Una licuadora que transforma los ingredientes en algo nuevo.
* **Complex to Float (Type Conversion):** No realiza cálculos matemáticos complejos, simplemente separa las partes de un número.
    * Ejemplo: Entrada `3 + 4j` $\rightarrow$ Salida A `3`, Salida B `4`.
    * Analogía: Separar un sándwich en pan y jamón.

[**➡️ Simulación: Diferencia Visual Interactiva**](https://dra-clea.github.io/TallerGRadioRTL/Extra1.html)

---

### 8.2. Cadena de Transmisión Digital: De Bits a Símbolos
Para transmitir un archivo digital por radio, no podemos enviar "bytes" directamente. Necesitamos una cadena de traducción específica.

1.  **File Source:** Entrega Bytes (paquetes de 8 bits, ej: `00001101`).
2.  **Packed to Unpacked:** Rompe el byte en trozos más pequeños (chunks). Para BPSK, necesitamos bits individuales.
    * Entrada: `13` (Decimal)
    * Salida: `0, 0, 0, 0, 1, 1, 0, 1`.
3.  **Chunks to Symbols:** Es el cartógrafo. Toma esos bits y los coloca en un mapa de coordenadas complejas (Constelación).
    * Entrada: `0` $\rightarrow$ Salida: `-1` (Izquierda).
    * Entrada: `1` $\rightarrow$ Salida: `+1` (Derecha).

[**➡️ Simulación: Visualizador de Mapeo de Bits**](https://dra-clea.github.io/TallerGRadioRTL/Extra2.html)

---

### 8.3. Bloques Jerárquicos y Diagrama de Ojo
A medida que los diagramas crecen, se vuelven ilegibles. GNU Radio permite crear **Bloques Jerárquicos**: diagramas completos que se comportan como un solo bloque dentro de otro flujo ("Bloques dentro de bloques").

Además, este módulo explica el funcionamiento interno del **Diagrama de Ojo**. No es magia; es una técnica de visualización que consiste en trocear la señal recibida y superponer todos los trazos.
* **Ojo Abierto:** Señal limpia, el receptor distingue claramente entre 0 y 1.
* **Ojo Cerrado:** Ruido o Jitter excesivo, la señal se cruza en el centro provocando errores de decisión.

[**➡️ Simulación: Construyendo un Diagrama de Ojo**](https://dra-clea.github.io/TallerGRadioRTL/Extra3.html)

---

### 8.4. Audio y Archivos: WAV vs DAT
El manejo de archivos y audio tiene trampas comunes relacionadas con la velocidad de muestreo.

* **Archivos .WAV:** Tienen una "cabecera" con información. El bloque sabe automáticamente a qué velocidad reproducirlos.
* **Archivos .DAT (Raw Float):** Son solo listas de números sin metadatos. El ordenador intentará leerlos tan rápido como permita la CPU.
    * **¡Importante!** Si usas un archivo `.dat` en una simulación sin hardware de radio, debes usar un bloque **Throttle**. Este bloque frena el flujo de datos para igualar el `Sample Rate` deseado, evitando que la simulación se ejecute instantáneamente y cuelgue la interfaz.

[**➡️ Simulación: Laboratorio de Audio AM**](https://dra-clea.github.io/TallerGRadioRTL/extra4.html)

---
