# Investigación y Análisis: Evaluación de la Propuesta de Ventanas Windows 12

## Valoración general

Esta propuesta resuelve un problema real. La triada Minimizar/Maximizar/Cerrar lleva congelada desde 1995 precisamente porque nadie se había molestado en cuestionar el axioma de fondo: que gestionar ventanas es un problema de *espacio físico*. Tu reencuadre — que el verdadero cuello de botella es el *estado cognitivo del usuario*, no los píxeles disponibles — es conceptualmente correcto y técnicamente ejecutable. Eso es raro en conceptos de UI especulativos, donde el diseño suele ser vistoso pero vacío de justificación funcional.

Dicho lo anterior, existe una brecha significativa entre la solidez del marco conceptual y la consistencia de la implementación. El análisis que sigue detalla exactamente dónde está esa brecha y qué tan fácil es cerrarla.

---

## El marco conceptual: la parte más fuerte

La decisión de anclar los nuevos modos a los botones existentes en vez de añadir botones nuevos es de alta inteligencia de diseño. Atar Focus al botón de Minimizar e invertir su lógica — "en lugar de ocultar esto, oculta todo lo demás" — no es solo un giro ingenioso: es una extensión coherente de la semántica del botón. El usuario ya tiene el modelo mental de "Minimizar = reducir el ruido", y Focus lleva eso al extremo lógico sin romper el paradigma.

Lo mismo aplica a Nano vinculado a Maximizar. El usuario ya entiende Maximizar como "quiero el 100% del contenido de esta app". Nano es simplemente el polo opuesto del mismo eje: "quiero el 5% más crítico". Son los extremos de un espectro, agrupados en un mismo control físico. Esa coherencia hace que la curva de aprendizaje sea casi nula una vez que el patrón se descubre — el problema es *que no se descubrirá*, pero lo abordo más adelante.

El mapa de tres estados cognitivos (Concentración profunda → Uso cotidiano → Monitoreo pasivo) tiene paralelos claros en la literatura de UX sobre *ambient display* y *flow state preservation*, y la propuesta no necesita citarlos para ser válida. Los fundamenta empíricamente con ejemplos concretos de comportamiento Pocket, lo cual es suficiente.

---

## Análisis de la máquina de estados: la inconsistencia crítica

El diagrama de arriba muestra el estado real de la implementación tal como está codificada en el HTML. Aquí emerge la primera discrepancia seria con la documentación.

La documentación describe el mecanismo de activación como **doble clic con temporizador de 250ms** ("Si se pulsa una segunda vez antes de 250ms, cancela el temporizador y ejecuta el ciclo con valor de clics = 2"). Sin embargo, el código implementado no usa ningún temporizador. En su lugar, usa dos manejadores completamente separados: `handleClick` para el clic izquierdo normal y `handleRightClick` (evento `contextmenu`) para los modos especiales. Son dos patrones de interacción fundamentalmente distintos con implicaciones UX opuestas.

El clic derecho como disparador de modos especiales es más seguro contra activaciones accidentales (nadie activa Nano sin querer al hacer clic derecho) pero es completamente opaco para el usuario — no hay ninguna convención en desktop que enseñe a hacer clic derecho sobre un botón de control de ventana para cambiar su comportamiento. El doble clic descrito en la documentación tiene mejor discoverability natural (los usuarios saben que doble clic en botones puede hacer cosas) pero requiere el temporizador para distinguirlo del clic simple.

Ninguno de los dos es incorrecto como decisión de diseño, pero el hecho de que el prototipo y la documentación describan sistemas distintos implica que el modelo de interacción no está consolidado. Esto hay que resolver antes de cualquier iteración de código.

---

## Análisis técnico: tres hallazgos específicos

**Primero: la topología SVG.** La documentación afirma que todos los estados del icono usan exactamente 16 comandos Bézier cúbicos (`C`) para garantizar morphing fluido. Esto es parcialmente falso. Contando los segmentos en el código:

- `state-max`: 16 segmentos C ✓
- `state-normal`: 16 segmentos C ✓
- `state-nano`: 16 segmentos C ✓
- `state-eye`: 4 segmentos C ✗
- `state-line-h`: 4 segmentos C ✗
- `state-line-v`: 4 segmentos C ✗

