# Especificación de API del Modo Nano
## Contrato de Diseño y Protocolo de Comunicación de Sistema Operativo para la Versión Pocket (Windows 12)

El **Modo Nano** es un estado operativo híbrido gestionado conjuntamente por el gestor de ventanas del sistema operativo y la aplicación activa. A diferencia de una ventana tradicional que simplemente se redimensiona, el Modo Nano exige un **contrato de interfaz adaptativa** para garantizar que las aplicaciones sigan siendo 100% funcionales en dimensiones ultra-compactas, previniendo recortes de contenido y barras de scroll rotas.

---

## 1. Breakpoints y Reglas de Container Queries

Para desligar el diseño adaptativo del tamaño total de la pantalla del usuario y asociarlo directamente al espacio asignado a la ventana, la interfaz de la aplicación debe estructurarse obligatoriamente usando **Container Queries** de CSS.

### A. Breakpoints Oficiales de Modo Nano
Se definen dos breakpoints estándar para optimizar la legibilidad en pantallas de alta densidad (DPI):

| Breakpoint | Dimensiones de Ventana (Límites) | Comportamiento del Layout |
| :--- | :--- | :--- |
| **Nano Pocket** | Ancho $\le$ `250px` / Alto $\le$ `350px` | Colapso total de sidebars, tabs y menús superiores en un menú hamburguesa flotante. Contenido clave a 1 columna. |
| **Micro Ambient** | Ancho $\le$ `180px` / Alto $\le$ `240px` | Modo ultra-condensado. Solo se pintan los controles primarios y el canvas de datos esenciales (ej. portada y play/pausa). |

### B. Ejemplo de Implementación CSS
El elemento raíz de la aplicación debe definirse como un contenedor de consultas (`container-type: inline-size` o `normal`):

```css
/* Definición del contenedor de la aplicación */
.app-root {
  container-name: window;
  container-type: size;
  width: 100%;
  height: 100%;
}

/* Aplicación del Breakpoint Nano Pocket */
@container window (max-width: 250px) and (max-height: 350px) {
  .sidebar, .main-navigation {
    display: none; /* Ocultar barras laterales de navegación */
  }
  .app-content {
    padding: 8px;
    font-size: 0.85rem;
  }
  .pocket-action-bar {
    display: flex; /* Mostrar barra de herramientas compacta inferior */
  }
}
```

---

## 2. Comportamiento Fallback para Aplicaciones Sin Soporte (Legacy)

Si una aplicación comercial no implementa el contrato adaptativo del Modo Nano (es decir, no responde al breakpoint inferior de `250px`), el gestor de ventanas del sistema operativo aplicará uno de los siguientes comportamientos de compatibilidad:

1.  **Escalado por Transformación (Default Fallback):**
    El gestor de ventanas mantiene la aplicación renderizada en un búfer interno a tamaño normal (`1024x768` o similar) y proyecta una versión miniaturizada en la ventana flotante del Modo Nano aplicando una transformación de escala por hardware (`transform: scale(0.25)`). La ventana Pocket interceptará los eventos de ratón y los remapeará por coordenadas a la resolución interna para mantener la interactividad básica.
2.  **Redirección a Minimización Clásica:**
    Si la aplicación se encuentra en una lista negra de incompatibilidad visual (ej. herramientas de diseño CAD, hojas de cálculo complejas sin modo lectura), la llamada a conmutar Modo Nano provocará una minimización clásica hacia la barra de tareas, indicándole al usuario mediante un diálogo nativo: *"Esta aplicación no soporta la vista compacta Nano"*.

---

## 3. Protocolo de Comunicación (IPC) entre el OS y la Aplicación

Para coordinar la transición estética de la decoración de la ventana (los botones de Minimizar/Maximizar) y el renderizado interno de la app, se establece un canal de comunicación de bajo nivel a través de eventos IPC (Inter-Process Communication):

```
┌─────────────────────────────────┐               ┌─────────────────────────────────┐
│     Gestor de Ventanas (OS)     │               │    Ventana de la App (Client)   │
└────────────────┬────────────────┘               └────────────────┬────────────────┘
                 │                                                 │
                 │ ─── 1. IPC Send: "os:window-state-change" ────> │ (Detecta state: 'nano')
                 │      Payload: { state: 'nano', bounds }         │ (Ajusta CSS e interfaz)
                 │                                                 │
                 │ <── 2. IPC Reply: "app:window-state-ready" ──── │ (Confirma redibujado listo)
                 │                                                 │
                 │ ─── 3. Aplica animación física de morphing ───> │ (Estrella azul y Squircle)
```

### A. Estructura de Mensajes IPC (JSON)

#### 1. Del Sistema Operativo hacia la Aplicación:
Cuando el usuario hace clic derecho en el botón verde de Maximizar, el OS envía este evento antes de redimensionar la ventana para permitir que la app prepare su DOM:

```json
{
  "event": "os:window-state-change",
  "payload": {
    "targetState": "nano",
    "bounds": {
      "width": 220,
      "height": 320
    },
    "dpiRatio": 2.0
  }
}
```

#### 2. De la Aplicación hacia el Sistema Operativo:
Una vez que el motor de renderizado de la app (Blink, Gecko, etc.) procesa el nuevo CSS adaptativo y está listo para mostrar la ventana compacta, envía una confirmación para que el OS realice el morphing del marco de la ventana de forma sincronizada:

```json
{
  "event": "app:window-state-ready",
  "payload": {
    "currentState": "nano",
    "success": true
  }
}
```

---

## 4. Requerimientos de Ventana Nativa en Modo Nano

Para consolidar la usabilidad del Modo Nano a nivel de escritorio, el gestor de ventanas nativo debe aplicar los siguientes atributos en caliente a la ventana cliente:

*   **Always-on-Top (Siempre al Frente):** Habilitado. La app pocket no debe quedar sepultada por ventanas de trabajo completo.
*   **Shadow Drop (Sombras Activas):** Reemplazar las sombras densas de ventana por un sutil borde translúcido coloreado en concordancia con el tema azul (`rgba(8, 145, 178, 0.4)`).
*   **Interacciones Táctiles y de Arrastre:** Toda la superficie de la ventana en Modo Nano (excepto los controles interactivos internos) actuará como barra de arrastre del sistema, permitiendo reubicar la ventana pocket de forma fluida por cualquier borde de la pantalla.
