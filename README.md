# REPOSITORIO-BRAZO-ROBOTICO-6-GDL-XK-LS5B

"https://drive.google.com/drive/folders/1eebz2k1js_J-q01hgVVoKYDfFnWV-fHu?usp=drive_link"


Esta sección complementa la documentación técnica de la planta flexible integrando los análisis de identificación de variables, modelado del lazo de control y desarrollo de software embebido para el brazo robótico de 6 GDL.  

Para poder visualizar el repositorio, descargar Archivos.html y ejecutar

---

## 📋 Tabla de Contenidos
- [1. Especificación del Hardware y Asignación de Pines](#1-especificación-del-hardware-y-asignación-de-pines)
  - [Actuadores y Drivers](#a-actuadores-y-drivers)
  - [Mapeo de I/O en Microcontrolador PIC16F1946](#b-mapeo-de-io-en-microcontrolador-pic16f1946)
- [2. Arquitectura de Control y Clasificación de Variables](#2-arquitectura-de-control-y-clasificación-de-variables)
  - [Lazo de Control (Lazo Abierto Actual)](#a-definición-del-lazo-de-control-lazo-abierto-actual)
  - [Matriz de Variables del Sistema](#b-matriz-de-variables-del-sistema)
- [3. Arquitectura del Software Embebido (MPLAB XC8)](#3-arquitectura-del-software-embebido-mplab-xc8)
  - [Estructura del Programa](#estructura-de-ejecución)
  - [Rutinas Principales](#a-mecanismo-de-control-y-rutinas-principales)
- [4. Trabajo Futuro: Implementación de Lazo Cerrado](#4-trabajo-futuro-implementación-de-lazo-cerrado)

---

## 1. Especificación del Hardware y Asignación de Pines

El control del brazo robótico se ejecuta mediante un microcontrolador **PIC16F1946** (14 KB Flash, 54 pines I/O, operando a 8 MHz con cristal HS), el cual gestiona la potencia a través de drivers de precisión, lee sensores de posición/límite y procesa la interfaz física del operador[cite: 1, 2].

### A. Actuadores y Drivers

El movimiento de las 6 articulaciones utiliza motores paso a paso bipolares distribuidos según requerimiento de torque[cite: 1, 2]:

* **Base (Eje 1), Hombro (Eje 2), Codo (Eje 3):** Motores paso a paso **NEMA 23** (Modelo `57J1854-828`, 2.8A/fase) comandados por drivers **2M542-N** (alimentación 24V DC, conmutación optoacoplada MOSFET).
* **Muñeca (Eje 4), Pinza (Eje 5):** Motores paso a paso **NEMA 17** (Modelo `42J1848-604`, 1.2A) controlados mediante drivers **2MA320 / 40KF**.
* **Giro de Muñeca (Eje 6):** Motor paso a paso **NEMA 16** accionado con reductor mecánico de velocidad.

### B. Mapeo de I/O en Microcontrolador PIC16F1946

#### Salidas Digitales (Drivers de Motores)
| Eje / Articulación | Pin PIC | Función / Etiqueta Firmware | Acción |
| :--- | :--- | :--- | :--- |
| **Eje 1 (Base)** | `RE0` / `RE1` | `zx` / `yx` | Izquierda / Derecha |
| **Eje 2 (Hombro)** | `RE2` / `RE3` | `xj1` / `ss1` | Abajo / Arriba |
| **Eje 3 (Codo)** | `RE4` / `RE5` | `ss2` / `xj2` | Arriba / Abajo |
| **Eje 4 (Muñeca)** | `RE6` / `RF0` | `ss3` / `xj3` | Arriba / Abajo |
| **Eje 5 (Pinza)** | `RF1` / `RF2` | `szzx` / `szyx` | Cerrar / Abrir[cite: 2] |
| **Eje 6 (Giro Pinza)** | `RF3` / `RF4` | `fs` / `zj` | Soltar / Agarrar[cite: 2] |

#### Entradas Digitales (Sensores de Posición / Límite)
* **Sensores de Efecto Hall NJK-5002C** (NPN NO, activo en 0V): Conectados a `RD0`–`RD4` para marcar la posición cero/referencia (*Homing*) en los Ejes 1 a 5, y en `RD5`/`RD6` para límites de giro[cite: 1, 2].
* **Finales de Carrera Mecánicos M102-011** (SPDT): Montados en la estructura de la pinza para tope físico de apertura/cierre.

#### Entradas de Control (Tablero del Operador)
* **Botones de Control Manual:** Asignados en los puertos `RA0`–`RA5`, `RB1`–`RB5` y `RC0`[cite: 2].
* **Control del Sistema:** 
  * `RC1`: Selector Modo Manual/Auto (`kaut`)[cite: 2]
  * `RC3`: Reset / Homing (`krest`)[cite: 2]
  * `RC4`: Inicio de Secuencia (`kstart`)[cite: 2]
  * `RC5`: Parada de Emergencia (`kstop`)[cite: 2]

---

## 2. Arquitectura de Control y Clasificación de Variables

### A. Definición del Lazo de Control (Lazo Abierto Actual)

Actualmente, la planta opera en un esquema de **Lazo Abierto** para la regulación de posicionamiento[cite: 1]:
1. **Proceso:** El PIC genera la Variable Manipulada $u(t)$ como señales discretas enviadas a los drivers de potencia[cite: 1]. Los drivers regulan la corriente en las bobinas de los motores NEMA para producir el desplazamiento angular $\theta_i$ (Variable Controlada $y(t)$)[cite: 1].
2. **Rol de la Senso-percepción:** Los sensores magnéticos y mecánicos no retroalimentan el error en tiempo real[cite: 1]. Se emplean exclusivamente como interrupciones de seguridad contra sobrecarrera y calibración del cero mecánico (*Homing*)[cite: 1].
3. **Perturbaciones ($d(t)$):** Cambios de masa en el efector final, rozamiento articular, backlash en reductores y posibles pérdidas de pasos bajo torque extremo[cite: 1].

### B. Matriz de Variables del Sistema

| Categoría | Símbolo | Componente / Señal | Descripción Técnica |
| :--- | :--- | :--- | :--- |
| **Referencia** | $r(t)$ | Firmware / Tablero | Vector de posiciones angulares objetivo $[\theta_{1d}, \dots, \theta_{6d}]^T$[cite: 1]. |
| **Manipulada (MV)** | $u(t)$ | PIC16F1946 $\to$ Drivers | Tren de pulsos PWM/digitales y direcciones $(0-5\text{V})$[cite: 1]. |
| **Controlada (PV)** | $y(t)$ | Ejes mecánicos | Ángulo real alcanzado por cada articulación $[\theta_1, \dots, \theta_6]^T$[cite: 1]. |
| **Protección** | $b(t)$ | NJK-5002C / M102-011 | Estado binario de carrera para interrupción del control[cite: 1]. |
| **Perturbación (DV)** | $d(t)$ | Carga / Fricción | Variaciones de inercia, torques externos y fluctuaciones de alimentación[cite: 1]. |

---

## 3. Arquitectura del Software Embebido (MPLAB XC8)

El software embebido fue desarrollado en lenguaje C ANSI bajo **MPLAB X IDE v6.x** y compilado con **XC8** (estándar C99)[cite: 2]. La arquitectura corresponde a una máquina de estados montada sobre un **Super-Loop**:

### Estructura de Ejecución

```c
void main(void) {
    IO_init();         // Configuración de TRISx y ANSELx (Desactivación analógica)
    PWM_init();        // Inicialización de CCP1/CCP2 con Timer2/Timer4 (Velocidad)

    while(1) {
        if (!kaut) {   // --- MODO MANUAL ---
            hand();    // Lectura de botonera y movimiento axis por axis
            check();   // Verificación de finales de carrera
            // Debounce delay (20ms)
        } 
        else {         // --- MODO AUTOMÁTICO ---
            keycan();  // Escaneo de banderas de control
            rest();    // Ejecución de rutina Homing (Calibración Cero)
            aut();     // Ejecución de 5 secuencias cinemáticas secuenciales
        }
    }
}