La buena noticia es que funcionalmente el morphing no explota, porque cada botón solo hace transiciones entre sus propios estados: el botón MIN siempre morfea entre `state-line-h` y `state-eye` (ambos de 4 segmentos), y el botón MAX siempre morfea entre sus tres estados de 16 segmentos. La topología es consistente *por botón*, aunque no globalmente. El problema es que la documentación crea un falso precedente técnico: si alguien añade un nuevo estado al botón MIN basándose en la narrativa de "16 segmentos", tendrá artifacts de morphing inevitables.

El segundo problema de la topología es más sutil: el `state-eye` define solo el contorno exterior del ojo (una curva almendra de 4 puntos), sin ningún path secundario para el iris o la pupila. A los 96×96 píxeles del botón en pantalla, el ojo vacío leerá como un óvalo con borde, no como un ojo. En SVG puedes resolver esto con un `<circle>` adicional superpuesto que no participe en el morphing, o expandir el path del ojo a uno compuesto usando subpaths (`M ... Z M ... Z` dentro del mismo atributo `d`).

**Segundo: la propiedad CSS `d` para animación.** La transición `transition: d 0.6s cubic-bezier(0.68, -0.6, 0.32, 1.6)` es elegante y los valores negativos del Bézier generan el overshoot elástico intencionado. El soporte de navegadores moderno es ahora sólido (Chrome ≥87, Safari ≥13.1, Firefox ≥112), pero vale la pena tener un fallback con `clip-path` o `transform` para entornos más conservadores como Electron con Chromium antiguo.

**Tercero: `backdrop-filter` en el Modo Focus.** La implementación propuesta usa `backdrop-filter: blur(10px)` en una overlay de `BrowserWindow` de Electron. Esto funciona — pero solo dentro del contexto de renderizado del proceso web. El filtro desenfocará el contenido web detrás de la ventana y el fondo de escritorio si el compositor del SO lo soporta, pero **no puede desenfocar otras ventanas nativas del sistema operativo** (Word, Finder, otra app de Electron). En Windows, el equivalente real requeriría `SetWindowCompositionAttribute` o DWM blur, que Electron no expone de forma directa. La descripción conceptual del Modo Focus es correcta; la implementación de referencia crea una expectativa de resultado que la API no puede cumplir completamente. Eso no invalida el concepto — significa que la nota técnica debe ser más precisa sobre qué se puede desenfocar y qué no.

---

## Las lagunas críticas

**Discoverability: el problema más grave.** No existe ningún affordance visual que indique que los modos especiales existen. Un usuario nuevo que vea los botones de esta propuesta no tiene manera de saber que el doble clic (o clic derecho) en el botón verde hace algo distinto al clic simple. macOS resuelve esto con símbolos que aparecen en hover (`×`, `−`, `+` o la flecha de pantalla completa). Esta propuesta necesita un mecanismo de indicación equivalente. La sugerencia más natural: en hover sobre el botón de Minimizar, mostrar brevemente un icono secundario en miniatura del ojo (tal vez en la esquina inferior derecha del botón), y en hover sobre Maximizar, mostrar un ícono de estrella. Esto se puede lograr con una segunda capa SVG de opacidad 0 que transiciona a 0.7 en hover.

**Accesibilidad: ausencia total.** Los botones no tienen `aria-label`, no hay navegación por teclado, y no hay indicadores de foco visible. La ironía de que un "Modo Focus" no tenga `:focus-visible` bien definido no pasa desapercibida. En un prototipo de concepto esto puede aceptarse como deuda, pero debe reconocerse explícitamente porque en cualquier implementación real de SO bloquea el cumplimiento de WCAG 2.1 AA.

**Dependencia del ecosistema para Nano Mode.** El Modo Nano en su formulación actual no es una feature del sistema operativo — es un *contrato de diseño* que cada aplicación debe implementar individualmente usando Container Queries. Sin una especificación pública de Nano Mode API (equivalente a los size classes de Apple o los window decorations hints de Wayland), el resultado real sería ventanas cortadas con scroll bars rotas, no layouts adaptativos. La propuesta necesita un documento de especificación de Nano Mode que defina al menos tres cosas: los breakpoints de Container Query mínimos requeridos, el comportamiento por defecto para apps que no lo implementen (¿minimizar clásico? ¿escalar con transform?), y el mecanismo de comunicación entre el gestor de ventanas y la app.

