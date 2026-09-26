---
layout: post
title:  "Building an In-Browser Interactive TTY Terminal with Sinatra and Next.js"
date:   2026-09-25 14:00:00 +0300
categories: ruby sinatra nextjs websockets docker pty fullstack
---

One of the standout features of modern cloud hosting platforms like Railway, Render, or Fly.io is the **in-browser interactive terminal**. Being able to jump directly into a running production container to inspect logs, troubleshoot an unexpected crash, or run database commands (`rake db:migrate`, `rails c`, or `python manage.py`) without leaving the browser dashboard is an incredible superpower.

When building [Enkihost](https://www.enkihost.com){:target="_blank"}, we wanted this exact experience for our developers. However, because our backend engine is deliberately built on **lightweight Sinatra** rather than full-blown Rails, we did not have Rails ActionCable or heavy server-side streaming engines out of the box.

So how do you bridge an in-browser terminal emulator (**xterm.js**) running in **Next.js** to a real Linux pseudo-terminal (**PTY**) running a containerized shell via **Sinatra** over WebSockets?

Here is the complete end-to-end guide on how we engineered our real-time interactive terminal.

---

### The Architecture at a Glance

An interactive terminal requires true, bidirectional, byte-level streaming with full ANSI escape sequence rendering. Keystrokes (including control characters like `Ctrl+C`, arrow keys, and tab completion) must flow instantly from the browser to the backend, and terminal output must stream back immediately with zero buffering.

```
┌────────────────────────────────────────────────────────┐
│             Next.js Client (Browser)                   │
│                                                        │
│  ┌────────────────────┐      ┌──────────────────────┐  │
│  │   xterm.js + Fit   │ <──> │  Native WebSocket    │  │
│  └────────────────────┘      └──────────┬───────────┘  │
└─────────────────────────────────────────┼──────────────┘
                                          │
                        WebSocket /cable  │ (JSON / ActionCable Protocol)
                                          ▼
┌────────────────────────────────────────────────────────┐
│             Sinatra Backend Server                     │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Rack Middleware: TerminalCableMiddleware        │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         ▼                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │  TerminalWebsocketHandler (faye-websocket)       │  │
│  │  - JWT Verification & App Authorization          │  │
│  │  - ActionCable Handshake Emulation               │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         ▼                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Ruby PTY (PTY.open -> [master, slave])          │  │
│  │  - Master IO <-> Background Reader Thread        │  │
│  │  - Slave IO  <-> Spawned Process:                │  │
│  │    `docker exec -it <container> bash --login`    │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

---

### Part 1: The Frontend — Next.js and xterm.js

On the client side, we use **xterm.js**, the industry-standard terminal component used by VS Code, Hyper, and countless cloud IDEs.

#### 1. Installing Dependencies

```bash
npm install xterm xterm-addon-fit
```

#### 2. Building the React Terminal Component

In Next.js (App Router), this component must run strictly on the client (`"use client"`), as it accesses the browser DOM and WebSocket APIs.

Here is the clean implementation (`components/TerminalTerminal.tsx`):

```tsx
"use client";

import { useEffect, useRef } from 'react';
import { Terminal as XTerm } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';
import 'xterm/css/xterm.css';

interface TerminalProps {
  appId: string;
}

export function TerminalTerminal({ appId }: TerminalProps) {
  const terminalRef = useRef<HTMLDivElement>(null);
  const xtermRef = useRef<XTerm | null>(null);
  const socketRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    if (!terminalRef.current) return;

    // 1. Initialize xterm.js instance with styling
    const term = new XTerm({
      cursorBlink: true,
      fontSize: 14,
      fontFamily: 'var(--font-ubuntu-mono), "Ubuntu Mono", Menlo, monospace',
      theme: {
        background: '#09090b', // Tailwind zinc-950
        foreground: '#fafafa', // Tailwind zinc-50
        cursor: '#a1a1aa'
      }
    });

    const fitAddon = new FitAddon();
    term.loadAddon(fitAddon);

    term.open(terminalRef.current);
    term.focus();
    fitAddon.fit();

    xtermRef.current = term;

    // Channel identifier for the ActionCable-compatible wire protocol
    const identifier = JSON.stringify({
      channel: "TerminalChannel",
      app_id: appId
    });

    term.writeln('\x1b[1;33mConnecting to Enkihost Terminal...\x1b[0m');

    // Retrieve authentication token
    const token = localStorage.getItem('enkihost_token');
    if (!token) {
      term.writeln('\x1b[1;31m✗ Error: Authentication token not found. Please log in.\x1b[0m');
      return;
    }

    // 2. Build WebSocket URL with JWT token
    const rawUrl = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000';
    const cleanUrl = rawUrl.replace('[::]', 'localhost');

    let wsUrl: string;
    try {
      const urlObj = new URL(cleanUrl.startsWith('http') ? cleanUrl : `http://${cleanUrl}`);
      urlObj.protocol = urlObj.protocol === 'https:' ? 'wss:' : 'ws:';
      urlObj.pathname = '/cable';
      urlObj.searchParams.set('token', token);
      wsUrl = urlObj.toString();
    } catch {
      wsUrl = `${cleanUrl.replace(/^http/, 'ws')}/cable?token=${encodeURIComponent(token)}`;
    }

    const socket = new WebSocket(wsUrl);
    socketRef.current = socket;

    // 3. Handle connection handshake
    socket.onopen = () => {
      const subscribeMsg = {
        command: "subscribe",
        identifier: identifier
      };
      socket.send(JSON.stringify(subscribeMsg));
      term.writeln('\x1b[1;32m✓ WebSocket Connected\x1b[0m');
    };

    socket.onerror = () => {
      term.writeln('\x1b[1;31m✗ Connection Error\x1b[0m');
    };

    socket.onclose = () => {
      term.writeln('\x1b[1;31m✗ Connection Closed\x1b[0m');
    };

    // 4. Handle incoming terminal stream chunks
    socket.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        if (data.type === 'ping') return;
        if (data.type === 'reject_subscription') {
          term.writeln('\x1b[1;31m✗ Subscription Rejected\x1b[0m');
          return;
        }
        if (data.type === 'disconnect') {
          term.writeln(`\x1b[1;31m✗ Disconnected: ${data.message || data.reason || 'Server disconnected'}\x1b[0m`);
          return;
        }
        if (data.message && data.message.data) {
          term.write(data.message.data);
        }
      } catch {
        // Fallback for raw text frames
      }
    };

    // 5. Send keystrokes directly to backend
    term.onData((data) => {
      if (socket.readyState === WebSocket.OPEN) {
        const msg = {
          command: "message",
          identifier: identifier,
          data: JSON.stringify({
            action: "send_input",
            input: data
          })
        };
        socket.send(JSON.stringify(msg));
      }
    });

    // 6. Handle viewport resize
    const handleResize = () => {
      fitAddon.fit();
    };
    window.addEventListener('resize', handleResize);
    setTimeout(handleResize, 1000);

    return () => {
      window.removeEventListener('resize', handleResize);
      socket.close();
      term.dispose();
    };
  }, [appId]);

  return (
    <div className="w-full h-[550px] rounded-2xl overflow-hidden border border-zinc-800 bg-zinc-950 p-4 font-mono shadow-2xl">
      <div ref={terminalRef} className="w-full h-full" />
    </div>
  );
}
```

---

### Part 2: The Backend — Sinatra, Faye-WebSocket, and Ruby PTY

Rails uses ActionCable, but in Sinatra, we can use the battle-tested gem `faye-websocket` combined with a custom Rack middleware.

Add these gems to your `Gemfile`:

```ruby
gem "sinatra"
gem "faye-websocket"
gem "jwt"
gem "puma"
```

#### 1. Intercepting WebSockets in Sinatra with Rack Middleware

In your Sinatra entry point (`app.rb`), mount a middleware that intercepts WebSocket upgrade requests directed to `/cable` before they hit regular Sinatra routing:

```ruby
# app.rb
require 'sinatra'
require 'faye/websocket'
require_relative 'services/terminal_websocket_handler'

