---
layout: post
title:  "Integrando Stripe: Guía Completa con Rails API y Next.js"
date:   2026-02-16 11:00:00 +0200
categories: rails nextjs stripe javascript
---

En el desarrollo de aplicaciones web modernas, es común separar el backend del frontend. Una combinación poderosa para esto es usar Ruby on Rails como una API robusta y Next.js para un frontend dinámico y reactivo. En esta guía, te mostraré cómo integrar Stripe para procesar pagos en una arquitectura de este tipo, permitiéndote aceptar pagos con tarjeta de crédito de forma segura.

### ¿Qué vamos a construir?

Crearemos un flujo de pago simple donde:
1.  El frontend (Next.js) mostrará un formulario para ingresar los datos de la tarjeta.
2.  El frontend solicitará un `PaymentIntent` al backend (Rails).
3.  El backend creará un `PaymentIntent` en Stripe y devolverá su `client_secret` al frontend.
4.  El frontend usará el `client_secret` para confirmar el pago directamente con Stripe.

### Prerrequisitos

- Ruby y Rails instalados.
- Node.js y Next.js instalados.
- Una cuenta de Stripe (puedes usar el modo de prueba).

---

## Parte 1: Configurando el Backend con Ruby on Rails

Nuestro backend se encargará de la comunicación segura con la API de Stripe para crear la intención de pago.

### Paso 1: Añadir la gema de Stripe

Primero, agrega la gema oficial de Stripe a tu `Gemfile`:

```ruby
# Gemfile
gem 'stripe'
```

Luego, ejecuta `bundle install` en tu terminal.

### Paso 2: Configurar las Claves de API

Es crucial mantener tus claves de Stripe seguras. Utiliza las credenciales de Rails para almacenarlas.

```bash
# En tu terminal
rails credentials:edit
```

Añade tus claves de Stripe (las encontrarás en tu Dashboard de Stripe):

```yaml
# config/credentials.yml.enc
stripe:
  api_key: sk_test_...
  publishable_key: pk_test_...
```

Ahora, crea un inicializador para que Stripe use estas claves al arrancar la aplicación.

```ruby
# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.stripe[:api_key]
```

### Paso 3: Crear el Endpoint para PaymentIntents

Necesitamos una ruta y un controlador para crear los `PaymentIntents`.

Primero, define la ruta en `config/routes.rb`:

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    post 'create_payment_intent', to: 'payments#create'
  end
end
```

Ahora, crea el controlador y la acción.

```ruby
# app/controllers/api/v1/payments_controller.rb
class Api::V1::PaymentsController < ApplicationController
  def create
    # Por ahora, usaremos un monto fijo. En una app real,
    # este valor vendría de los parámetros o de la base de datos.
    payment_intent = Stripe::PaymentIntent.create(
      amount: 1000, # en centavos, ej: 10.00 USD
      currency: 'usd',
      automatic_payment_methods: {
        enabled: true,
      },
    )

    render json: {
      clientSecret: payment_intent.client_secret
    }
  rescue Stripe::StripeError => e
    render json: { error: e.message }, status: :unprocessable_entity
  end
end
```

### Paso 4: Configurar CORS

Como tu frontend y backend corren en dominios diferentes (incluso en `localhost`), necesitas configurar CORS (Cross-Origin Resource Sharing).

Añade la gema `rack-cors` a tu `Gemfile` y ejecuta `bundle install`.

```ruby
# Gemfile
gem 'rack-cors'
```

Luego, configura CORS en `config/initializers/cors.rb`:

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'http://localhost:3000' # La URL de tu app de Next.js
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

¡Listo! Tu API de Rails ya está preparada para crear `PaymentIntents` para tu frontend.

---

## Parte 2: Configurando el Frontend con Next.js

Ahora vamos a construir el formulario de pago en nuestra aplicación de Next.js.

### Paso 1: Instalar las Librerías de Stripe

Necesitarás las librerías de Stripe para React.

```bash
npm install @stripe/react-stripe-js @stripe/stripe-js
```

### Paso 2: Inicializar Stripe

Para usar los componentes y hooks de Stripe, debes envolver tu aplicación (o al menos tu formulario de pago) con el componente `Elements`.

Primero, carga tu clave publicable de Stripe. Es seguro exponer esta clave en el frontend.

```javascript
// lib/stripe.js
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY);

export default stripePromise;
```

Asegúrate de añadir tu clave publicable a un archivo `.env.local`:

```
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
```

### Paso 3: Crear el Formulario de Pago

Ahora crearemos un componente `CheckoutForm` que se conectará a nuestra API de Rails y gestionará el pago.

```jsx
// components/CheckoutForm.js
import { PaymentElement, useStripe, useElements } from '@stripe/react-stripe-js';
import { useState } from 'react';

export default function CheckoutForm() {
  const stripe = useStripe();
  const elements = useElements();
  const [message, setMessage] = useState(null);
  const [isProcessing, setIsProcessing] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();

    if (!stripe || !elements) {
      return;
    }

    setIsProcessing(true);

    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/completion`,
      },
    });

    if (error) {
      setMessage(error.message);
    }

    setIsProcessing(false);
  };

  return (
    <form id="payment-form" onSubmit={handleSubmit}>
      <PaymentElement id="payment-element" />
      <button disabled={isProcessing || !stripe || !elements} id="submit">
        <span id="button-text">
          {isProcessing ? "Procesando..." : "Pagar ahora"}
        </span>
      </button>
      {message && <div id="payment-message">{message}</div>}
    </form>
  );
}
```

### Paso 4: Juntando todo en una Página

Finalmente, crea una página que obtenga el `clientSecret` de tu API de Rails y renderice el `CheckoutForm` dentro del proveedor `Elements`.

```jsx
// pages/pay.js
import { Elements } from '@stripe/react-stripe-js';
import { useEffect, useState } from 'react';
import CheckoutForm from '../components/CheckoutForm';
import stripePromise from '../lib/stripe';

export default function PayPage() {
  const [clientSecret, setClientSecret] = useState('');

  useEffect(() => {
    // Llama a tu backend para crear el PaymentIntent
    fetch('http://localhost:3001/api/v1/create_payment_intent', { // Asegúrate que el puerto es el de tu API de Rails
      method: 'POST',
    })
      .then((res) => res.json())
      .then((data) => setClientSecret(data.clientSecret));
  }, []);

  const options = {
    clientSecret,
  };

  return (
    <div>
      <h1>Realizar Pago</h1>
      {clientSecret && (
        <Elements options={options} stripe={stripePromise}>
          <CheckoutForm />
        </Elements>
      )}
    </div>
  );
}
```

No olvides crear la página de `completion` a la que Stripe redirigirá al usuario.

```jsx
// pages/completion.js
import { useStripe } from '@stripe/react-stripe-js';
import { useEffect, useState } from 'react';

export default function CompletionPage() {
    // NOTA: Esta página es muy simple. Para una implementación completa,
    // envuelve esta página también con `Elements` y `stripePromise`.
    return <h2>¡Pago completado! Gracias por tu compra.</h2>;
}
```

### Conclusión

¡Y eso es todo! Has configurado con éxito un flujo de pago con Stripe usando una API de Rails y un frontend de Next.js. Esta arquitectura desacoplada te da una gran flexibilidad. Desde aquí, puedes explorar funcionalidades más avanzadas como **webhooks** para confirmar los pagos de forma asíncrona en tu backend, gestionar suscripciones o guardar métodos de pago de clientes.
