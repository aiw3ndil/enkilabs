---
layout: post
title:  "Google Auth: Implementación Completa con Rails API y Next.js"
date:   2026-03-07 08:00:00 +0200
categories: rails nextjs auth google security
---

Implementar autenticación social es una de las mejores formas de mejorar la experiencia de usuario (UX) al eliminar la fricción de crear una cuenta manual. En una arquitectura desacoplada con **Ruby on Rails** como API y **Next.js** en el frontend, el flujo de autenticación con Google requiere una coordinación precisa para mantener la seguridad.

En esta guía, aprenderás a implementar un flujo de Google Auth donde el frontend gestiona el login y el backend valida la identidad de forma segura.

### El Flujo de Autenticación

1.  **Frontend (Next.js):** El usuario inicia sesión con el botón de Google y recibe un `id_token` (JWT).
2.  **Backend (Rails):** Recibe el `id_token`, lo verifica con los servidores de Google y busca (o crea) al usuario en la base de datos.
3.  **Backend (Rails):** Emite un token propio (JWT) para las sesiones posteriores del usuario.

---

## Parte 1: El Backend con Ruby on Rails

Nuestro objetivo en Rails es verificar que el token enviado por el frontend es legítimo y pertenece a nuestra aplicación.

### Paso 1: Instalar Dependencias

Añade estas gemas a tu `Gemfile`:

```ruby
# Gemfile
gem 'google-id-token' # Para validar el token de Google
gem 'jwt'            # Para emitir nuestros propios tokens de sesión
```

Ejecuta `bundle install`.

### Paso 2: Configurar las Credenciales

Necesitarás tu `Client ID` de Google (obtenido en la Google Cloud Console).

```bash
rails credentials:edit
```

```yaml
# config/credentials.yml.enc
google:
  client_id: "tu_client_id_de_google.apps.googleusercontent.com"
jwt_secret: "un_secreto_muy_largo_y_seguro"
```

### Paso 3: Servicio de Verificación de Google

Crearemos un servicio para encapsular la lógica de validación.

```ruby
# app/services/google_auth_service.rb
class GoogleAuthService
  def self.validate(id_token)
    validator = GoogleIDToken::Validator.new
    begin
      payload = validator.check(id_token, Rails.application.credentials.google[:client_id])
      payload # Retorna los datos del usuario si es válido (email, nombre, etc.)
    rescue GoogleIDToken::ValidationError => e
      nil
    end
  end
end
```

### Paso 4: El Controlador de Autenticación

Este controlador recibirá el token y gestionará el login.

```ruby
# app/controllers/api/v1/auth_controller.rb
class Api::V1::AuthController < ApplicationController
  def google
    payload = GoogleAuthService.validate(params[:id_token])

    if payload
      user = User.find_or_create_by(email: payload['email']) do |u|
        u.name = payload['name']
        u.password = SecureRandom.hex(16) # Contraseña aleatoria para usuarios de Google
      end

      # Generamos nuestro JWT de sesión
      token = JWT.encode({ user_id: user.id, exp: 24.hours.from_now.to_i }, Rails.application.credentials.jwt_secret)

      render json: { token: token, user: user }
    else
      render json: { error: 'Token de Google inválido' }, status: :unauthorized
    end
  end
end
```

---

## Parte 2: El Frontend con Next.js

Usaremos la librería oficial `@react-oauth/google` para simplificar la integración con el SDK de Google.

### Paso 1: Instalación

```bash
npm install @react-oauth/google
```

### Paso 2: Configurar el Proveedor

Envuelve tu aplicación (normalmente en `_app.js` o `layout.tsx`) con el `GoogleOAuthProvider`.

```jsx
// pages/_app.js
import { GoogleOAuthProvider } from '@react-oauth/google';

function MyApp({ Component, pageProps }) {
  return (
    <GoogleOAuthProvider clientId={process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID}>
      <Component {...pageProps} />
    </GoogleOAuthProvider>
  );
}

export default MyApp;
```

### Paso 3: El Componente de Login

Ahora implementamos el botón que enviará el token a nuestro backend.

```jsx
// components/GoogleLoginButton.js
import { GoogleLogin } from '@react-oauth/google';

export default function GoogleLoginButton() {
  const handleSuccess = async (credentialResponse) => {
    const res = await fetch('http://localhost:3001/api/v1/auth/google', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ id_token: credentialResponse.credential })
    });

    const data = await res.json();

    if (data.token) {
      localStorage.setItem('token', data.token);
      window.location.href = '/dashboard';
    } else {
      alert('Error en el login');
    }
  };

  return (
    <GoogleLogin
      onSuccess={handleSuccess}
      onError={() => console.log('Login Failed')}
    />
  );
}
```

---

### Consideraciones de Seguridad

1.  **Validación en el Servidor:** Nunca confíes solo en el frontend. El backend **debe** verificar el `id_token` con Google para asegurar que no ha sido manipulado.
2.  **HTTPS:** En producción, tanto tu API como tu frontend deben usar HTTPS para proteger los tokens en tránsito.
3.  **CORS:** Asegúrate de que tu API de Rails permita peticiones desde el dominio de tu frontend (configurado en `config/initializers/cors.rb`).

### Conclusión

Combinar la potencia de **Rails** para la lógica de negocio y la agilidad de **Next.js** para la interfaz nos permite construir sistemas de autenticación robustos y modernos. Al delegar la verificación del token de Google a nuestro backend, garantizamos un nivel de seguridad empresarial sin sacrificar la facilidad de uso para nuestros usuarios.
