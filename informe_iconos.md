# Informe de Análisis Técnico y de Diseño UI/UX
**Proyecto:** Nueva Propuesta de Interfaz (Windows 12) - Botones de Control de Ventana  
**Archivo de Origen:** [botones.html](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html)

---

## 1. Concepto y Estética Visual

El archivo propone un panel de control de ventanas con una estética **premium y moderna**, alineada con las directrices visuales de interfaces de próxima generación. Combina dos corrientes de diseño actuales:

*   **Glassmorphism (Efecto Vidrio):** El contenedor principal ([.glass-container](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L45)) utiliza `backdrop-filter: blur(16px)` junto con un color blanco semi-transparente y un sutil borde claro para simular vidrio esmerilado que se integra dinámicamente con el fondo.
*   **Neumorphism Modernizado (Embossed & Glossy):** En lugar de las sombras planas tradicionales, la clase base de los botones ([.btn-surface](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L58)) aplica:
    *   Una sombra interior superior blanca (`inset 0 2px 2px rgba(255, 255, 255, 0.6)`) para simular un borde iluminado en 3D.
    *   Un gradiente diagonal interno (`::after`) que genera un reflejo de luz.
    *   Una sombra proyectada difusa coloreada según el tema del botón (`var(--shadow-r)`) que le da profundidad flotante.
    *   Una física elástica mediante transiciones con curvas Bézier personalizadas (`cubic-bezier(0.175, 0.885, 0.32, 1.275)`) que provocan un efecto de "rebote" orgánico cuando el botón recupera su estado.

---

## 2. El Motor de Animación: Morfosis de SVG (SVG Path Morphing)

El aspecto más avanzado y preciso de estos iconos es la **animación fluida de cambio de forma (morphing)**. 

### ¿Cómo funciona técnicamente?
1.  **Topología Homogénea por Botón:** Para que un navegador pueda animar la transición de un vector SVG a otro sin distorsiones ni parpadeos, ambas formas en una transición deben compartir el mismo número de puntos de control (nodos Bézier).
2.  **Consistencia de Segmentos:** 
    *   Los estados del botón de **Maximizar/Nano** (`state-max`, `state-normal`, `state-nano`) emplean exactamente **16 comandos Bézier cúbicos (`C`)** en su path, lo que garantiza una transición elástica limpia.
    *   Los estados del botón de **Minimizar/Focus** (`state-line-h`, `state-line-v`, `state-eye`) emplean exactamente **4 comandos Bézier cúbicos (`C`)**, manteniendo la sencillez del morphing para este botón de forma independiente.
3.  **Transición CSS:** La propiedad CSS `transition: d 0.6s cubic-bezier(0.68, -0.6, 0.32, 1.6)` en la clase [.morph-path](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L170) se encarga de interpolar las coordenadas del atributo `d` del elemento `<path>` SVG de forma nativa a velocidad fluida cuando cambia la clase de estado del SVG contenedor.

---

## 3. Comportamiento y Estados de los Iconos

Los botones reaccionan a diferentes interacciones físicas (clics izquierdos y clics derechos) que a su vez mutan la estética del botón (su tema cromático) y la geometría del icono.

### A. Botón de Maximizar / Restaurar / Nano (Verde / Azul)
Es el botón cuadrado con el color base verde (`t-green`).

*   **Estado Inicial (Max):** Muestra un **Squircle** (cuadrado perfecto de bordes redondeados) utilizando la clase [.state-max](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L227).
*   **Clic Izquierdo (Toggle Max $\leftrightarrow$ Normal):**
    *   Transforma el Squircle exterior en un cuadrado más pequeño y concéntrico ([.state-normal](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L183)), que representa la acción clásica de "restaurar ventana" a su tamaño original.
*   **Clic Derecho (Modo Nano - Azul):**
    *   **Cambio de Tema:** Cambia el botón de verde (`t-green`) a un gradiente azul cian (`t-blue`), actualizando las sombras correspondientes.
    *   **Morfosis del Icono:** El path se transforma suavemente en una **estrella compleja de 16 puntas** ([.state-nano](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L188)). Esto activa la **versión Pocket** de la ventana: un tamaño ultra-compacto flotante que conserva intactas todas las funcionalidades operativas principales pero adaptando su layout de forma inteligente para no perder espacio ni usabilidad.
    *   *Nota:* Un nuevo clic izquierdo o derecho en este estado lo devuelve al estado normal o maximizado.

