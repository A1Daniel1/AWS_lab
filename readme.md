# Laboratorio de Alta Disponibilidad con AWS

Laboratorio práctico sobre alta disponibilidad usando **EC2**, **Application Load Balancer** y **Target Groups** en AWS.

---

## Arquitectura

```
       Usuario
          |
          | (DNS del ALB)
          v
  ┌─────────────────┐
  │  Application    │
  │  Load Balancer  │
  └────────┬────────┘
           |
  ┌────────┴────────┐
  │  Target Group   │
  └──┬──────────┬───┘
     |          |
     v          v
 ┌────────┐ ┌────────┐
 │ EC2 A  │ │ EC2 B  │
 │ (AZ A) │ │ (AZ B) │
 └────────┘ └────────┘
```

---

## Requisitos previos

- Cuenta de AWS con permisos para crear EC2, ALB, Target Groups y Security Groups.
- AWS CLI configurado (opcional).
- Navegador web para verificar el acceso.

---

## Recursos del laboratorio

| Archivo | Descripción |
|---|---|
| `userData.sh` | Script de bootstrap para la **Instancia A** |
| `userData2.sh` | Script de bootstrap para la **Instancia B** |
| `readme.md` | Documentación del laboratorio |
| `assets/image-*.png` | Capturas de evidencia del laboratorio |

### Scripts de bootstrap

Ambos scripts instalan **Apache (httpd)**, obtienen metadatos de la instancia (ID y AZ) mediante **IMDSv2**, y generan una página `index.html` que identifica qué instancia respondió. También crean un endpoint `/health` para los health checks.

---

## Pasos del laboratorio

### 1. Crear las instancias EC2

Lanzar dos instancias EC2 con Amazon Linux 2023, cada una en una **zona de disponibilidad distinta** (AZ A y AZ B). Asignar el script `userData.sh` a la instancia A y `userData2.sh` a la instancia B.

![Instancias EC2](assets/image.png)

### 2. Verificar acceso individual

Acceder a la IP pública de cada instancia desde el navegador para confirmar que responden correctamente.

![Acceso a instancia](assets/image-1.png)

### 3. Crear el Target Group

Crear un **Target Group** que registre ambas instancias EC2 en el puerto 80 con un health check en la ruta `/health`.

![Target Group](assets/image-2.png)

### 4. Crear el Application Load Balancer

Crear un **ALB** público asociado al Target Group. El ALB distribuye el tráfico entre ambas instancias.

![Application Load Balancer](assets/image-3.png)

### 5. Verificar estado de las instancias

Confirmar que ambas instancias aparecen como **Healthy** en el Target Group.

![Health check - instancias sanas](assets/image-4.png)

### 6. Acceder a través del ALB

Ingresar desde el **DNS del Load Balancer**. Refrescar la página para observar cómo el balanceador alterna entre ambas instancias.

![Acceso vía ALB](assets/image-5.png)

---

## Preguntas — Parte 1

1. ¿Qué instancia respondió primero?
2. ¿El balanceador alternó entre ambas instancias?
3. ¿Qué información permite confirmar que hay más de una instancia activa?
4. ¿Qué papel cumple el Target Group?
5. ¿Qué papel cumplen los health checks?
6. ¿Por qué el usuario no necesita conocer las IP públicas de las instancias?

---

## Simulación de falla

Se detuvo la **instancia A** para simular una caída.

![Instancia A detenida](assets/image-6.png)

A pesar de la falla, el sistema sigue respondiendo desde la **instancia B** a través del balanceador.

![Sigue respondiendo instancia B](assets/image-5.png)

### Preguntas — Parte 2

1. ¿Qué ocurrió cuando se detuvo la instancia A?
2. ¿El sistema completo dejó de estar disponible?
3. ¿Qué hizo el Load Balancer cuando detectó la falla?
4. ¿Qué diferencia habría si solo existiera una instancia?
5. ¿Qué atributo de calidad mejora esta arquitectura?

---

## Recuperación

Se volvió a levantar la **instancia A**. Ambas instancias quedan operativas nuevamente.

![Instancia A recuperada](assets/image-7.png)

### Preguntas — Parte 3

1. ¿Qué ocurrió cuando la instancia A volvió a estar saludable?
2. ¿El balanceador volvió a enviarle tráfico?
3. ¿Por qué es importante que la recuperación sea automática desde el punto de vista del usuario?
4. ¿Qué limitaciones tiene esta arquitectura si la instancia no se reinicia manualmente?

### Tabla de componentes

| Elemento | Función en la arquitectura |
|---|---|
| EC2 instancia A | Ejecuta la aplicación web en la AZ A |
| EC2 instancia B | Ejecuta la aplicación web en la AZ B (redundancia) |
| Application Load Balancer | Distribuye el tráfico entrante entre las instancias sanas |
| Target Group | Agrupa las instancias y define las reglas de health check |
| Health Check | Monitorea el estado de cada instancia periódicamente |
| Security Group del ALB | Controla el tráfico entrante hacia el balanceador |
| Security Group de EC2 | Controla el tráfico entrante hacia las instancias |
| Zonas de disponibilidad | Aíslan las instancias físicamente para evitar puntos únicos de falla |

---

## Verificación de health checks

![Verificación health check](assets/image-8.png)

---

## Propuesta de mejora para producción

A partir de la arquitectura implementada, una versión mejorada para producción debería incluir:

### Recuperación automática
- Usar **Auto Scaling Group** con un mínimo de 2 instancias para reemplazar instancias fallidas automáticamente.
- Configurar **Recovery de CloudWatch Alarm** para reiniciar instancias ante fallos de sistema.

### Seguridad
- Colocar las EC2 en **subredes privadas** y usar un **NAT Gateway** para salida a internet.
- Solo el ALB debe estar en una **subred pública**.

### HTTPS
- Asociar un **certificado SSL/TLS** desde **AWS Certificate Manager (ACM)** al ALB.
- Redirigir el tráfico HTTP (puerto 80) a HTTPS (puerto 443) a nivel del ALB.

### Observabilidad
- Habilitar **Access Logs** del ALB en S3.
- Configurar **CloudWatch Metrics** y **CloudWatch Alarms** para CPU, estado de health checks, etc.
- Usar **AWS X-Ray** para trazar solicitudes entre el ALB y las instancias.

### Despliegues sin caída
- Implementar **rolling updates** o **blue/green deployments** con **CodeDeploy**.
- Usar **Auto Scaling** con **lifecycle hooks** para drenar conexiones antes de terminar instancias.

### Base de datos altamente disponible
- Reemplazar una base de datos local por **Amazon RDS Multi-AZ** (sincrónico entre AZs).
- Para cargas de trabajo NoSQL, usar **Amazon DynamoDB** con tablas globales.

---

## Entregables del laboratorio

El estudiante debe entregar un documento breve con:

1. Diagrama de arquitectura implementada.
2. Captura de las dos instancias EC2.
3. Captura del Target Group con targets Healthy.
4. Captura del Application Load Balancer.
5. Evidencia de respuesta desde instancia A.
6. Evidencia de respuesta desde instancia B.
7. Evidencia de falla simulada.
8. Explicación de cómo el balanceador mantiene la disponibilidad.
9. Limitaciones de la arquitectura.
10. Propuesta de mejora hacia producción.
