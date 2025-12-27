# Examen sustitutorio (variante B)

- Nombre y apellidos: Flores Alberca, Ángel Aarón
- Código de estudiante: 20221346A
- Nombre y código del curso : Desarrollo de Software (CC3S2B)

- Link al video de explicación: https://drive.google.com/file/d/1t8QhBkG2igzWCxD027FpVaFn7Xg99bQg/view?usp=sharing

## Contexto único: "Salud crítica - Nodo Local de Diagnóstico Clínico"

Sistema local de clínica rural para:
- Adquisición de señales biomédicas
- Inferencia ML local
- Almacenamiento temporal cifrado
- Sincronización diferida

Microservicios mínimos:
- `acquisition-service`
- `inference-service`
- `audit-service`

Restricciones clave:
- Datos altamente sensibles
- Cero confianza en red interna (zero trust)
- Evidencia legal de qué versión diagnosticó qué paciente
- Actualizaciones poco frecuentes y muy controladas

## Resolución a preguntas clave

1. Justifica por qué **IaC + supply chain security** es indispensable en este sistema.
Conectar explícitamente:
- Reproducibilidad
- SBOM
- Firmas
- Provenance 
- Responsabilidad legal y clínica

Incluir un ejemplo de disputa técnica realista

### Respuesta:
Para un contexto clínico, como aquel del caso de estudio, Infraestructure As Code (IaC) es obligatoria debido a la rigurosidad legal a la que estos sistemas son sometidos, donde un pequeño error puede tener consecuencias desastrozas en pacientes. Podemos elaborar esta necesidad bajo los siguientes puntos:
- **Responsabilidad clínica y Provenance**: Si un paciente recibiese una malapraxis o un diagnóstico incorrecto, la clínica tiene como obligación probar su *no repudio* y validez del sistema por el servicio dado. Gracias a Provenance (Proof of origin), se puede demostrar qué versión y qué tipo de modelo de ML fue el ejecutado durante su diagnóstico.
- **SBOM (Software Bill of Materials)**: Se necesita la generación de un SBOM por cada despliegue realizado para garantizar la validación de las dependencias utilizadas en nuestro sistema. Un ejemplo podría ser el descubrimiento de una vulnerabilidad en una libreria como `numpy` o `pytorch` que altere los resultados dados por el servicio de inferencia. 
- **Firmas verificadas**: Las firmas digitales verifican la autoría del desarrollador del sistema asi como el del constructor del pipeline CI/CD. Esto ayuda a prevenir vulnerabilidades de autores no rastreables (man-in-the-middle) o sospechosos dentro del pipeline.

*Ejemplo de disputa técnica realista*
Puede ser que un paciente reciba un diagnóstico de cáncer de pulmón, para el cual es sometido a distintos rigurosos examenes y tratamientos experimentales inmediatamente. Sin embargo, semanas después se descubre que el paciente nunca tuvo cáncer, por lo qu el paciente ahora demanda a la clínica por negligencia médica y daños físicos severos debido al tratamiento innecesario. Los abogados del paciente alegarían a que el modelo de inferencia del hospital fue uno defectuoso por tanto el hospital ahora necesita probar la validez del sistema empleado para el diagnóstico.

---

2. Se descubre que:
- Una imagen fue reconstruida _igual que antes_, pero sin tag
- El clúster no puede probar que versión corrió
- Los manifiestos no estaban versionados

Diseñar una cadena de evidencia técnica que permita responder:
- ¿Qué se desplegó?
- ¿Cuándo?
- ¿Por quién?
- ¿Con qué dependencias?

Incluye herramientas, artefactos y verificaciones

### Respuesta: 
Para resolver este tipo de problemas, necesitamos transicionar de tags mutables (ej. Latest) a digests inmutables.

- **Digests inmutables (SHA256)**: Implementación automatizada de tags de imágenes con el hash asociado en lugar de utilizar SemVer para mayor rigurosidad.
- **Uso de Cosign para firmas**:
  - En el pipeline CI/CD, inmeditadamente después de construir la imagen Docker, se firma su validez usando una llave privada almacenada en una repositorio de secrets.
  - Uso de validadores en Kubernetes como Gatekeeper/OPA para el rechazo de la creación de cualquier pod cuya firma sea inválida o faltante
- **Declaración de provenance**: Se añade un archivo de metadata al image registry que contenga el hash del commit de git, el builder ID y el timestamp de la creación de dicho artefacto.
- **Generación de logs**: Dentro de un repositorio de logs públicos o privados donde se muestren los eventos de construcción e inicialización requeridos. Previene a que cualquiera pueda cambiar *secretamente* la imagen.

---

3. Define un estándar extremo de IaC limpia para un entorno Clínico, que incluyan
- Commits firmados
- PRs con doble aprobación
- Gates de seguridad
- Política de secretos rotables
- Convención de variables con tipado y validación fuerte

