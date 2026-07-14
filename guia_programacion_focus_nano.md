# Guía de Programación: Implementación de los Modos Focus y Nano

Este documento explica de forma breve qué son los modos **Focus** y **Nano**, y cómo deben programarse a nivel de arquitectura de software y sistema operativo tomando en cuenta tu propuesta interactiva.

---

## 1. Definición Conceptual de Programación

Desde el punto de vista del desarrollo de software, ambos modos se tratan como **estados de control del espacio de trabajo** que interactúan directamente con el gestor de ventanas del sistema operativo y con el motor de renderizado de la aplicación:

```
                  ┌─────────────── Gestor de Ventanas ───────────────┐
                  ▼                                                  ▼
      Modo FOCUS (Aislamiento)                           Modo NANO (Versión Pocket)
 ───────────────────────────────────                ───────────────────────────────────
 - OS: Crea una capa de oscurecimiento             - OS: Contenedor flotante ultracompacto,
   detrás de la ventana activa.                       mantiene interactividad de entrada.
 - OS: Desenfoca (blur) el resto de                 - UI: Versión "bolsillo" adaptativa;
   las ventanas del escritorio.                       mismo núcleo funcional de la app.
```

---

## 2. Estrategia de Programación (Paso a Paso)

### A. Gestión de Estados (State Management)
Se define un estado centralizado (usando un *hook*, *store* o *máquina de estados*) para controlar el modo visual activo:

```javascript
// Definición de estados posibles
const ViewModes = {
  NORMAL: 'normal',
  MAXIMIZED: 'maximized',
  FOCUS: 'focus',
  NANO: 'nano'
};

// Estado reactivo en la aplicación
let currentMode = ViewModes.NORMAL;
```

---

## 3. Programación del Modo Focus (Aislamiento del Escritorio)

El Modo Focus pone en relieve la ventana principal y desactiva el ruido del escritorio circundante.

### A. Implementación a Nivel de Sistema Operativo (OS / Electron / Tauri)
1.  **Capa de Atenuación (Dimming Overlay):** El proceso principal de la aplicación abre una ventana secundaria sin bordes, transparente, de pantalla completa (`fullscreen`), posicionada inmediatamente por detrás de la ventana principal de la app en la pila de renderizado (Z-Index).
2.  **Color y Filtro:** Esta ventana de fondo aplica un color negro semi-translúcido (ej. `rgba(0, 0, 0, 0.6)`) y un filtro CSS `backdrop-filter: blur(8px)` para desenfocar visualmente todas las ventanas ajenas del escritorio.
3.  **Captura de Foco:** La ventana de atenuación bloquea los clics accidentales en otras aplicaciones mientras el modo Focus esté activo.

```javascript
// Ejemplo conceptual en Electron (Process Main)
const { BrowserWindow } = require('electron');

function activateFocusMode(mainWindow) {
  // 1. Crear ventana de atenuación (Dimming Curtain)
  const overlay = new BrowserWindow({
    fullscreen: true,
    frame: false,
    transparent: true,
    alwaysOnTop: false, // Debe quedar justo debajo de mainWindow
    hasShadow: false
  });

  overlay.loadURL('data:text/html,<body style="background: rgba(0,0,0,0.65); backdrop-filter: blur(10px); margin:0;"></body>');
  
  // 2. Colocar la ventana principal al frente
  mainWindow.setAlwaysOnTop(true);
  mainWindow.focus();
  mainWindow.setAlwaysOnTop(false); // Quitar el bloqueo pero mantenerlo arriba
}
```

### B. Limitaciones Técnicas de `backdrop-filter` y Soluciones Nativas
> [!IMPORTANT]
> **Limitación de Entorno Web:** El uso de `backdrop-filter: blur(10px)` en CSS de Electron/Tauri solo es capaz de desenfocar el contenido web renderizado dentro del propio contexto de la ventana de la aplicación. **No puede desenfocar otras aplicaciones nativas del sistema operativo** (como MS Word o el Explorador de Archivos) ni el fondo real de escritorio del SO.
> 
> **Soluciones de Integración Nativa:** Para lograr el efecto de atenuación y desenfocado global del SO, el backend de la app debe invocar llamadas nativas del sistema:
> - **En Windows:** Utilizar la API de composición de ventanas DWM a través de módulos nativos de Node.js (ej. `node-gyp` con C++), ejecutando `SetWindowCompositionAttribute` para habilitar el material Acrylic o Mica en la ventana de atenuación, u ocultando temporalmente otras ventanas mediante llamadas de la API de Windows `ShowWindow(hWnd, SW_MINIMIZE)`.
> - **En macOS:** Emplear `NSVisualEffectView` configurando el material en modo `behindWindow` a nivel de Cocoa nativo para lograr la translucidez del sistema en la ventana de fondo.

