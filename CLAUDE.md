# CLAUDE.md — Plataforma Social para Parejas

## Rol del Agente

Actúa como un desarrollador web experto. Tu misión es construir una plataforma social completa para parejas usando **HTML, CSS y JavaScript puro con Firebase como backend**. El código debe ser simple, legible y fácil de mantener por alguien sin experiencia técnica.

---

## Descripción del Producto

Una red social privada donde **parejas ya formadas** pueden conectar con otras parejas para hacer planes, actividades o amistad. **No es una app de citas** — no hay matches románticos ni swipe. Es una comunidad de networking social entre parejas.

---

## Stack Tecnológico

| Capa | Tecnología |
|------|------------|
| Frontend | HTML + CSS + JavaScript vanilla |
| Backend | Firebase (Firestore + Auth + Hosting) |
| Base de datos | Firebase Firestore |
| Autenticación | Firebase Authentication (email/password) |
| Hosting | Firebase Hosting (gratuito) |

**Sin instalaciones. Sin terminal. Sin frameworks. Sin bundlers.**

---

## Estructura de Archivos

```
/
├── index.html          # Landing page pública
├── login.html          # Inicio de sesión
├── register.html       # Registro
├── onboarding.html     # Configurar perfil de pareja (tras registro)
├── feed.html           # Muro principal del grupo
├── members.html        # Listado de parejas en la comunidad
├── profile.html        # Perfil propio y de otras parejas
├── messages.html       # Mensajes privados
├── admin.html          # Panel de administración (solo superadmin)
├── css/
│   └── styles.css      # Estilos globales
├── js/
│   ├── firebase.js     # Configuración e inicialización de Firebase
│   ├── auth.js         # Login, registro, logout
│   ├── feed.js         # Publicaciones y comentarios
│   ├── members.js      # Listado de miembros
│   ├── profile.js      # Perfil de pareja
│   ├── messages.js     # Mensajería privada
│   └── admin.js        # Funciones del panel admin
└── assets/
    └── avatars/        # Avatares prediseñados (imágenes PNG/SVG)
```

---

## Fase Inicial — Comunidad Cerrada (MUY IMPORTANTE)

- Solo existe **un único grupo principal**, creado manualmente por el superadmin desde el panel admin.
- Las parejas al registrarse **entran automáticamente** a este grupo.
- **No se pueden crear nuevos grupos** en esta fase.
- El objetivo es validar el producto con una única comunidad activa y controlada.

---

## Base de Datos — Colecciones Firestore

### `users`
```
{
  uid: string,
  email: string,
  role: "user" | "superadmin",
  isBlocked: boolean,
  createdAt: timestamp
}
```

### `profiles`
```
{
  uid: string,
  coupleNames: { person1: string, person2: string },
  ages: { person1: number, person2: number },
  location: string,
  interests: array,
  description: string,
  avatar: string,
  createdAt: timestamp
}
```

### `posts`
```
{
  authorUid: string,
  authorName: string,
  authorAvatar: string,
  content: string,
  type: "mensaje" | "plan" | "actividad",
  likes: array,
  createdAt: timestamp,
  isDeleted: boolean
}
```

### `comments`
```
{
  postId: string,
  authorUid: string,
  authorName: string,
  content: string,
  createdAt: timestamp
}
```

### `messages`
```
{
  fromUid: string,
  toUid: string,
  content: string,
  readAt: timestamp | null,
  createdAt: timestamp
}
```

### `connections`
```
{
  requesterUid: string,
  recipientUid: string,
  status: "pending" | "accepted" | "rejected",
  createdAt: timestamp
}
```

---

## Páginas y Funcionalidades

### `index.html` — Landing
- Presentación de la plataforma
- Botones de "Entrar" y "Registrarse"
- Sin acceso a contenido sin login

### `login.html` — Login
- Formulario email + contraseña
- Firebase Authentication
- Redirige a `feed.html` si ya está logado

### `register.html` — Registro
- Formulario email + contraseña
- Crea usuario en Firebase Auth
- Crea documento en colección `users`
- Redirige a `onboarding.html`

### `onboarding.html` — Perfil de pareja
- Nombres de ambos (opcional)
- Edades de ambos
- Ubicación
- Intereses (etiquetas seleccionables)
- Descripción
- Selección de avatar (galería de avatares prediseñados)
- Guarda en colección `profiles`
- Redirige a `feed.html`