Incluye 2 anti-patrones silenciosos y su impacto Clínico

### Respuesta:
Para nuestro estándar extremo de IaC limpia explicaremos y justificaremos en más profundidad los requerimientos mostrados:
- **Commits firmados (GPG)**: El pipeline CI/CD rechaza commits no verificados y que no cumplan con estándares como conventional commits. Permiten probar quienes fueron los autores de cambios en el código del sistema clínico. 
- **PRs con doble aprobación**: Los Pull Requests ahora requieren de la aprobación de un senior y un QA developer para ser aceptados. Así como los archivos CODEOWNERS requieren aprobación obligatoria del equipo de seguridad y operaciones para el acceso a ciertos directorios.
- **Gates de seguridad**: Haciendo uso del `shift left` definimos el siguiente flujo
  - Usando `gitleaks` o `pre-commit-hooks` escaneamos los commits en búsqueda de secretos hardcodeados.
  - Linting y formatting con `tflint`, `black` o `yamllint`
  - Policy as Code (estático) para verificación de reglas de seguridad estáticas `terrascan` o `checkov`
  - Verificación del grafo de cambios para evitar problemas de drift `kubectl diff` o `terraform plan`
  - Policy as Code (dinámico) para rechazo en tiempo de admisión si un recurso no cumple con las reglas con `OPA Gatekeeper`
- **Política de secretos rotables**: Automatización de renovación y cambio para variables o secretos temporales como API KEYS o certificados TLS con TTLs moderados. Rotación automática en caso de servicios duraderos mediante *Sidecars*.
- **Convención de variables con tipado y validación fuerte**: Prohibir el uso de tipos genéricos para valores numéricos o booleanos. Contar con nombres de variables auto documentables, es decir, indicar su propósito de existencia en el mismo nombre. Implementación de reglas de validación estrictas y significativas para evitar problemas de tipado.

Antipatrones silenciosos y su impacto:
- **Uso de valores predeterminados**: Caer en el uso de valores por default (ej. límite de recursos asignados) o en variables ambiguas en lugar de realizar declaraciones explícitas en el código. Puede conducir a comportamientos ambiguos e inentendibles donde se requiera atención crítica, como por ejemplo un proceso eliminado por falta de memoria.
- **Selectores permisivos**: Ejecución de comandos con etiquetas demasiado genéricas sin restricciones estrictas, puede llevar consigo el despliegue de recursos insolicitados acrecentando costos y en la sobreescritura de configuraciones en producción, lo que resulta en la corrupción de expedientes médicos activos. 

---

4. Selecciona y aplica patrones:
- **Builder** para imágenes
- **Prototype** para entornos de prueba
- **Composite** para observabilidad clínica

Justifica como reducen:
- Errores humanos
- Costos de auditoría
- Tiempo de recuperación

### Respuesta:
- **Builder** para imágenes:
En lugar de la declaración extensa y a veces repetitivo de Dockerfiles inmensos, se puede implementar un proceso de _Multi-Stage build_ orquestrado haciendo uso del patrón **Builder**. 

  - Al automatizar el ensamblaje, nos libramos de errores de configuración de seguridad por desarrolladores y garantizamos una imagen resultante determinista en base a la llamada de métodos predefinidos. 

  - Acortamos tiempos de auditoría en los que en vez de verificar cientos de Dockerfiles inmensos, solo se necesita validar la funcionalidad del constructor. 

  - Por último, si es que se detectase alguna vulnerabilidad, podemos realizar una actualización inmediata a las imágenes comprometidas con una reconstrucción desde el mismo builder y sin tener que reconfigurar manualmente a todas.

- **Prototype** para entornos de prueba:
Definimos un conjunto de manifiestos de K8S que incluyan todo lo necesario para un entorno de pruebas clínico estándar y _clonamos_ múltiples instancias de la misma con ligeras variaciones para diferentes escenarios.

  - Eliminamos posibilidad de _configuration drift_ en la creación de entornos de pruebas causando falsos positivos que transicionen a producción.
  
  - Así como con Builder, el auditor solo necesita verificar la plantilla inicial del cual deslindan sus variaciones, reduciendo horas de revisión.
  
  - Permite un reset instantáneo donde si un entorno es corrompido o falla, entonces se descarta y se clona uno nuevo desde la plantilla inicial, así reduciendo tiempo de inactividad a los desarrolladores. 

