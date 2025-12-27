# Examen sustitutorio (variante B)

- Nombre y apellidos: Flores Alberca, Ángel Aarón
- Código de estudiante: 20221346A
- Nombre y código del curso : Desarrollo de Software (CC3S2B)

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

3. Define un estándar extremo de IaC limpia para un entorno Clínico, que incluyan
- Commits firmados
- PRs con doble aprobación
- Gates de seguridad
- Política de secretos rotables
- Convención de variables con tipado y validación fuerte

Incluye 2 anti-patrones silenciosos y su impacto Clínico

4. Selecciona y aplica patrones:
- **Builder** para imágenes
- **Prototype** para entornos de prueba
- **Composite** para observabilidad clínica

Justifica como reducen:
- Errores humanos
- Costos de auditoría
- Tiempo de recuperación

5. Rediseña las dependencias entre servicios usando
- DIP (Dependency Inversion Principle)
- Mediator
- Contratos versionados

Define
- Invariantes de datos 
- Límites de responsabilidad
- ¿Qué ocurre ante degradación parcial?

6. Define un plan de pruebas IaC y plataforma que incluya:
- Seguridad IaC 
- Pruebas de rollback con datos sensibles
- E2E con teardown seguro
- Políticas OPA especificas del dominio salud 

Lista 5 defectos críticos detectables y 2 no detectables automáticamente.

7. Diseña el despliegue final
- Docker con aislamiento fuerte
- Kubernetes con zero-trust real 
- NetworkPolicies por identidad
- Estrategia de actualización conservadora (justiificada)
- Alertas clínicas (no técnicas)

Incluir evidencia mínima de no-repudio técnico