class App < Sinatra::Base
  # Intercept ActionCable-compatible WebSocket upgrades
  class TerminalCableMiddleware
    def initialize(app)
      @app = app
    end

    def call(env)
      path = env['PATH_INFO']
      if (path == '/cable' || path == '/api/v1/cable') && Faye::WebSocket.websocket?(env)
        TerminalWebsocketHandler.call(env)
      else
        @app.call(env)
      end
    end
  end

  use TerminalCableMiddleware

  # Normal REST endpoints continue below...
end
```

#### 2. The Core Handler: `TerminalWebsocketHandler`

Now comes the secret sauce: `PTY` from Ruby's standard library. 

Normally, commands spawned in subshells don't think they are running in an interactive terminal. They buffer output, hide interactive prompts, and don't receive terminal signals. 

By using `PTY.open`, the OS allocates a virtual master/slave terminal pair. We spawn `docker exec -it ...` connected to the **slave**, while our Sinatra process reads from and writes to the **master**.

Here is the complete implementation (`services/terminal_websocket_handler.rb`):

```ruby
# frozen_string_literal: true

require 'faye/websocket'
require 'json'
require 'jwt'
require 'pty'
require 'rack/utils'

class TerminalWebsocketHandler
  def self.call(env)
    new(env).response
  end

  def initialize(env)
    @env = env
    @user = nil
    @app = nil
    @identifier = nil
    @master = nil
    @pid = nil
    @reader_thread = nil
  end

  def response
    unless Faye::WebSocket.websocket?(@env)
      return [400, { 'Content-Type' => 'application/json' }, [{ error: 'WebSocket upgrade required' }.to_json]]
    end

    ws = Faye::WebSocket.new(@env, nil, { ping: 25 })

    ws.on :open do |_event|
      handle_open(ws)
    end

    ws.on :message do |event|
      handle_message(ws, event.data)
    end

    ws.on :close do |_event|
      handle_close
    end

    ws.rack_response
  end

  private

  # 1. Authenticate user from query string or headers
  def handle_open(ws)
    @user = authenticate_user
    unless @user
      ws.send({ type: 'disconnect', reason: 'unauthorized', message: 'Authentication required' }.to_json)
      ws.close
      return
    end

    # Send ActionCable welcome message
    ws.send({ type: 'welcome' }.to_json)
  end

  # 2. Handle ActionCable commands
  def handle_message(ws, raw_data)
    data = JSON.parse(raw_data) rescue nil
    return unless data.is_a?(Hash)

    command = data['command']
    identifier = data['identifier']

    case command
    when 'subscribe'
      handle_subscribe(ws, identifier)
    when 'message'
      handle_client_message(data['data'])
    end
  rescue StandardError => e
    warn "[TerminalWS] Error: #{e.message}"
  end

  # 3. Establish PTY and spawn Docker shell
  def handle_subscribe(ws, identifier_str)
    @identifier = identifier_str
    params = JSON.parse(identifier_str) rescue {}

    channel = params['channel']
    app_id = params['app_id']

    unless channel == 'TerminalChannel' && app_id.present?
      ws.send({ identifier: identifier_str, type: 'reject_subscription' }.to_json)
      return
    end

    # Authorize app ownership
    begin
      @app = @user.apps.find(app_id)
    rescue ActiveRecord::RecordNotFound
      ws.send({ identifier: identifier_str, type: 'reject_subscription' }.to_json)
      ws.send({ identifier: identifier_str, message: { data: "\r\n\x1b[31mError: App not found or unauthorized.\x1b[0m\r\n" } }.to_json)
      ws.close
      return
    ensure
      ActiveRecord::Base.connection_handler.clear_active_connections! if defined?(ActiveRecord::Base)
    end

    container_name = @app.current_container_name
    docker_bin = find_docker_bin

    # Verify container is running
    is_running = `#{docker_bin} inspect -f '{% raw %}{{.State.Running}}{% endraw %}' #{container_name} 2>/dev/null`.strip == 'true' rescue false
    unless is_running
      ws.send({ identifier: identifier_str, message: { data: "\r\n\x1b[31mError: Container #{container_name} is not running.\x1b[0m\r\n" } }.to_json)
      ws.close
      return
    end

    # Auto-detect shell (bash or sh fallback)
    has_bash = system("#{docker_bin} exec #{container_name} which bash >/dev/null 2>&1")
    shell = has_bash ? 'bash' : 'sh'
    login_flag = (shell == 'bash' ? '--login' : '-l')

    # Allocate Pseudo-Terminal
    @master, slave = PTY.open
    command = "#{docker_bin} exec -it #{container_name} #{shell} #{login_flag}"
    @pid = spawn(command, in: slave, out: slave, err: slave, pgroup: true)
    slave.close # Close slave in parent process immediately

    # Confirm subscription to frontend
    ws.send({ identifier: identifier_str, type: 'confirm_subscription' }.to_json)
    ws.send({ identifier: identifier_str, message: { data: "\x1b[1;32m✓ Connected to container (#{container_name})! Ready for input.\x1b[0m\r\n" } }.to_json)

    # 4. Background thread to stream PTY output to the WebSocket
    @reader_thread = Thread.new do
      begin
        loop do
          rs, = IO.select([@master], nil, nil, 1.0)
          if rs
            begin
              chunk = @master.readpartial(4096)
              ws.send({ identifier: @identifier, message: { data: chunk } }.to_json)
            rescue EOFError, Errno::EIO
              ws.send({ identifier: @identifier, message: { data: "\r\n\x1b[33mTerminal session closed.\x1b[0m\r\n" } }.to_json)
              ws.close rescue nil
              break
            end
          end

          # Check if child process died
          begin
            Process.getpgid(@pid)
          rescue Errno::ESRCH
            ws.send({ identifier: @identifier, message: { data: "\r\n\x1b[33mProcess exited.\x1b[0m\r\n" } }.to_json)
            ws.close rescue nil
            break
          end
        end
      rescue StandardError => e
        warn "[TerminalWS] Reader thread exception: #{e.message}"
      ensure
        cleanup_process
      end
    end
  end

  # 5. Write incoming client keystrokes into Master PTY
  def handle_client_message(data_payload)
    parsed = data_payload.is_a?(String) ? (JSON.parse(data_payload) rescue {}) : data_payload
    return unless parsed.is_a?(Hash)

    action = parsed['action']
    if action == 'send_input'
      input = parsed['input']
      if input && @master
        begin
          @master.syswrite(input)
          @master.flush rescue nil
        rescue StandardError => e
          warn "[TerminalWS] PTY write error: #{e.message}"
        end
      end
    end
  end

  def handle_close
    @reader_thread&.kill rescue nil
    @reader_thread = nil
    cleanup_process
  end

  def cleanup_process
    if @pid
      begin
        Process.kill('TERM', @pid)
        Process.wait(@pid)
      rescue StandardError
        # Already terminated
      end
      @pid = nil
    end

    if @master
      begin
        @master.close
      rescue StandardError
      end
      @master = nil
    end
  end

  def authenticate_user
    query_params = Rack::Utils.parse_nested_query(@env['QUERY_STRING'] || '')
    token = query_params['token'].presence || @env['HTTP_AUTHORIZATION']&.split(' ')&.last
    return nil unless token.present?

    secret = ENV['JWT_SECRET_KEY']
    if token.count('.') == 2
      decoded, _ = JWT.decode(token, secret, true, { algorithm: 'HS256' })
      if decoded && decoded['sub']
        return User.find_by(id: decoded['sub'])
      end
    end
    nil
  rescue StandardError => e
    warn "[TerminalWS] Auth error: #{e.message}"
    nil
  ensure
    ActiveRecord::Base.connection_handler.clear_active_connections! if defined?(ActiveRecord::Base)
  end

  def find_docker_bin
    ENV['DOCKER_BINARY'] || '/usr/bin/docker'
  end
