# ADMIN.md — Panel de Superadministración · Dúos

## Descripción

`admin.html` es el panel de control exclusivo para la superadministradora de la comunidad Dúos. Desde aquí se gestionan todas las solicitudes de inscripción, los miembros activos y se consulta el historial de actividad del grupo.

**Acceso restringido**: solo visible para usuarios con `role === "superadmin"` en Firestore. Cualquier otro usuario que intente acceder debe ser redirigido a `feed.html`.

---

## Estructura de la página

```
admin.html
├── Sidebar fijo (izquierda)
│   ├── Logo → enlaza a index.html
│   ├── Navegación: Solicitudes / Miembros / Actividad
│   └── Footer: nombre admin + cerrar sesión
├── Topbar
│   ├── Título de la sección activa
│   └── Fecha actual
├── Métricas (4 tarjetas)
└── Secciones (una visible a la vez)
    ├── Solicitudes pendientes
    ├── Miembros activos
    └── Actividad reciente
```

---

## Secciones

### 1. Métricas (siempre visibles)

Cuatro tarjetas en la parte superior con datos en tiempo real:

| Métrica | Fuente Firestore |
|---|---|
| Miembros activos | `users` donde `isBlocked == false` y `role == "user"` |
| Solicitudes pendientes | `solicitudes` donde `status == "pending"` |
| Aprobadas total | `solicitudes` donde `status == "approved"` |
| Rechazadas | `solicitudes` donde `status == "rejected"` |

---

### 2. Solicitudes pendientes

Lista de parejas que han rellenado el formulario de inscripción y están esperando aprobación.

**Datos que se muestran por solicitud:**
- Nombres de la pareja
- Edades de cada uno
- Zona de Madrid
- Email de contacto
- Fecha/hora de la solicitud
- Estado actual (pendiente / aprobada / rechazada)

**Acciones disponibles:**

| Acción | Resultado |
|---|---|
| ✓ Aprobar | Cambia `status` a `"approved"` en Firestore · Crea documento en `users` y `profiles` · Notifica por Telegram |
| ✗ Rechazar | Cambia `status` a `"rejected"` en Firestore · Notifica por Telegram (opcional) |
| Ver detalle | Abre modal con todos los datos de la solicitud incluyendo descripción libre |

**Colección Firestore:** `solicitudes`

```js
{
  nombres: { persona1: string, persona2: string },
  edades:  { persona1: number, persona2: number },
  zona:    string,
  email:   string,
  descripcion: string,          // texto libre del formulario (si se añade)
  status:  "pending" | "approved" | "rejected",
  creadoEn: timestamp,
  revisadoEn: timestamp | null
}
```

---

### 3. Miembros activos

Lista de todas las parejas aprobadas y con acceso a la comunidad.

**Datos por miembro:**
- Inicial + nombre de la pareja
- Edades y zona de Madrid
- Fecha de incorporación
- Estado: activo / bloqueado

**Acciones disponibles:**

| Acción | Resultado |
|---|---|
| Ver perfil | Navega a `profile.html?uid=XXX` |
| Bloquear | Cambia `isBlocked` a `true` en `users` · El usuario pierde acceso al feed |
| Desbloquear | Cambia `isBlocked` a `false` |
| Eliminar | Elimina documentos en `users` y `profiles` · Requiere confirmación |

**Colección Firestore:** `users` + `profiles`

---

### 4. Actividad reciente

Registro cronológico de las últimas acciones en la plataforma:
- Nuevas solicitudes recibidas
- Aprobaciones y rechazos
- Incorporaciones al grupo
- Bloqueos o eliminaciones

**Colección Firestore sugerida:** `actividad`

```js
{
  tipo:    "solicitud" | "aprobacion" | "rechazo" | "bloqueo" | "baja",
  nombres: string,
  uid:     string | null,
  mensaje: string,
  creadoEn: timestamp
}
```

---

## Integración con Telegram Bot

Cuando se aprueba o rechaza una solicitud, el panel debe enviar un mensaje automático al bot de Telegram de la administradora.

**Cuándo se dispara:**
- Al aprobar: `"✅ Nueva pareja aprobada: [nombres] ([zona])"`
- Al rechazar: `"❌ Solicitud rechazada: [nombres]"`
- Al recibir una nueva solicitud (desde `register.html`): `"📩 Nueva solicitud de [nombres], [edades], [zona]"`

**Implementación:**