- **Composite** para observabilidad clínica:
Considerando al sistema de salud como una _jerarquía_ de diferentes etapas y servicios podemos estructurarlo en forma de árbol junto a un servicio de observabilidad hacia los nodos de dicho arbol que facilite la visualización de arquitecturas complejas. 

  - Estructura adecuadamente las alertas técnicas, en lugar de recibir grandes mensajes no distribuidos y desconectados, sino que se recibe una señal semántica. Así mismo, evita ignorar fallos críticos ocultos en el ruido.
  
  - Los logs generados ahora estan correlacionados por ID de transacción clínica, asociados a sus ID por composite, por lo que un auditor puede verificar datos inmediatos en lugar de tener que ejecutar múltiples búsquedas a distintos servicios.
  
  - Permite la detección de qué nodos del árbol fallaron mediante reversión de logs, así evitando pérdidas de tiempo en encontrar el problema real. 

---

5. Rediseña las dependencias entre servicios usando
- DIP (Dependency Inversion Principle)
- Mediator
- Contratos versionados

Define
- Invariantes de datos 
- Límites de responsabilidad
- ¿Qué ocurre ante degradación parcial?

### Respuesta:

Para garantizar un sistema desacoplado, auditable y resistente a fallos, reestructuramos las dependencias directas mediante abstracciones y mediación explícita. Bajo **DIP**, los servicios como `acquisition-service` e `inference-service` no se conocen entre sí, sino que ambos dependen de contratos versionados que funcionan como interfaces. Ningún servicio requiere de conocer la implementacion del otro, lo que permite grandes actualizaciones sin coordinar cambios entre múltiples servicios, siempre que se respete los contratos establecidos. 

Así mismo podemos incluir un **Mediator** que actúa como bus de eventos centralizado coordinando el flujo entre los servicios. Cuando `acquisition-service` capture una señal, publica un evento que indique así mismo la versión del contrato que notifica a los servicios suscritos. Este patrón resuelve la sincronización temporal crítica en conectividad intermitente, si la inferencia esta inactiva, el mediador retiene todos los mensajes hasta su recuperación evitando pérdidas de datos de diagnósticos.

Por último, los contratos versiones siguen SemVer estricto definidos bajo esquemas con metadata clínica, podemos valernos de unit tests, compilación y ejecución para la validación de estos contratos.

- **Invarianza de datos**: El sistema garantizaría tres invariantes críticos verificados centralmente por el mediador antes de aceptar mensajes. Primero sería el añadimiento de timestamps e identificadores para pacientes hasheado. Segundo, es acerca de la inmutabilidad absoluta, ningún dato puede modificarse post-registro, solo agregarse anotaciones que hagan referencia al original mediante su hash. Por último, toda inferencia diagnósticada deberá de incluir hash de datos de entrada, del modelo ejecutado y el timestamp de ejecución, así conformando una cadena de evidencia criptográfica. 

- **Limites de responsabilidad**: Se requiere de una distribución estricta entre los servicios definidos (siguiendo Single Responsability Principle - SOLID) para el aislamiento de incidentes. Rápidamente, podemos decir que:
  - `acquisition-service`: Captura señales con calidad suficiente y etiqueta el tipo de señal.
  - `inference-service`: Aplica modelos ML a datos que cumplan su contrato de entrada, sin acceder a ninguna base de datos o a registros del paciente.
  - `audit-service`: Registra inmutablemente toda transacción incluyendo fallos y rechazos mediante logs cifrados.
Esto ayuda a que si un modelo produciese resultados anómalos, la auditoría podría demostrar los datos de entrada, acotando la investigación únicamente a `inference-service`

- **¿Qué ocurre ante degradación parcial?**: 
En caso de que alguno de los servicios falle, los servicios deben de contar con protocolos de espera hasta la detección de recuperación. Si el mediador falla, servicios entran en comunicación directa temporal usando endpoints de emergencia predefinidos, registrando localmente transacciones para reconciliación.

---

6. Define un plan de pruebas IaC y plataforma que incluya:
- Seguridad IaC 
- Pruebas de rollback con datos sensibles
- E2E con teardown seguro
- Políticas OPA especificas del dominio salud 

Lista 5 defectos críticos detectables y 2 no detectables automáticamente.

### Respuesta:

Diseñamos una estrategia de seguridad en profunidad integrada en el pipeline CI/CD para validar la lógica de IaC como la lógica del negocio antes del despliegue:

- **Seguridad IaC (Shift-left)**: Ejecución obligatoria de SAST (`Trivy`) sobre manifiestos terraform/k8s antes de algún _apply_. Bloquea configuraciones vulnerables como buckets públicos o `securityContext` privilegiados.
- **Pruebas de rollback con datos sensibles**: Validación de compatibilidad de esquimas y claves. Simulamos una migración de datos y su reversión inmediata, la prueba solo pasaría si la versión antigua logra descifrar los datos escritos por la versión nueva, evitando _corrupción de expedientes_.
- **E2E con teardown seguro**: Una vez terminadas las pruebas funcionales, se destruyen las llaves de cifrado en el entorno de prueba y aplicamos eliminación total de todos los volúmenes empleados
- **Políticas OPA específicas del dominio de salud**:
  - Bloqueo total de Egress a internet para `inference-service`, solo se permite trafico interno contando con mTLS.
  - Solo se permiten que pods de namespaces `acquisiton` firmados puedan montar dispositivos host.
  - Rechazo total a cualquier PVC que no declare `StorageClass` con cifrado en reposo

