# Update: Examen [Número]

## Control de Horario de Atención de Tutorías

### Objetivo
Evitar que los estudiantes soliciten tutorías fuera del horario de atención de la coordinación (Lunes a Viernes, 8:00 AM a 6:00 PM), bloqueando el flujo de solicitud mientras se mantiene disponible la consulta de estado de tutorías en todo momento.

### Lógica implementada

1. **Punto de entrada — `¿Solicitar Tutoría?`**
   Justo después de recibir el mensaje de Telegram, un nodo `IF` evalúa si el texto del usuario corresponde a *"Solicitar Tutoría"*. La comparación se configuró como **insensible a mayúsculas/minúsculas** (`caseSensitive: false`) y se aplica `.trim()` al texto recibido, para evitar que espacios extra o variaciones como "solicitar tutoría" rompan la detección.

2. **Validación de horario — `¿Dentro del horario de atención?`**
   Si el usuario eligió "Solicitar Tutoría", el flujo pasa por un segundo nodo `IF` que evalúa la expresión temporal:
   ```
   $now.weekday >= 1 && $now.weekday <= 5 && $now.hour >= 8 && $now.hour < 18
   ```
   - `$now.weekday` usa la convención 1 = Lunes … 7 = Domingo, por lo que `1-5` cubre Lunes a Viernes.
   - `$now.hour >= 8 && $now.hour < 18` cubre el rango de 8:00 AM a 5:59 PM (hasta antes de las 6:00 PM).

3. **Ramas de salida según horario**
   - **Dentro de horario (`true`)** → Nodo `Telegram - Solicitud Permitida`: confirma al estudiante que puede continuar con su solicitud.
   - **Fuera de horario (`false`)** → Nodo `Telegram - Coordinación Cerrada`: envía el mensaje de cierre y **detiene el flujo** (no hay nodos posteriores), evitando que la solicitud avance hacia la asignación de tutores.
     ```
     🌙 Coordinación Cerrada. Nuestro horario de atención es de Lunes a Viernes, 8am a 6pm. ¡Escríbenos mañana!
     ```

4. **Excepción — Consultar estado de tutorías**
   Si el usuario **no** eligió "Solicitar Tutoría", el flujo se dirige al nodo `¿Consultar estado?`, que verifica si el texto corresponde a *"Consultar estado de tutorías"` (también insensible a mayúsculas). Esta rama **no pasa por la validación de horario**, por lo que los estudiantes pueden consultar el estado de sus tutorías en cualquier momento, día o noche.

5. **Manejo de mensajes no reconocidos (mejora agregada)**
   Se añadió una rama de salida adicional en `¿Consultar estado?` para el caso en que el texto del usuario no coincida con ninguna opción válida. En ese caso, se envía el mensaje:
   ```
   🤔 No entendí tu mensaje. Por favor selecciona una opción válida: "Solicitar Tutoría" o "Consultar estado de tutorías".
   ```
   Esto evita que el bot quede en silencio ante entradas inesperadas y mejora la experiencia del estudiante.

### Resumen del flujo

```
Telegram - Recibir Mensaje
        │
        ▼
¿Solicitar Tutoría? ──(false)──► ¿Consultar estado? ──(true)──► Telegram - Consultar Estado
        │                               │
     (true)                          (false)
        │                               ▼
        ▼                    Telegram - Mensaje No Reconocido
¿Dentro del horario de atención?
        │
   ┌────┴────┐
 (true)    (false)
   │           │
   ▼           ▼
Solicitud   Coordinación
Permitida    Cerrada (fin)
```

### Nodos nuevos/modificados respecto a la versión anterior
- `¿Solicitar Tutoría?` — ahora insensible a mayúsculas y con `.trim()`.
- `¿Consultar estado?` — insensible a mayúsculas y con nueva rama `false`.
- `Telegram - Mensaje No Reconocido` — **nodo nuevo**, maneja entradas no válidas.