---

## Recomendaciones concretas

En orden de prioridad e impacto:

**Inmediatas (sin refactoring):** Unificar el modelo de interacción entre documentación y código — elige entre doble clic con timer o clic derecho, documenta solo eso, y asegúrate de que el prototipo lo refleje. Corregir la afirmación de "16 segmentos para todos los estados" en la documentación técnica. Añadir un `<circle>` de pupila superpuesto al `state-eye` para que el ícono sea legible.

**A corto plazo (refinamiento del prototipo):** Implementar el hover-reveal de modos secundarios — una segunda capa SVG de opacidad baja que muestre el ícono del estado alternativo (ojo para Min, estrella para Max) cuando el cursor esté sobre el botón. Esto resuelve el 80% del problema de discoverability con ~20 líneas de CSS adicionales. Añadir `aria-label` dinámico que cambie con el estado (`"Minimizar — doble clic para Focus"`, etc.) y un `tabindex` con manejo de `keydown`.

**A medio plazo (maduración del concepto):** Redactar la especificación de Nano Mode API como documento independiente — breakpoints, comportamiento fallback, señales de IPC entre OS y app. Especificar exactamente qué puede y qué no puede desenfocar el Modo Focus según el contexto de ejecución (Electron, Tauri nativo, implementación de SO). Considerar un tercer indicador visual por botón — un pequeño punto o notch en el borde — que persista siempre (no solo en hover) para señalar que hay capacidades adicionales disponibles, similar a los indicadores de notificación.

---

# Decisión Final de Diseño y Arquitectura

Para resolver las brechas identificadas, consolidar el proyecto y garantizar que el prototipo y la documentación estén en sintonía perfecta, se adopta la siguiente hoja de ruta y decisiones de diseño:

### 1. Modelo de Interacción Unificado: Clic Derecho (Contextmenu)
* **Decisión:** Mantendremos el **clic derecho** (`contextmenu`) como el disparador físico oficial para los modos especiales (*Focus* y *Nano*), ya que fue una petición explícita de diseño y previene activaciones accidentales causadas por fallos de temporización del doble clic.
* **Acción:** Actualizaremos todos los archivos de documentación para sustituir las referencias de "doble clic con timer" por "clic derecho / menú contextual".

### 2. Solución de Discoverability (Descubribilidad)
* **Affordance en Hover:** En hover sobre el botón de Minimizar o Maximizar, se revelará un mini-icono secundario semi-transparente (un ojo para Minimizar, una estrella para Maximizar) en la esquina del botón, invitando a la interacción secundaria.
* **Indicador Permanente (Notch):** Añadiremos un pequeño punto sutil en el borde de los botones con funcionalidad dual para indicar físicamente que hay una acción alternativa disponible.

### 3. Ajustes y Correcciones Técnicas de SVG
* **Pupila en State-Eye:** Agregaremos un círculo central `<circle>` dentro del SVG del botón de minimizar que represente la pupila del ojo, logrando legibilidad visual instantánea.
* **Topología Correcta:** Modificaremos la documentación técnica para explicar con precisión la consistencia topológica: el botón MIN es internamente consistente (4 segmentos) y el botón MAX es internamente consistente (16 segmentos).

### 4. Accesibilidad (WCAG 2.1 AA)
* Añadiremos `aria-label` dinámicos que se actualicen en base al estado de la ventana (ej. *"Maximizar - clic derecho para Modo Nano"*).
* Habilitaremos navegación por teclado añadiendo `tabindex="0"`, estilos `:focus-visible` (anillo de foco nítido y premium) y manejadores `keydown` que simulen el clic normal con `Enter` / `Space` y el modo alternativo mediante `Shift + Enter` o `Shift + Space`.

### 5. Documentos Complementarios
* **Especificación de la API de Nano Mode:** Escribiremos un documento formal (`especificacion_nano_mode_api.md`) definiendo breakpoints, contenedor adaptable de consultas y el flujo IPC.
* **Aclaración del Modo Focus:** Detallaremos las limitaciones de `backdrop-filter` nativo en entornos web y cómo interactúa con el sistema operativo de escritorio.