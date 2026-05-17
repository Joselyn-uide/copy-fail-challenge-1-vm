# Reporte Técnico: Análisis del CVE-2026-31431 (Copy-Fail)
**Estudiante:** Joselyn Laguna 

---

### 1. ¿Cuál es el bug raíz y en qué archivo/función está?
El problema principal se encuentra en el subsistema criptográfico del Kernel de Linux, específicamente en el archivo `crypto/algif_aead.c` dentro de la función `_aead_recvmsg()`, que maneja los sockets de la familia de algoritmos `AF_ALG`. 

El origen del bug se remonta a una optimización de rendimiento implementada en el año 2017. Con el objetivo de ahorrar memoria y evitar copias innecesarias de buffers, se utilizó la función `sg_chain()` para realizar una operación *in-place*. Esto causó que las páginas de transmisión (TX SGL) y recepción (RX SGL) compartieran el mismo espacio físico en la memoria del Kernel, haciendo que la asignación lógica apuntara a `req->src = req->dst`. Aunque parecía una mejora lógica para agilizar el procesamiento criptográfico, esta falta de aislamiento provocó que el Kernel perdiera la distinción entre buffers de origen y destino, permitiendo que operaciones de escritura  afectaran regiones de memoria sensible.

### 2. ¿Por qué el write a dst[assoclen + cryptlen] es peligroso?
Este write es extremadamente peligroso porque escribe el tag de autenticación criptográfica exactamente al final del buffer de datos combinados (`assoclen + cryptlen`). Debido al bug raíz donde `src == dst`, el Kernel termina colocando páginas que pertenecen al Page Cache del sistema directamente en un scatterlist diseñado para operaciones de escritura.

Cuando la operación criptográfica finaliza, el subsistema escribe el resultado en la dirección calculada de `dst`. Al no estar aislada, esa escritura externa altera directamente las páginas físicas de la memoria RAM administradas por el Page Cache. Lo crítico aquí es que el hardware procesa el evento como una operación criptográfica interna y legítima del Kernel, por lo que los mecanismos tradicionales de protección de memoria no detectan la anomalía, permitiendo la corrupción de archivos críticos del sistema en tiempo de ejecución.

### 3. ¿Por qué el exploit es "stealthy" (no modifica el archivo en disco)?
El exploit es considerado "stealthy" (sigiloso) porque opera exclusivamente en el espacio de la memoria volátil (RAM) y jamás altera un solo byte del archivo físico almacenado en el disco duro o estado sólido.

El ataque aprovecha el comportamiento nativo de Linux: cuando un usuario ejecuta un binario como `/usr/bin/su`, el Kernel primero lo carga del disco al Page Cache en la RAM para acelerar su ejecución. El exploit intercepta esa copia temporal en memoria y la modifica "al vuelo". Cuando el sistema procesa la ejecución, utiliza la versión corrompida que reside en la RAM. Herramientas de auditoría como el cálculo de hashes (SHA256/MD5) o antivirus basados en firmas fallan en detectar el ataque porque, si inspeccionan el almacenamiento físico, el archivo original permanece completamente intacto y limpio. La alteración desaparece por completo en cuanto la página es desalojada de la memoria o el sistema se reinicia.

### 4. Conecta esto con lo que vimos en clase: page cache, chmod, setuid, inodos

Este laboratorio conecta de forma directa varios de los conceptos abalizados en clase. Todo el ataque se fundamenta en el comportamiento nativo del **Page Cache**, el mecanismo que utiliza el Kernel de Linux para almacenar en la memoria RAM réplicas de los bloques de disco de uso frecuente. Cuando un usuario interactúa con un archivo, el sistema no lee el almacenamiento físico constantemente, sino que reutiliza estas páginas en memoria. El exploit aprovecha esta arquitectura para actuar de forma sigilosa; no necesita alterar el ejecutable real en reposo, sino que corrompe directamente la página activa mapeada en la memoria volátil.

Para que el Kernel pueda organizar este espacio de intercambio eficientemente, depende de los **Inodos** , que son las estructuras que Linux utiliza internamente para identificar y administrar las propiedades de cada archivo. El Page Cache indexa sus páginas de memoria basándose precisamente en el número de inodo del archivo y sus desplazamientos numéricos u *offsets*. El exploit aprovecha este direccionamiento calculando matemáticamente el offset exacto dentro del inodo del binario objetivo (en este caso, `/usr/bin/su`) para inyectar el código malicioso en el sector de memoria exacto donde se procesan las credenciales.

Finalmente, el impacto de esta corrupción de memoria se vuelve crítico debido a la existencia de los permisos especiales controlados por **chmod** , específicamente el bit **setuid**. Cuando un programa legítimo tiene este bit activo (visualmente representado con una 's' en los permisos de ejecución), el Kernel fuerza a que el proceso corra con los privilegios del dueño del archivo —que usualmente es `root`— y no con los del usuario común que lo invoca. Al interceptar y modificar la página en caché de un binario que posee esta propiedad, el atacante logra que el código inyectado herede de inmediato los privilegios de administrador del sistema, completando una elevación de privilegios exitosa que transforma el prompt de `student` a `root` sin levantar alertas.