---

## 4. Programación del Modo Nano (Versión Pocket Adaptable)

A diferencia de un widget estático que solo muestra información de lectura, el **Modo Nano** es una **versión Pocket (de bolsillo) funcional de la ventana**. No pierde capacidades, sino que las adapta dinámicamente al tamaño mínimo.

### A. Diseño UI Adaptativo mediante Container Queries (Frontend)
En lugar de usar un Media Query de pantalla completo, se utilizan **Container Queries** en CSS. Al reducir el tamaño de la ventana, la UI responde reorganizándose para seguir siendo 100% funcional.

```css
/* La UI se adapta al contenedor Pocket */
@container window (max-width: 250px) {
  .app-layout {
    grid-template-columns: 1fr; /* Pasa a flujo de columna simple */
  }
  .advanced-controls {
    display: flex;
    flex-wrap: wrap; /* Controles colapsados pero interactivos */
    scale: 0.85;     /* Escala táctil optimizada */
  }
}
```

### B. Control del Gestor de Ventanas (Backend)
Cuando se activa el Modo Nano, el backend ajusta las dimensiones mínimas y activa la persistencia sobre otras capas, sin destruir ni recargar la interfaz interna de la app:

```javascript
// Ejemplo de transición a Modo Pocket
function enableNanoMode(windowInstance) {
  // Cambiar a tamaño pocket manteniendo proporciones funcionales
  windowInstance.setSize(220, 320, true); // (width, height, animate)
  windowInstance.setMinimumSize(180, 260);
  windowInstance.setAlwaysOnTop(true);    // Mantener la versión pocket visible
}
```

---

## 5. Programación de Accesibilidad e Interacción por Teclado

Para cumplir con el estándar **WCAG 2.1 AA**, los controles de ventana del prototipo deben admitir navegación semántica y soporte completo de teclado.

### A. Estructura HTML y Atributos ARIA
Cada botón debe configurarse con rol de botón, índice de tabulación y etiquetas descriptivas dinámicas:

```html
<div role="button" 
     tabindex="0" 
     aria-label="Maximizar ventana (Clic derecho para Modo Nano)" 
     onkeydown="handleKeyDown(event, this, 'max')">
  ...
</div>
```

### B. Mapeo de Eventos en JavaScript
Debido a que el clic derecho no tiene un equivalente de teclado directo e intuitivo, se implementa una combinación de teclas modificadoras (en este caso, `Shift + Enter` o `Shift + Space`) para disparar la acción especial del clic derecho:

```javascript
// Controlador unificado de eventos de teclado
function handleKeyDown(event, element, type) {
  if (event.key === ' ' || event.key === 'Enter') {
    event.preventDefault(); // Evita el desplazamiento de la página
    
    if (event.shiftKey) {
      // Activa el modo especial (Focus o Nano) simulando Clic Derecho
      handleRightClick(event, element, type);
    } else {
      // Activa el comportamiento tradicional (Maximizar/Normalizar) simulando Clic Izquierdo
      handleClick(element, type);
    }
  }
}
```

### C. Actualización de Accesibilidad Semántica en Caliente
Es fundamental que la etiqueta `aria-label` cambie dinámicamente al transicionar entre estados para que los lectores de pantalla puedan anunciar correctamente la función disponible:

```javascript
function updateAriaLabel(element, type) {
  if (type === 'max') {
    const isNano = element.classList.contains('t-blue'); // Clase de estado Nano
    if (isNano) {
      element.setAttribute('aria-label', "Modo Nano activo (Clic izquierdo para volver a Maximizar)");
    } else {
      element.setAttribute('aria-label', "Maximizar ventana (Clic derecho para Modo Nano)");
    }
  }
}
```
