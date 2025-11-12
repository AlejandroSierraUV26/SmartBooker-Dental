# SmartBooker Dental — Arquitectura y Diseño (Fase 1)

**Propósito:** Documento base que describe la arquitectura técnica, roles, flujos y componentes del chatbot de reservas para un consultorio odontológico. Sirve como entregable inicial de la Fase 1 (Planificación y Diseño).

---

## 1. Resumen ejecutivo

SmartBooker Dental es un sistema de reservas automatizado que permite a los pacientes agendar, consultar y cancelar citas odontológicas mediante un chatbot conversacional. El sistema centraliza la gestión de citas, envía confirmaciones/recordatorios por correo y ofrece un panel web para que asistentes y administradores gestionen la agenda del consultorio.

---

## 2. Roles y permisos

* **Paciente**

  * Reservar cita
  * Consultar cita
  * Cancelar cita

* **Asistente**

  * Visualizar agenda completa
  * Confirmar o cancelar citas
  * Agregar notas y observaciones a una cita

* **Administrador**

  * Gestionar servicios y horarios
  * Crear/eliminar cuentas de asistentes
  * Configurar Google Calendar / correo
  * Acceder a reportes y métricas

---

## 3. Stack tecnológico (decidido)

* **Backend / Lógica:** Node.js (Lambda) — funciones serverless escritas en Node.js con AWS SDK.
* **Chatbot:** Amazon Lex V2.
* **Base de datos:** Amazon DynamoDB (primaria). Opcional: Amazon RDS (Postgres) si se requiere modelado relacional.
* **Correo / Notificaciones:** Amazon SES (correo). SNS para SMS/Push (opcional).
* **Panel Web:** React + Tailwind, desplegado con AWS Amplify (o S3+CloudFront).
* **Autenticación:** AWS Cognito (asistentes y administradores).
* **Monitoreo / Logs:** Amazon CloudWatch.
* **Integración externa:** Google Calendar API para sincronizar la agenda del odontólogo.

---

## 4. Arquitectura general (diagrama)

```
[Paciente (WhatsApp / Webchat / SMS)]
            │
            ▼
        Amazon Lex (V2)
            │
            ▼
        AWS Lambda (Node.js)  <---->  Google Calendar API
            │
   ┌────────┼────────┐
   │        │        │
   ▼        ▼        ▼
DynamoDB   SES     CloudWatch
   │
   ▼
API Gateway (exponer endpoints REST para panel web)
   │
   ▼
React App (Admin/Asistente)  -- Cognito (auth)
```

---

## 5. Componentes y responsabilidades

* **Amazon Lex**

  * Detecta intenciones: `ReservarCita`, `ConsultarCita`, `CancelarCita`, `ReprogramarCita`, `PreguntasGenerales`.
  * Extrae slots: `nombre`, `fecha`, `hora`, `servicio`, `telefono`, `email`.
  * Invoca Lambda para decisiones transaccionales.

* **AWS Lambda (Node.js)**

  * `createReservation(payload)` — valida disponibilidad y escribe en DynamoDB; devuelve confirmación.
  * `getReservation(userId | email | phone)` — consulta próximas/pasadas reservas.
  * `cancelReservation(reservationId)` — anula reserva y sincroniza con Google Calendar.
  * `notify(email|phone, template)` — envía confirmaciones/recordatorios via SES/SNS.

* **DynamoDB**

  * Tabla `Reservations` con clave primaria `reservationId` y atributos: `userId`, `name`, `service`, `date`, `time`, `status`, `notes`, `createdAt`, `updatedAt`.
  * Tabla opcional `Users` para persistir pacientes frecuentes.

* **API Gateway + React (Panel)**

  * Panel administrativo consume endpoints REST protegidos por Cognito.
  * Funcionalidades: vista calendario, búsqueda por paciente, cambios de estado (confirmado/cancelado), configuración de servicios y horarios.

* **Google Calendar API**

  * Sincroniza citas confirmadas con el calendario del odontólogo (creación / actualización / eliminación de eventos).

---

## 6. Modelo de datos (ejemplo JSON)

```json
{
  "reservationId": "res_001",
  "userId": "user_123",
  "name": "Juan Pérez",
  "service": "Valoración dental",
  "date": "2025-11-15",
  "time": "09:00",
  "status": "CONFIRMED",
  "email": "juan.perez@example.com",
  "phone": "+57XXXXXXXXX",
  "notes": "Paciente con historial de sensibilidad",
  "createdAt": "2025-11-01T08:23:00Z",
  "updatedAt": "2025-11-01T08:23:00Z"
}
```

---

## 7. Flujos clave (secuencia resumida)

### Reservar cita

1. Usuario inicia conversación en WhatsApp / Webchat.
2. Lex detecta intención `ReservarCita` y solicita slots faltantes.
3. Lambda valida disponibilidad consultando `Reservations` en DynamoDB.
4. Si hay disponibilidad: Lambda crea la reserva, sincroniza con Google Calendar y llama a SES para enviar confirmación.
5. Lex responde al usuario con la confirmación.

### Cancelar cita

1. Usuario solicita cancelar.
2. Lex invoca Lambda con `reservationId` o datos del paciente.
3. Lambda actualiza estado en DynamoDB, elimina/actualiza evento en Google Calendar y envía notificación.

---

## 8. Seguridad y cumplimiento mínimo

* Habilitar HTTPS para todos los endpoints (API Gateway).
* Politicas IAM mínimas para roles Lambda: permisos limitados a DynamoDB, SES, CloudWatch y Google Calendar (si aplica).
* Cognito para gestión de usuarios y roles del panel administrativo.
* Revisar políticas de privacidad y manejo de datos sensibles (datos personales y de salud). Asegurar cumplimiento local de protección de datos (ej. Leyes aplicables en Colombia).

---

## 9. Consideraciones de despliegue y entorno

* **Entornos:** `dev`, `staging`, `production` (separar tablas y buckets por prefijo/entorno).
* Uso de IaC: **AWS SAM** o **Serverless Framework** para desplegar Lambdas, API Gateway, DynamoDB y otras infra.
* Automatizar despliegue frontend con Amplify CI/CD.

---

## 10. Próximos pasos (para completar la Fase 1)

1. Definir horarios de atención y reglas de disponibilidad (ej. duración por servicio, días no laborables, tiempos de buffer).
2. Especificar los intents y utterances detalladas para Lex (lista de frases de ejemplo).
3. Decidir esquema final de tablas DynamoDB (índices secundarios si se requiere consultas por fecha o por paciente).
4. Preparar credenciales Google Cloud para Calendar y plan de pruebas (sandbox).
5. Preparar diagrama visual (Mermaid o gráfico) para la presentación final.

---

*Documento generado como entregable de la Fase 1 — SmartBooker Dental.*