### `feed.html` — Muro principal
- Lista de publicaciones ordenadas por fecha
- Formulario para crear publicación (tipo: mensaje / plan / actividad)
- Likes y comentarios en cada post
- Navbar con acceso a todas las secciones

### `members.html` — Miembros
- Listado de todas las parejas registradas
- Tarjeta con avatar, nombre, ubicación, intereses
- Botón para enviar solicitud de conexión
- Botón para ver perfil completo

### `profile.html` — Perfil
- Perfil propio: editable
- Perfil ajeno: ver info + conectar + mensaje si conectados

### `messages.html` — Mensajes
- Lista de conversaciones activas
- Chat en tiempo real con Firestore (onSnapshot)
- Solo disponible entre parejas conectadas

### `admin.html` — Panel Admin
- Solo accesible si `role === "superadmin"`
- Listado de usuarios con opciones: bloquear / desbloquear / eliminar
- Listado de publicaciones con opción de eliminar
- Configurar nombre y descripción del grupo principal
- Vista de actividad reciente

---

## Diseño

- **Colores**: blanco `#FFFFFF`, negro `#111111`, grises `#F5F5F5` / `#9E9E9E`, azul `#2563EB`
- **Tipografía**: Inter (Google Fonts, gratis)
- **Estilo**: minimalista, limpio, tipo red social profesional
- **Responsive**: funciona en móvil y escritorio
- **Sin librerías CSS externas** — solo el archivo `styles.css`

---

## Sistema de Avatares

- Galería de avatares ilustrados prediseñados (sin fotos reales)
- El usuario elige uno al hacer onboarding
- Se guarda la URL/nombre del avatar en Firestore
- Usar avatares de DiceBear generados como URLs (no requiere descarga)

Ejemplo de URL de avatar DiceBear:
```
https://api.dicebear.com/7.x/fun-emoji/svg?seed=NombrePareja
```

---

## Configuración de Firebase (paso a paso)

1. Ir a firebase.google.com e iniciar sesión con Google
2. Crear nuevo proyecto
3. Activar **Authentication** → método Email/Password
4. Crear **Firestore Database** en modo producción
5. Activar **Hosting**
6. Copiar la configuración del proyecto en `js/firebase.js`:

```js
// js/firebase.js
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.0.0/firebase-app.js";
import { getAuth } from "https://www.gstatic.com/firebasejs/10.0.0/firebase-auth.js";
import { getFirestore } from "https://www.gstatic.com/firebasejs/10.0.0/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "TU_API_KEY",
  authDomain: "TU_PROYECTO.firebaseapp.com",
  projectId: "TU_PROYECTO",
  storageBucket: "TU_PROYECTO.appspot.com",
  messagingSenderId: "TU_ID",
  appId: "TU_APP_ID"
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
export const db = getFirestore(app);
```

---

## Crear el Superadmin

1. Registrarse normalmente en `register.html`
2. Ir a Firestore → colección `users` → buscar el documento con tu UID
3. Cambiar el campo `role` de `"user"` a `"superadmin"` manualmente desde la consola de Firebase
4. Ya tienes acceso a `admin.html`

---

## Reglas de Seguridad Firestore (copiar y pegar)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /profiles/{uid} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == uid;
    }

    match /posts/{postId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null;
      allow update, delete: if request.auth.uid == resource.data.authorUid;
    }

    match /messages/{msgId} {
      allow read, write: if request.auth.uid == resource.data.fromUid
                         || request.auth.uid == resource.data.toUid;
    }

    match /connections/{connId} {
      allow read, write: if request.auth != null;
    }

    match /users/{uid} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == uid;
    }
  }
}
```

---

## Despliegue

Opción sin terminal: subir los archivos directamente desde la consola de Firebase Hosting con drag & drop. La plataforma queda disponible en `https://TU-PROYECTO.web.app` de forma gratuita.

---

## Restricciones Importantes de Esta Fase

1. **Solo existe un grupo** — no hay UI para crear grupos
2. **Todo usuario registrado entra automáticamente** a la comunidad
3. **Los mensajes privados solo funcionan entre parejas conectadas**
4. **El panel admin solo es visible** si `role === "superadmin"`
5. **Sin fotos reales** — solo avatares ilustrados