```js
async function notificarTelegram(mensaje) {
  const TOKEN   = 'TU_BOT_TOKEN';
  const CHAT_ID = 'TU_CHAT_ID';
  await fetch(`https://api.telegram.org/bot${TOKEN}/sendMessage`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ chat_id: CHAT_ID, text: mensaje, parse_mode: 'HTML' })
  });
}
```

**Cómo obtener el Bot Token y Chat ID:**
1. Abrir Telegram → buscar `@BotFather`
2. Escribir `/newbot` → seguir los pasos → copiar el token
3. Escribir al bot un mensaje cualquiera
4. Abrir en el navegador: `https://api.telegram.org/botTU_TOKEN/getUpdates`
5. Copiar el valor de `"id"` dentro de `"chat"` — ese es el Chat ID

---

## Integración con Firebase

### Leer solicitudes pendientes

```js
import { collection, query, where, onSnapshot } from "firebase/firestore";

const q = query(collection(db, "solicitudes"), where("status", "==", "pending"));
onSnapshot(q, (snapshot) => {
  snapshot.forEach(doc => renderSolicitud(doc.id, doc.data()));
});
```

### Aprobar una solicitud

```js
import { doc, updateDoc, setDoc, serverTimestamp } from "firebase/firestore";

async function aprobar(solicitudId, datos) {
  // 1. Actualizar estado en solicitudes
  await updateDoc(doc(db, "solicitudes", solicitudId), {
    status: "approved",
    revisadoEn: serverTimestamp()
  });

  // 2. Crear usuario en Firestore
  await setDoc(doc(db, "users", datos.uid), {
    uid: datos.uid,
    email: datos.email,
    role: "user",
    isBlocked: false,
    createdAt: serverTimestamp()
  });

  // 3. Crear perfil
  await setDoc(doc(db, "profiles", datos.uid), {
    uid: datos.uid,
    coupleNames: datos.nombres,
    ages: datos.edades,
    location: datos.zona,
    createdAt: serverTimestamp()
  });

  // 4. Notificar Telegram
  await notificarTelegram(`✅ Nueva pareja aprobada: ${datos.nombres.persona1} & ${datos.nombres.persona2} (${datos.zona})`);
}
```

### Rechazar una solicitud

```js
async function rechazar(solicitudId, datos) {
  await updateDoc(doc(db, "solicitudes", solicitudId), {
    status: "rejected",
    revisadoEn: serverTimestamp()
  });

  await notificarTelegram(`❌ Solicitud rechazada: ${datos.nombres.persona1} & ${datos.nombres.persona2}`);
}
```

### Bloquear un miembro

```js
async function bloquear(uid) {
  await updateDoc(doc(db, "users", uid), { isBlocked: true });
}
```

---

## Reglas de seguridad Firestore para solicitudes

Añadir a las reglas existentes:

```
match /solicitudes/{solicitudId} {
  allow create: if request.auth != null;
  allow read, update, delete: if get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "superadmin";
}

match /actividad/{actividadId} {
  allow read, write: if get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "superadmin";
}
```

---

## Flujo completo de una inscripción

```
1. Pareja rellena register.html
        ↓
2. Se guarda en Firestore (solicitudes · status: "pending")
        ↓
3. Telegram avisa a la admin: "📩 Nueva solicitud de..."
        ↓
4. Admin entra a admin.html → ve la solicitud
        ↓
5a. APRUEBA → se crean documentos en users + profiles
             → Telegram confirma aprobación
             → La pareja puede entrar con login.html
        ↓
5b. RECHAZA → status: "rejected"
             → La pareja no puede acceder
```

---

## Diseño y estilos

- **Colores**: misma paleta que el resto del proyecto (azul `#2C4A7C`, marrón `#8B6343`, crema `#F7F2EC`)
- **Sidebar**: fondo azul `#2C4A7C` con navegación por secciones
- **Tipografía**: Playfair Display (títulos) + Jost (cuerpo)
- **Estados visuales**:
  - Pendiente: fondo amarillo suave `#FEF9EC`
  - Aprobada: fondo verde suave `#EAF5F0`
  - Rechazada: fondo rojo suave `#FDF0EF`
- **Modal de detalle**: se abre al pulsar "Ver detalle" en cualquier solicitud
- **Responsive**: el sidebar se oculta en móvil (< 900px)

---

## Pendiente de implementar

- [ ] Conectar con Firebase Auth para proteger el acceso (redirigir si no es superadmin)
- [ ] Leer solicitudes reales desde Firestore con `onSnapshot`
- [ ] Botones de aprobar/rechazar que escriban en Firestore
- [ ] Integrar el Bot de Telegram con token real
- [ ] Buscador/filtro de miembros
- [ ] Exportar lista de miembros a CSV
- [ ] Enviar email de bienvenida al aprobar (via EmailJS o similar)