end
```

---

### Critical Lessons & Production Gotchas

When you run persistent WebSockets in Sinatra and stream terminal sessions, there are a few subtle production traps to watch out for:

#### 1. Preventing Zombie Shell Processes
Notice the `cleanup_process` method:
```ruby
Process.kill('TERM', @pid)
Process.wait(@pid)
```
If a developer closes their browser tab or experiences a network dropout, the WebSocket closes. If you don't explicitly terminate the spawned process (`docker exec`), orphaned shells will accumulate inside your containers and server, hogging memory and CPU.

#### 2. Always Release ActiveRecord Connections
WebSockets are persistent connections that live outside the standard request/response lifecycle. If your authentication logic touches the database (`User.find_by(id: ...)`) and you don't call:
```ruby
ActiveRecord::Base.connection_handler.clear_active_connections!
```
that DB connection will remain checked out of the connection pool for the entire duration of the terminal session! A few open terminal tabs will quickly exhaust your PostgreSQL connection pool. Always run connection clearing in an `ensure` block.

#### 3. Emulating the ActionCable Protocol
Instead of writing a custom WebSocket protocol from scratch, mimicking ActionCable's wire protocol (`subscribe`, `confirm_subscription`, `message`, `ping`) offers great benefits:
- The frontend stays clean and consistent with standard real-time patterns.
- If you ever decide to run Rails ActionCable or AnyCable in the future, the frontend requires zero changes.

#### 4. Handling `Errno::EIO`
On Linux, when the slave end of a PTY is closed by the child process (for instance, if the user types `exit`), reading from the master raises `Errno::EIO`. Many developers assume this is an unhandled I/O error and crash their server. In reality, `Errno::EIO` simply indicates an **End Of File (EOF)** on Linux PTYs. Treat it as a clean exit.

---

### Summary

With fewer than 200 lines of Ruby and a clean Next.js React component, you can build a lightning-fast, production-grade interactive terminal for your cloud infrastructure.

By pairing **Ruby's built-in `PTY` library**, **`faye-websocket`**, and **`xterm.js`**, we got complete interactive control over remote Docker containers without the overhead of heavy third-party services.

If you are building developer tools, cloud dashboards, or internal admin consoles, give this architecture a try!
