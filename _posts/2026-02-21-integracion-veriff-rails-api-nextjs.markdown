---
layout: post
title:  "Verificación de Identidad con Veriff: Guía de Integración con Rails API y Next.js"
date:   2026-02-21 11:00:00 +0000
categories: rails nextjs veriff kyc security
---

En el ecosistema fintech y de aplicaciones SaaS, la verificación de identidad (KYC) se ha vuelto un paso obligatorio. **Veriff** es una de las plataformas líderes para automatizar este proceso mediante IA.

En este artículo, te mostraré cómo orquestar un flujo de verificación completo utilizando **Ruby on Rails** como API y **Next.js** en el frontend.

### ¿Cómo funciona el flujo de Veriff?

1.  **Backend (Rails):** Crea una "sesión" de verificación en Veriff y obtiene una URL única.
2.  **Frontend (Next.js):** Recibe la URL y redirige al usuario (o abre un iframe) para que complete el proceso (foto de ID, selfie, etc.).
3.  **Webhook (Rails):** Veriff notifica a tu servidor cuando la verificación ha sido aprobada o rechazada.

---

## Parte 1: El Backend con Ruby on Rails

El backend es el guardián de tus claves secretas y el encargado de procesar los resultados de forma segura.

### Paso 1: Configurar Credenciales

Necesitarás tu `API Key` y tu `Private Key` de Veriff.

```bash
rails credentials:edit
```

```yaml
# config/credentials.yml.enc
veriff:
  api_key: your_api_key
  private_key: your_private_key
```

### Paso 2: Crear una Sesión de Verificación

Crearemos un servicio para interactuar con la API de Veriff. Usaremos la gema `httparty` o simplemente `Net::HTTP`.

```ruby
# app/services/veriff_service.rb
require 'openssl'

class VeriffService
  BASE_URL = 'https://stationapi.veriff.com/v1'

  def self.create_session(user_id)
    payload = {
      verification: {
        callback: "https://tu-api.com/api/v1/veriff/webhooks",
        vendorData: user_id.to_s
      }
    }.to_json

    signature = generate_signature(payload)

    response = HTTParty.post(
      "#{BASE_URL}/sessions",
      body: payload,
      headers: {
        'Content-Type' => 'application/json',
        'x-auth-client' => Rails.application.credentials.veriff[:api_key],
        'x-signature' => signature
      }
    )

    JSON.parse(response.body)
  end

  def self.generate_signature(payload)
    OpenSSL::HMAC.hexdigest('sha256', Rails.application.credentials.veriff[:private_key], payload)
  end
end
```

### Paso 3: Controlador y Webhook

Es vital validar la firma del webhook para asegurarte de que la notificación realmente viene de Veriff.

```ruby
# app/controllers/api/v1/veriff_controller.rb
class Api::V1::VeriffController < ApplicationController
  def create
    session_data = VeriffService.create_session(current_user.id)
    render json: { url: session_data.dig('verification', 'url') }
  end

  def webhook
    payload = request.raw_post
    signature = request.headers['x-signature']

    if VeriffService.generate_signature(payload) == signature
      data = JSON.parse(payload)
      status = data.dig('verification', 'status')
      user_id = data.dig('verification', 'vendorData')

      # Actualiza el estado en tu DB
      User.find(user_id).update(verification_status: status)

      head :ok
    else
      render json: { error: 'Invalid signature' }, status: :unauthorized
    end
  end
end
```

---

## Parte 2: El Frontend con Next.js

En Next.js, simplemente necesitamos disparar la creación de la sesión y manejar la redirección.

### Paso 1: Componente de Verificación

```jsx
// components/VeriffButton.js
import { useState } from 'react';

export default function VeriffButton() {
  const [loading, setLoading] = useState(false);

  const handleVerify = async () => {
    setLoading(true);
    try {
      const res = await fetch('http://localhost:3001/api/v1/veriff/create', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('token')}`,
          'Content-Type': 'application/json'
        }
      });
      const data = await res.json();

      if (data.url) {
        // Redirige al usuario al flujo de Veriff
        window.location.href = data.url;
      }
    } catch (error) {
      console.error("Error iniciando Veriff:", error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <button
      onClick={handleVerify}
      disabled={loading}
      className="bg-black text-white px-6 py-3 rounded-lg font-bold"
    >
      {loading ? 'Preparando...' : 'Verificar mi Identidad'}
    </button>
  );
}
```

### Paso 2: Pantalla de Retorno

Veriff te permite configurar una URL de redirección cuando el usuario termina. Puedes crear una página en Next.js (`/verification-status`) que simplemente diga: "Estamos procesando tu verificación, te avisaremos pronto".

---

### Conclusión

La integración de **Veriff** destaca por su seguridad. Al utilizar firmas HMAC para las sesiones y los webhooks, garantizamos que los datos de identidad no puedan ser manipulados. Combinar esto con la robustez de **Rails** y la agilidad de **Next.js** permite crear flujos de onboarding de alta confianza en tiempo récord.