### B. Botón de Minimizar / Focus (Amarillo / Morado)
Es el botón alargado horizontalmente.

*   **Estado Inicial (Minimizar):** Muestra una barra horizontal redondeada ([.state-line-h](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L243)) en color ámbar/amarillo (`t-amber`).
*   **Clic Derecho (Modo Focus / Ojo - Morado):**
    *   **Cambio de Tema:** Transiciona del color amarillo al color morado/violeta (`t-purple`).
    *   **Morfosis del Icono:** El path del icono se abre en forma de **Ojo / Lente** ([.state-eye](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L193)) y se revela una pupila circular interior. Este modo activa la **atenuación del escritorio**: oscurece el fondo de pantalla y desenfoca todas las ventanas ajenas a la aplicación principal, permitiendo centrar el enfoque visual del usuario únicamente en la app activa.
    *   *Nota:* Un clic izquierdo o clic derecho adicional restaura el botón al color ámbar y a la línea de minimización original.

### C. Botón de Cerrar (Rojo)
Es el botón cuadrado con el color base rojo (`t-red`).

*   **Diseño:** A diferencia de los otros dos, este botón no cuenta con lógica de morfosis activa en Javascript.
*   **Implementación Técnica:** Para pintar la "X", utiliza una máscara SVG global ([#global-x-mask](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L214)). Esta máscara recorta un rectángulo de color oscuro semi-transparente (`fill="rgba(0,0,0,0.25)"`) con la forma exacta de la "X", logrando que el fondo del botón rojo se trasluzca a través del icono de cierre con una sombra sutil e integrada.

---

## 4. Descubribilidad y Accesibilidad Premium

Para subsanar la opacidad inherente del clic derecho en botones de control y cumplir con el estándar **WCAG 2.1 AA**, el sistema implementa:

1.  **Affordance en Hover:** Al posicionar el cursor sobre los botones en su estado base, se revela una miniatura semitransparente del icono secundario (ojo o estrella) en la esquina inferior derecha. Esto insinúa visualmente la existencia de un modo alternativo.
2.  **Indicador Permanente (Notch):** Un pequeño punto físico (`.dual-dot`) ubicado en el borde superior de los botones con funcionalidad dual alerta permanentemente al usuario de que el control posee comportamientos extendidos.
3.  **Accesibilidad por Teclado e Interactividad:**
    *   Los botones admiten foco mediante tabulación (`tabindex="0"`) y se marcan con un contorno de foco premium `:focus-visible`.
    *   Admiten accionamiento por teclado: `Space` / `Enter` simula el clic izquierdo normal, mientras que `Shift + Space` / `Shift + Enter` simula el clic derecho/acción especial.
    *   Los lectores de pantalla reciben `aria-label` dinámicos que se actualizan de forma continua según el estado actual de la ventana.

---

## 5. Análisis de la Lógica de Control (JavaScript)

El flujo de control se gestiona mediante tres funciones clave en el script:

1.  **Objeto [CONFIG](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L276):**
    Asocia cada tipo de interacción (`max` o `min`) con sus clases CSS iniciales, sus clases especiales de estado alternativo (nano y focus) y los nombres de las clases de los paths SVG.
2.  **Manejador [handleClick](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L306) y [handleRightClick](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L321):**
    Separan físicamente los clics del mouse de manera limpia. El clic izquierdo gestiona los estados de tamaño tradicional (Maximizar/Normal/Minimizar), mientras que el clic derecho conmuta los modos avanzados de estado mental (Focus/Nano).
3.  **Actualizador [updateAriaLabel](file:///c:/Desarrollos%20DEV%20daniel/nueva%20propuesta%20windows%2012/botones.html#L284):**
    Modifica las etiquetas semánticas de accesibilidad de manera dinámica tras cada cambio de estado para guiar a usuarios que empleen tecnologías de asistencia.
