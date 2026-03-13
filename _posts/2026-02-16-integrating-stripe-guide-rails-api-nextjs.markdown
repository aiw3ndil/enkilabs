---
layout: post
title:  "Integrating Stripe: Complete Guide with Rails API and Next.js"
date:   2026-02-16 11:00:00 +0200
categories: rails nextjs stripe javascript
---

In modern web application development, it is common to separate the backend from the frontend. A powerful combination for this is using Ruby on Rails as a robust API and Next.js for a dynamic and reactive frontend. In this guide, I will show you how to integrate Stripe to process payments in such an architecture, allowing you to securely accept credit card payments.

### What are we going to build?

We will create a simple payment flow where:
1.  The frontend (Next.js) will show a form to enter card details.
2.  The frontend will request a `PaymentIntent` from the backend (Rails).
3.  The backend will create a `PaymentIntent` in Stripe and return its `client_secret` to the frontend.
4.  The frontend will use the `client_secret` to confirm the payment directly with Stripe.

### Prerequisites

- Ruby and Rails installed.
- Node.js and Next.js installed.
- A Stripe account (you can use test mode).

---

## Part 1: Configuring the Backend with Ruby on Rails

Our backend will handle secure communication with the Stripe API to create the payment intent.

### Step 1: Add the Stripe gem

First, add the official Stripe gem to your `Gemfile`:

```ruby
# Gemfile
gem 'stripe'
```

Then, run `bundle install` in your terminal.

### Step 2: Configure API Keys

It is crucial to keep your Stripe keys secure. Use Rails credentials to store them.

```bash
# In your terminal
rails credentials:edit
```

Add your Stripe keys (you will find them in your Stripe Dashboard):

```yaml
# config/credentials.yml.enc
stripe:
  api_key: sk_test_...
  publishable_key: pk_test_...
```

Now, create an initializer so that Stripe uses these keys when the application starts.

```ruby
# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.stripe[:api_key]
```

### Step 3: Create the Endpoint for PaymentIntents

We need a route and a controller to create `PaymentIntents`.

First, define the route in `config/routes.rb`:

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    post 'create_payment_intent', to: 'payments#create'
  end
end
```

Now, create the controller and the action.

```ruby
# app/controllers/api/v1/payments_controller.rb
class Api::V1::PaymentsController < ApplicationController
  def create
    # For now, we'll use a fixed amount. In a real app,
    # this value would come from parameters or the database.
    payment_intent = Stripe::PaymentIntent.create(
      amount: 1000, # in cents, e.g., 10.00 USD
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

### Step 4: Configure CORS

Since your frontend and backend run on different domains (even on `localhost`), you need to configure CORS (Cross-Origin Resource Sharing).

Add the `rack-cors` gem to your `Gemfile` and run `bundle install`.

```ruby
# Gemfile
gem 'rack-cors'
```

Then, configure CORS in `config/initializers/cors.rb`:

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'http://localhost:3000' # Your Next.js app URL
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

Ready! Your Rails API is now prepared to create `PaymentIntents` for your frontend.

---

## Part 2: Configuring the Frontend with Next.js

Now we are going to build the payment form in our Next.js application.

### Step 1: Install Stripe Libraries

You will need the Stripe libraries for React.

```bash
npm install @stripe/react-stripe-js @stripe/stripe-js
```

### Step 2: Initialize Stripe

To use Stripe components and hooks, you must wrap your application (or at least your payment form) with the `Elements` component.

First, load your Stripe publishable key. It is safe to expose this key on the frontend.

```javascript
// lib/stripe.js
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY);

export default stripePromise;
```

Make sure to add your publishable key to a `.env.local` file:

```
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
```

### Step 3: Create the Payment Form

Now we will create a `CheckoutForm` component that will connect to our Rails API and manage the payment.

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
          {isProcessing ? "Processing..." : "Pay now"}
        </span>
      </button>
      {message && <div id="payment-message">{message}</div>}
    </form>
  );
}
```

### Step 4: Putting it all together on a Page

Finally, create a page that gets the `clientSecret` from your Rails API and renders the `CheckoutForm` inside the `Elements` provider.

```jsx
// pages/pay.js
import { Elements } from '@stripe/react-stripe-js';
import { useEffect, useState } from 'react';
import CheckoutForm from '../components/CheckoutForm';
import stripePromise from '../lib/stripe';

export default function PayPage() {
  const [clientSecret, setClientSecret] = useState('');

  useEffect(() => {
    // Call your backend to create the PaymentIntent
    fetch('http://localhost:3001/api/v1/create_payment_intent', { // Make sure the port is your Rails API port
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
      <h1>Make Payment</h1>
      {clientSecret && (
        <Elements options={options} stripe={stripePromise}>
          <CheckoutForm />
        </Elements>
      )}
    </div>
  );
}
```

Don't forget to create the `completion` page that Stripe will redirect the user to.

```jsx
// pages/completion.js
import { useStripe } from '@stripe/react-stripe-js';
import { useEffect, useState } from 'react';

export default function CompletionPage() {
    // NOTE: This page is very simple. For a full implementation,
    // wrap this page also with `Elements` and `stripePromise`.
    return <h2>Payment completed! Thank you for your purchase.</h2>;
}
```

### Conclusion

And that's it! You have successfully configured a payment flow with Stripe using a Rails API and a Next.js frontend. This decoupled architecture gives you great flexibility. From here, you can explore more advanced features like **webhooks** to confirm payments asynchronously on your backend, manage subscriptions, or save customer payment methods.