Defectos críticos detectables:
- Secretos o API KEYS hardcodeados.
- Contenedores sin usuarios restringidos
- Ausencia de definición límite o mínimo de recursos
- Exposición de puertos inseguros sin mTLS
- Imágenes base sin firma digital verificable

Defectos críticos no detectables: 
- Incumplimiento de trato de datos legal en base legislaciones del país o lugar de despliegue del centro clínico.
- Sesgo algorítimo en modelos ML en base a datos de entrenamiento maliciosos.

---

7. Diseña el despliegue final
- Docker con aislamiento fuerte
- Kubernetes con zero-trust real 
- NetworkPolicies por identidad
- Estrategia de actualización conservadora (justificada)
- Alertas clínicas (no técnicas)

Incluir evidencia mínima de no-repudio técnico

### Respuesta:
- **Docker con aislamiento fuerte**: 
Se definieron las siguientes características para la garantía de aislamiento de los contenedores Docker:
  - Contar con estrategia de construcción MultiStage-build haciendo uso de imágenes distroless/scratch con el fin de reducir la superficie de ataque de los contenedores generados, así como su tamaño final.
  - Creación de usuarios no root para el cumplimiento del **principio de mínimo privilegio** así evitando mapeos al root del host
  - Eliminación de todas las capabilities de Linux por defecto y solo añadir aquellas que sean estrictamente necesarias para la ejecución del servicio como `NET_BIND_SERVICE`
  - Declaración del sistema de archivos de solo lectura `read_only_root_filesystem` para evitar ataques de malware en la modificación de tiempos de ejecución. Los únicos directorios escribibles serían temporales en memoria que se borrarían al detener el contendor.

- **Kubernetes con zero-trust real**: 
Se nos solicita seguir un principio de _zero-trust_. 
  - Implementación de mTLS obligatorio para todo tráfico entre los microservicios `acquisition-service` e `inference-service`. Todos ellos rechazarían cualquier conexión que no presente un certificado válido emitido por la autoridad de certificación interna del clúster
  - Contar con inyección de secretos en memoria, las contraseñas o API KEYS se inyectan via Vault directamente en la memoria RAM, jamáse se exponen como variables de entorno evitando logs de errores o exposición de datos.

- **NetworkPolicies por identidad**: 
Complementando los requerimientos de seguridad anteriores se cumple con:
  - Política `Default Deny-All`: Bloqueo total de tráfico de entrada (Ingress) y salida (Egress) por defecto en el namespace `clinical`
  - Declaración de listas blancas granulares, solo admitir tráfico TCP en puertos declarados con labels específicas
  - Prohibición absoluta de salida a internet para mitigación de filtración de datos sensibles de pacientes como historiales clínicos considerando casos donde contenedores se vean comprometidos. Solo `acquisition-service` tiene salida restringida a la IP fija del servidor central del hospital.

- **Estrategia de actualización conservadora**:
Se contaría con la estrategia `blue-green deployment` siendo esta la más _estable_ para la actualización de software o requerimientos utilizados.
  - Primero hacemos un despliegue de la nueva versión (green) en paralelo sin recibir tráfico
  - Realizamos un _shadow testing_ duplicando tráfico real hacia green para validar la funcionalidad de nuevos modelos
  - Una vez validado podemos realizar el switch y el tráfico se cambia 100% a green.

- **Alertas clínicas técnicas**:
Como parte de la observabilidad requerida, podemos contar con las siguientes métricas para el supervisión de la salud del negocio:
  - **Métricas de seguridad**: Historial de intentos de acceso a expedientes médicos no autorizados agrupados por IP interna.
  - **Alertas de integridad clínica**: Divergencia entre hashes de auditoría, si es que aquellos hashes que estan presenten en los servicios no coinciden con los registrados, se dispara una alerta de manipulación de evidencia.
  - **Alertas de latencia clínica**: Métrica de tiempo total desde `acquisition-service` hasta el diagnóstico final.

- **Evidencia mínima de no-repudiación técnica**:
Retomando el ejemplo presentado en el ejercicio 1, el sistema debería de ser capaz de generar automáticamente _un manifiesto de atestación_ criptográfico por cada diagnóstico. En la que como parámetros de entrada se tienen los datos biométricos crudos, hashes e identificadores de los modelos ML usados en el proceso y el resultado de los diagnósticos firmados digitalmente por la clave privada de `audit-service`. Finalmente se tendría en el log de auditoria la el hash del commit donde se almacena dicho registro de atestación.
