---
layout: post
title:  "Google Auth: Full Implementation with Rails API and Next.js"
date:   2026-03-07 08:00:00 +0200
categories: rails nextjs auth google security
---

Implementing social authentication is one of the best ways to improve user experience (UX) by removing the friction of manual account creation. In a decoupled architecture with **Ruby on Rails** as an API and **Next.js** on the frontend, the Google authentication flow requires precise coordination to maintain security.

In this guide, you will learn to implement a Google Auth flow where the frontend manages the login and the backend securely validates the identity.

### The Authentication Flow

1.  **Frontend (Next.js):** The user logs in with the Google button and receives an `id_token` (JWT).
2.  **Backend (Rails):** Receives the `id_token`, verifies it with Google's servers, and searches for (or creates) the user in the database.
3.  **Backend (Rails):** Issues its own token (JWT) for the user's subsequent sessions.

---

## Part 1: The Backend with Ruby on Rails

Our goal in Rails is to verify that the token sent by the frontend is legitimate and belongs to our application.

### Step 1: Install Dependencies

Add these gems to your `Gemfile`:

```ruby
# Gemfile
gem 'google-id-token' # To validate Google's token
gem 'jwt'            # To issue our own session tokens
```

Run `bundle install`.

### Step 2: Configure Credentials

You will need your Google `Client ID` (obtained in the Google Cloud Console).

```bash
rails credentials:edit
```

```yaml
# config/credentials.yml.enc
google:
  client_id: "your_google_client_id.apps.googleusercontent.com"
jwt_secret: "a_very_long_and_secure_secret"
```

### Step 3: Google Verification Service

We'll create a service to encapsulate the validation logic.

```ruby
# app/services/google_auth_service.rb
class GoogleAuthService
  def self.validate(id_token)
    validator = GoogleIDToken::Validator.new
    begin
      payload = validator.check(id_token, Rails.application.credentials.google[:client_id])
      payload # Returns user data if valid (email, name, etc.)
    rescue GoogleIDToken::ValidationError => e
      nil
    end
  end
end
```

### Step 4: Authentication Controller

This controller will receive the token and manage the login.

```ruby
# app/controllers/api/v1/auth_controller.rb
class Api::V1::AuthController < ApplicationController
  def google
    payload = GoogleAuthService.validate(params[:id_token])

    if payload
      user = User.find_or_create_by(email: payload['email']) do |u|
        u.name = payload['name']
        u.password = SecureRandom.hex(16) # Random password for Google users
      end

      # Generate our session JWT
      token = JWT.encode({ user_id: user.id, exp: 24.hours.from_now.to_i }, Rails.application.credentials.jwt_secret)

      render json: { token: token, user: user }
    else
      render json: { error: 'Invalid Google token' }, status: :unauthorized
    end
  end
end
```

---

## Part 2: The Frontend with Next.js

We'll use the official `@react-oauth/google` library to simplify integration with the Google SDK.

### Step 1: Installation

```bash
npm install @react-oauth/google
```

### Step 2: Configure Provider

Wrap your application (usually in `_app.js` or `layout.tsx`) with the `GoogleOAuthProvider`.

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

### Step 3: Login Component

Now we implement the button that will send the token to our backend.

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
      alert('Login error');
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

### Security Considerations

1.  **Server Validation:** Never trust only the frontend. The backend **must** verify the `id_token` with Google to ensure it hasn't been tampered with.
2.  **HTTPS:** In production, both your API and your frontend must use HTTPS to protect tokens in transit.
3.  **CORS:** Ensure your Rails API allows requests from your frontend's domain (configured in `config/initializers/cors.rb`).

### Conclusion

Combining the power of **Rails** for business logic and the agility of **Next.js** for the interface allows us to build robust and modern authentication systems. By delegating Google token verification to our backend, we guarantee an enterprise-level of security without sacrificing ease of use for our users.facilidad de uso para nuestros usuarios.
