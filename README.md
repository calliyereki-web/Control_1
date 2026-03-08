# Control_1: Sistema de Control de Motor DC 
> **"Los tibios aprobarán"**

Este proyecto contiene el firmware para el control de posición y velocidad de un motor de Corriente Directa (DC) utilizando el microcontrolador **PIC18F4550**. El sistema integra lectura de encoder de cuadratura, control por PWM y comunicación serial bidireccional mediante un protocolo de paquetes robusto.

---

## 🛠️ Cosas por Corregir / Pendientes
- [ ] **Sincronización:** Coordinar el envío y recibimiento de datos con la comunicación serial para evitar colisiones de buffer.
- [ ] **Lazo Cerrado:** Implementar formalmente el algoritmo PID (actualmente en fase de pruebas proporcionales).
- [ ] **Frenado Dinámico:** Optimizar la transición entre los estados `FW` (Forward) y `BW` (Backward) para evitar picos de corriente (Back EMF).

---

## ⚙️ Funcionamiento Detallado

### 1. Generación de PWM y Potencia
El control de la energía entregada al motor se realiza mediante el módulo **CCP1** del PIC.
* **Frecuencia de Trabajo:** Configurada a **20 kHz** (mediante `PR2 = 240`) para asegurar un movimiento fluido y fuera del rango auditivo humano, eliminando el silbido característico en bajas frecuencias.
* **Resolución:** El sistema cuenta con una resolución de **10 bits**, permitiendo valores de Duty Cycle entre **0 y 964**.
* **Driver (Puente H):** El sentido de giro se gestiona mediante los pines `D0` (FW) y `D1` (BW).



### 2. Lectura del Encoder (Interrupciones RB)
Para conocer la posición real del eje en tiempo real, se utiliza la interrupción por cambio de estado en el Puerto B (`#int_rb`).
* **Detección de Cuadratura:** Se monitorean los pines **B6 y B7**. El firmware aplica una operación XOR entre el estado actual y el anterior para determinar el sentido de giro.
* **Filtro de Ruido:** El algoritmo descarta "saltos" inválidos (donde el cambio de fase no es lógico) para evitar errores de conteo producidos por ruido electromagnético.



### 3. Comunicación Serial con Manejo de Errores
El sistema opera a **115200 baudios**. Se ha implementado un manejador de excepciones para garantizar que la comunicación no se detenga:
* **Manejo de Overrun (OERR):** Si el buffer de recepción se satura, el firmware reinicia el módulo `CREN` automáticamente.
* **Protocolo de Paquetes:** La recepción se sincroniza con el byte de encabezado `0xCA`. Se utilizan **apuntadores** (`int8* pinstruccion`) para procesar los paquetes de 4 bytes sin bloquear la ejecución del programa principal.

### 4. Base de Tiempo Determinista (Timer 0)
El **Timer 0** actúa como el reloj maestro para el muestreo de datos.
* **Frecuencia de Muestreo:** Configurada a **1 kHz** (Carga `61000`).
* **Importancia:** Esta base de tiempo constante permite que el cálculo del error y la impresión de resultados sean deterministas, un requisito técnico fundamental para la futura implementación de un control PID estable.



---

## 🚀 Especificaciones Técnicas

| Parámetro | Valor |
| :--- | :--- |
| **Microcontrolador** | PIC18F4550 |
| **Frecuencia de Oscilación** | 48 MHz (HSPLL) |
| **Baudrate** | 115200 bps |
| **Resolución Encoder** | 500 CPR (2000 ppr en cuadratura) |
| **Rango Duty Cycle** | 0 - 964 |

---

## 📁 Estructura del Paquete Serial
El sistema espera y envía paquetes con la siguiente estructura:

| Byte | Función | Descripción |
| :--- | :--- | :--- |
| 0 | Encabezado | Siempre `0xCA` |
| 1 | Instrucción | Dirección de giro (`0xF0`, `0xBA`, etc.) |
| 2 | Data High | Parte alta del Duty Cycle (16 bits) |
| 3 | Data Low | Parte baja del Duty Cycle (16 bits) |
