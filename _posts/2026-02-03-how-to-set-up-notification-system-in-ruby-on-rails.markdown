---
layout: post
title: "How to Set Up a Notification System in Ruby on Rails"
date: 2026-02-03 12:34:00 +0200
categories: web development ruby rails
tags: [ruby on rails, notifications, sidekiq, actioncable, backend, real-time]
---

Notifications are a critical component in any modern application. Whether it's email alerts, real-time messages, or push notifications, a well-designed system keeps your users engaged and informed. In this guide, I will show you how to build a robust and scalable notification system in **Ruby on Rails**.

## Why do you need a notification system?

Before diving into the code, it's important to understand the value:

- **Engagement**: Notifications keep users informed and coming back to the application.
- **Improved Experience**: Users know exactly what is happening.
- **Scalability**: A good system can grow with your application.
- **Flexibility**: Different channels (email, SMS, in-app, push).

## Basic Architecture

A typical notification system has these components:

1. **Notification Model**: Stores the notification information.
2. **Generators**: Logic that creates notifications based on events.
3. **Delivery Channels**: Email, SMS, push, in-app.
4. **Job Queue**: For asynchronous processing (Sidekiq, Resque).

## Step 1: Create the Notification model

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

We need a migration:

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

## Step 2: Create a Notification Generator Service

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
    
    # Process in background
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

## Step 3: Create the Job for Asynchronous Processing

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
      # In-app notifications are already in the DB
      broadcast_notification(notification)
    end
    
    notification.update(status: :sent)
  rescue => e
    notification.update(status: :failed)
    Sentry.capture_exception(e)
  end
  
  private
  
  def send_sms_notification(notification)
    # Using Twilio or similar
    # SmsService.send(notification.user.phone, notification.message)
  end
  
  def send_push_notification(notification)
    # Using Firebase Cloud Messaging or APNs
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

## Step 4: Mailer for Email Notifications

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
  <%= link_to 'View in Application', 
      root_url(utm_source: 'email', utm_medium: 'notification') %>
</p>
```

## Step 5: Controller to Manage Notifications

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

## Step 6: Using the System in Your Application

Now you can create notifications anywhere:

```ruby
# In a controller, service, or callback
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    
    if @post.save
      # Notify the creator
      NotificationService.notify(
        current_user,
        'Your post was published',
        'Your new post is now visible to everyone',
        notifiable: @post
      )
      
      # Notify followers
      followers = current_user.followers
      NotificationService.notify_multiple(
        followers,
        "#{current_user.name} shared a new post",
        @post.title,
        notification_type: :push
      )
      
      redirect_to @post, notice: 'Post successfully created'
    else
      render :new
    end
  end
end
```

## Step 7: ActionCable for Real-time Notifications

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

In the frontend (JavaScript):

```javascript
// app/javascript/channels/notifications_channel.js
import consumer from "./consumer"

consumer.subscriptions.create("NotificationsChannel", {
  connected() {
    console.log("Connected to notification channel")
  },

  received(data) {
    // Update UI with the new notification
    console.log("New notification:", data)
    addNotificationToUI(data.notification)
  }
})
```

## Best Practices

### 1. **User Preferences**
```ruby
class User < ApplicationRecord
  store :notification_preferences, accessors: [
    :email_notifications_enabled,
    :push_notifications_enabled,
    :sms_notifications_enabled
  ]
end
```

### 2. **Rate Limiting**
```ruby
class NotificationService
  def self.notify(user, title, message, notification_type: :in_app)
    # Avoid spam
    if user.notifications.where(created_at: 1.hour.ago..).count > 10
      return false
    end
    
    # ...create notification
  end
end
```

### 3. **Logging and Monitoring**
```ruby
class NotificationJob < ApplicationJob
  def perform(notification_id, notification_type)
    Rails.logger.info("Sending notification #{notification_id}")
    # ... shipping code
  end
end
```

## Useful Alternatives and Libraries

- **Noticed**: Gem that simplifies this whole process.
- **Devise**: For authentication (already mentioned).
- **Sidekiq**: For background processing.
- **Firebase**: For push notifications.
- **Twilio**: For SMS.

## Conclusion

A well-implemented notification system is fundamental to retaining users and providing an exceptional experience. Rails provides all the necessary tools to build a robust, scalable, and maintainable system.sistema robusto, escalable y mantenible.
