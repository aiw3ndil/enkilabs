---
layout: post
title: "Cómo Montar un Sistema de Notificaciones en Ruby on Rails"
date: 2026-02-03 12:34:00 +0200
categories: desarrollo web ruby rails
tags: [ruby on rails, notificaciones, sidekiq, actioncable, backend, real-time]
---

Las notificaciones son un componente crítico en cualquier aplicación moderna. Ya sea alertas de correo, mensajes en tiempo real o notificaciones push, un sistema bien diseñado mantiene a tus usuarios enganchados e informados. En esta guía, te mostraré cómo construir un sistema de notificaciones robusto y escalable en **Ruby on Rails**.

## ¿Por qué necesitas un sistema de notificaciones?

Antes de sumergirnos en el código, es importante entender el valor:

- **Engagement**: Las notificaciones mantienen a los usuarios informados y volviendo a la aplicación
- **Experiencia mejorada**: Los usuarios saben exactamente qué está pasando
- **Escalabilidad**: Un buen sistema puede crecer con tu aplicación
- **Flexibilidad**: Diferentes canales (email, SMS, in-app, push)

## Arquitectura básica

Un sistema de notificaciones típico tiene estos componentes:

1. **Modelo de Notificación**: Almacena la información de la notificación
2. **Generadores**: Lógica que crea notificaciones basada en eventos
3. **Canales de Entrega**: Email, SMS, push, in-app
4. **Cola de Trabajo**: Para procesamiento asincrónico (Sidekiq, Resque)

## Paso 1: Crear el modelo de Notificación

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :notifications, only: [:index, :destroy, :mark_as_read]
end

# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :user
  belongs_to :notifiable, polymorphic: true
  
  enum status: { pending: 0, sent: 1, failed: 2 }
  enum notification_type: { email: 0, sms: 1, in_app: 2, push: 3 }
  
  validates :user_id, :title, :message, :notification_type, presence: true
end
```

Necesitamos una migración:

```ruby
# db/migrate/20260203000000_create_notifications.rb
class CreateNotifications < ActiveRecord::Migration[7.0]
  def change
    create_table :notifications do |t|
      t.references :user, null: false, foreign_key: true
      t.references :notifiable, polymorphic: true, null: true
      t.string :title, null: false
      t.text :message, null: false
      t.integer :notification_type, default: 2
      t.integer :status, default: 0
      t.datetime :read_at
      
      t.timestamps
    end
    
    add_index :notifications, [:user_id, :created_at]
    add_index :notifications, [:user_id, :status]
  end
end
```

## Paso 2: Crear un servicio generador de notificaciones

```ruby
# app/services/notification_service.rb
class NotificationService
  def self.notify(user, title, message, notification_type: :in_app, notifiable: nil)
    notification = user.notifications.create!(
      title: title,
      message: message,
      notification_type: notification_type,
      notifiable: notifiable
    )
    
    # Procesar en background
    NotificationJob.perform_later(notification.id, notification_type)
    
    notification
  end
  
  def self.notify_multiple(users, title, message, notification_type: :in_app)
    users.each do |user|
      notify(user, title, message, notification_type: notification_type)
    end
  end
end
```

## Paso 3: Crear el Job para procesamiento asincrónico

```ruby
# app/jobs/notification_job.rb
class NotificationJob < ApplicationJob
  queue_as :default
  
  def perform(notification_id, notification_type)
    notification = Notification.find(notification_id)
    
    case notification_type
    when 'email'
      NotificationMailer.send_notification(notification).deliver_later
    when 'sms'
      send_sms_notification(notification)
    when 'push'
      send_push_notification(notification)
    when 'in_app'
      # Las notificaciones in-app ya están en la BD
      broadcast_notification(notification)
    end
    
    notification.update(status: :sent)
  rescue => e
    notification.update(status: :failed)
    Sentry.capture_exception(e)
  end
  
  private
  
  def send_sms_notification(notification)
    # Usando Twilio o similar
    # SmsService.send(notification.user.phone, notification.message)
  end
  
  def send_push_notification(notification)
    # Usando Firebase Cloud Messaging o APNs
    # PushService.send(notification.user, notification)
  end
  
  def broadcast_notification(notification)
    ActionCable.server.broadcast(
      "user_#{notification.user_id}_notifications",
      notification: render_notification(notification)
    )
  end
  
  def render_notification(notification)
    NotificationsHelper.notification_data(notification)
  end