### 5. ¿Qué aprendiste sobre cómo múltiples cambios "razonables" pueden crear un bug grave?
Lo que más me llamó la atención es que el bug no apareció porque alguien programó “mal” intencionalmente. El problema fue que varias decisiones aparentemente normales terminaron interactuando de una forma inesperada.

Por otro lado, usar memoria compartida, aprovechar splice() o tener binarios con setuid no parecía algo crítico. Pero cuando todas esas piezas se combinaron, crearon una vulnerabilidad enorme que pasó desapercibida durante años.

Esto me hizo entender que en ciberseguridad no basta con revisar cada componente individualmente. Muchas veces los errores graves nacen de cómo diferentes partes del sistema interactúan entre sí. También aprendí que optimizar rendimiento sin pensar en seguridad puede traer consecuencias muy serias.


---

# Technical Report: Analysis of CVE-2026-31431 (Copy-Fail)
**Student:** Joselyn Laguna

---

### 1. What is the root bug and in which file/function is it located?
The main problem is found in the Linux kernel's cryptographic subsystem, specifically in the `crypto/algif_aead.c` file within the `_aead_recvmsg()` function, which handles sockets of the `AF_ALG` algorithm family.

The bug originated from a performance optimization implemented in 2017. To save memory and avoid unnecessary buffer copies, the `sg_chain()` function was used to perform an *in-place* operation. This caused the transmit (TX SGL) and receive (RX SGL) pages to share the same physical space in kernel memory, making the logical allocation point to `req->src = req->dst`. While this seemed like a logical improvement to speed up cryptographic processing, this lack of isolation caused the kernel to lose the distinction between source and destination buffers, allowing write operations to affect sensitive memory regions.

### 2. Why is `write to dst[assoclen + cryptlen]` dangerous?

This write is extremely dangerous because it writes the cryptographic authentication tag directly to the end of the combined data buffer (`assoclen + cryptlen`). Due to the root bug where `src == dst`, the kernel ends up placing pages belonging to the system page cache directly into a scatterlist designed for write operations.

When the cryptographic operation completes, the subsystem writes the result to the calculated address of `dst`. Because it is not isolated, this external write directly alters the physical pages of RAM managed by the Page Cache. The critical point here is that the hardware processes the event as a legitimate, internal cryptographic operation of the Kernel, so traditional memory protection mechanisms fail to detect the anomaly, allowing the corruption of critical system files at runtime.

### 3. Why is the exploit "stealthy" (doesn't modify the file on disk)?

The exploit is considered "stealthy" because it operates exclusively in volatile memory (RAM) and never alters a single byte of the physical file stored on the hard drive or solid-state drive.

The attack takes advantage of a native Linux behavior: when a user executes a binary like `/usr/bin/su`, the kernel first loads it from disk into the Page Cache in RAM to speed up execution. The exploit intercepts this temporary copy in memory and modifies it "on the fly." When the system processes the execution, it uses the corrupted version residing in RAM. Auditing tools such as hash calculations (SHA256/MD5) or signature-based antivirus software fail to detect the attack because, if they inspect the physical storage, the original file remains completely intact and clean. The alteration disappears completely as soon as the page is evicted from memory or the system restarts.

### 4. Connect this to what we saw in class: page cache, chmod, setuid, inodes

This lab directly connects several of the concepts discussed in class. The entire attack is based on the native behavior of the **Page Cache**, the mechanism the Linux kernel uses to store replicas of frequently used disk blocks in RAM. When a user interacts with a file, the system doesn't constantly read the physical storage, but rather reuses these pages in memory. The exploit takes advantage of this architecture to act stealthily; it doesn't need to alter the actual executable at rest, but instead directly corrupts the active page mapped in volatile memory.

For the kernel to efficiently organize this swap space, it relies on **Inodes**, which are the structures Linux uses internally to identify and manage the properties of each file. The Page Cache indexes its memory pages based precisely on the file's inode number and its numerical offsets. The exploit leverages this addressing by mathematically calculating the exact offset within the inode of the target binary (in this case, `/usr/bin/su`) to inject malicious code into the precise memory sector where credentials are processed.

Finally, the impact of this memory corruption becomes critical due to the existence of special permissions controlled by **chmod**, specifically the **setuid** bit. When a legitimate program has this bit set (visually represented by an 's' in the execution permissions), the kernel forces the process to run with the privileges of the file owner—usually `root`—and not those of the regular user invoking it. By intercepting and modifying the cached page of a binary with this property, the attacker ensures that the injected code immediately inherits system administrator privileges, completing a successful privilege escalation that transforms the prompt from `student` to `root` without raising any alerts.

### 5. What did you learn about how multiple "reasonable" changes can create a serious bug?

What struck me most was that the bug didn't appear because someone intentionally programmed "badly." The problem was that several seemingly normal decisions ended up interacting in an unexpected way.

On the other hand, using shared memory, leveraging splice(), or having binaries with setuid didn't seem critical. But when all those pieces combined, they created a huge vulnerability that went unnoticed for years.

This made me understand that in cybersecurity, it's not enough to review each component individually. Often, serious errors arise from how different parts of the system interact with each other. I also learned that optimizing performance without considering security can have very serious consequences.
