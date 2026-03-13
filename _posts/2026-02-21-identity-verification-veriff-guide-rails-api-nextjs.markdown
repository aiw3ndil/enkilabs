---
layout: post
title:  "Identity Verification with Veriff: Integration Guide with Rails API and Next.js"
date:   2026-02-21 11:00:00 +0200
categories: rails nextjs veriff kyc security
---

In the fintech and SaaS application ecosystem, identity verification (KYC) has become a mandatory step. **Veriff** is one of the leading platforms for automating this process using AI.

In this article, I will show you how to orchestrate a complete verification flow using **Ruby on Rails** as an API and **Next.js** on the frontend.

### How does the Veriff flow work?

1.  **Backend (Rails):** Creates a verification "session" in Veriff and obtains a unique URL.
2.  **Frontend (Next.js):** Receives the URL and redirects the user (or opens an iframe) to complete the process (ID photo, selfie, etc.).
3.  **Webhook (Rails):** Veriff notifies your server when the verification has been approved or rejected.

---

## Part 1: The Backend with Ruby on Rails

The backend is the guardian of your secret keys and responsible for processing results securely.

### Step 1: Configure Credentials

You will need your Veriff `API Key` and `Private Key`.

```bash
rails credentials:edit
```

```yaml
# config/credentials.yml.enc
veriff:
  api_key: your_api_key
  private_key: your_private_key
```

### Step 2: Create a Verification Session

We'll create a service to interact with the Veriff API. We'll use the `httparty` gem or simply `Net::HTTP`.

```ruby
# app/services/veriff_service.rb
require 'openssl'

class VeriffService
  BASE_URL = 'https://stationapi.veriff.com/v1'

  def self.create_session(user_id)
    payload = {
      verification: {
        callback: "https://your-api.com/api/v1/veriff/webhooks",
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

### Step 3: Controller and Webhook

It is vital to validate the webhook signature to ensure that the notification actually comes from Veriff.

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

      # Update the status in your DB
      User.find(user_id).update(verification_status: status)

      head :ok
    else
      render json: { error: 'Invalid signature' }, status: :unauthorized
    end
  end
end
```

---

## Part 2: The Frontend with Next.js

In Next.js, we simply need to trigger the session creation and handle the redirection.

### Step 1: Verification Component

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
        // Redirect the user to the Veriff flow
        window.location.href = data.url;
      }
    } catch (error) {
      console.error("Error starting Veriff:", error);
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
      {loading ? 'Preparing...' : 'Verify My Identity'}
    </button>
  );
}
```

### Step 2: Return Screen

Veriff allows you to configure a redirect URL when the user finishes. You can create a page in Next.js (`/verification-status`) that simply says: "We are processing your verification, we will notify you soon."

---

### Conclusion

The **Veriff** integration stands out for its security. By using HMAC signatures for sessions and webhooks, we ensure that identity data cannot be manipulated. Combining this with the robustness of **Rails** and the agility of **Next.js** allows creating high-trust onboarding flows in record time.