end
```

## Paso 4: Mailer para notificaciones por email

```ruby
# app/mailers/notification_mailer.rb
class NotificationMailer < ApplicationMailer
  default from: 'notifications@enkilabs.com'
  
  def send_notification(notification)
    @notification = notification
    @user = notification.user
    
    mail(
      to: @user.email,
      subject: @notification.title
    )
  end
end

# app/views/notification_mailer/send_notification.html.erb
<h1><%= @notification.title %></h1>
<p><%= @notification.message %></p>
<p>
  <%= link_to 'Ver en la aplicación', 
      root_url(utm_source: 'email', utm_medium: 'notification') %>
</p>
```

## Paso 5: Controlador para gestionar notificaciones

```ruby
# app/controllers/notifications_controller.rb
class NotificationsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_notification, only: [:destroy, :mark_as_read]
  
  def index
    @notifications = current_user.notifications
                     .order(created_at: :desc)
                     .page(params[:page])
                     .per(20)
  end
  
  def mark_as_read
    @notification.update(read_at: Time.current)
    
    respond_to do |format|
      format.html { redirect_back fallback_location: notifications_path }
      format.json { render json: { status: 'ok' } }
    end
  end
  
  def destroy
    @notification.destroy
    respond_to do |format|
      format.html { redirect_back fallback_location: notifications_path }
      format.json { head :no_content }
    end
  end
  
  private
  
  def set_notification
    @notification = current_user.notifications.find(params[:id])
  end
end
```

## Paso 6: Usar el sistema en tu aplicación

Ahora puedes crear notificaciones en cualquier lugar:

```ruby
# En un controlador, servicio o callback
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    
    if @post.save
      # Notificar al creador
      NotificationService.notify(
        current_user,
        'Tu post fue publicado',
        'Tu nuevo post está ahora visible para todos',
        notifiable: @post
      )
      
      # Notificar a seguidores
      followers = current_user.followers
      NotificationService.notify_multiple(
        followers,
        "#{current_user.name} compartió un nuevo post",
        @post.title,
        notification_type: :push
      )
      
      redirect_to @post, notice: 'Post creado exitosamente'
    else
      render :new
    end
  end
end
```

## Paso 7: ActionCable para notificaciones en tiempo real

```ruby
# app/channels/notifications_channel.rb
class NotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_from "user_#{current_user.id}_notifications"
  end
  
  def unsubscribed
    stop_all_streams
  end
end

# config/routes.rb
Rails.application.routes.draw do
  mount ActionCable.server => '/cable'
end
```

En el frontend (JavaScript):

```javascript
// app/javascript/channels/notifications_channel.js
import consumer from "./consumer"

consumer.subscriptions.create("NotificationsChannel", {
  connected() {
    console.log("Conectado al canal de notificaciones")
  },

  received(data) {
    // Actualizar UI con la nueva notificación
    console.log("Nueva notificación:", data)
    addNotificationToUI(data.notification)
  }
})
```

## Mejores prácticas

### 1. **Preferencias de usuario**
```ruby
class User < ApplicationRecord
  store :notification_preferences, accessors: [
    :email_notifications_enabled,
    :push_notifications_enabled,
    :sms_notifications_enabled
  ]
end
```

### 2. **Rate limiting**
```ruby
class NotificationService
  def self.notify(user, title, message, notification_type: :in_app)
    # Evitar spam
    if user.notifications.where(created_at: 1.hour.ago..).count > 10
      return false
    end
    
    # ...crear notificación
  end
end
```

### 3. **Logging y monitoreo**
```ruby
class NotificationJob < ApplicationJob
  def perform(notification_id, notification_type)
    Rails.logger.info("Enviando notificación #{notification_id}")
    # ... código de envío
  end
end
```

## Alternativas y librerías útiles

- **Noticed**: Gem que simplifica todo este proceso
- **Devise**: Para autenticación (ya mencionado)
- **Sidekiq**: Para procesamiento en background
- **Firebase**: Para notificaciones push
- **Twilio**: Para SMS

## Conclusión

Un sistema de notificaciones bien implementado es fundamental para retener usuarios y proporcionar una experiencia excepcional. Rails te proporciona todas las herramientas necesarias para construir un sistema robusto, escalable y mantenible.
