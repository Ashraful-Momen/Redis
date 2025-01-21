If you'd like to implement real-time group chat with WebSockets directly, using an alternative to Pusher (for example, **Socket.IO** or **Laravel Websockets**), here's how to adapt the system for WebSockets. We'll use **Laravel WebSockets**, which is an open-source WebSocket server for Laravel, instead of Pusher.

### **Steps to Set Up Real-Time Group Chat Using WebSockets in Laravel**

---

## **1. Install Laravel Websockets**
First, you need to install **Laravel WebSockets**, a package that provides WebSocket support for Laravel applications without needing Pusher.

### **Install the Package**:

```bash
composer require beyondcode/laravel-websockets
```

### **Publish Configuration Files**:

```bash
php artisan websockets:install
```

This will generate a `config/websockets.php` file where you can configure WebSocket settings.

---

## **2. Configure WebSockets**

### **Configure `.env` for WebSockets:**

```env
BROADCAST_DRIVER=websockets
```

### **Configure `config/broadcasting.php`:**

In your `config/broadcasting.php`, update the `pusher` driver to use `websockets`:

```php
'connections' => [
    'websockets' => [
        'driver' => 'pusher',
        'key' => env('PUSHER_APP_KEY'),
        'secret' => env('PUSHER_APP_SECRET'),
        'app_id' => env('PUSHER_APP_ID'),
        'options' => [
            'cluster' => env('PUSHER_APP_CLUSTER'),
            'encrypted' => true,
        ],
    ],
],
```

### **Configure `config/websockets.php`:**

Here you can configure WebSocket settings, such as the port, and which apps can connect.

Example configuration for local development:

```php
'apps' => [
    [
        'id' => env('PUSHER_APP_ID'),
        'name' => env('PUSHER_APP_NAME', 'Laravel WebSockets'),
        'key' => env('PUSHER_APP_KEY'),
        'secret' => env('PUSHER_APP_SECRET'),
        'path' => env('PUSHER_APP_PATH'),
        'capacity' => null,
        'enabled' => true,
    ],
],
```

### **Install Socket.IO (Optional)**

If you're using **Socket.IO** as a frontend client, install it using npm:

```bash
npm install socket.io-client
```

---

## **3. Create the Event for Broadcasting**

### **Create Event for Chat Messages:**

You need an event to broadcast messages to clients via WebSockets.

```php
// app/Events/NewMessageEvent.php
namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class NewMessageEvent implements ShouldBroadcast
{
    use Dispatchable, SerializesModels;

    public $messageData;

    public function __construct($messageData)
    {
        $this->messageData = $messageData;
    }

    public function broadcastOn()
    {
        // Broadcast to the room's channel
        return new Channel('chat-room.' . $this->messageData['room']);
    }
}
```

This event will be used to broadcast the message to clients connected to the channel `chat-room.{roomName}`.

---

## **4. Set Up the Publisher (API Endpoint)**

The API will allow users to send messages, which will then be broadcast via WebSockets.

### **Create Controller for Sending Messages:**

```php
// app/Http/Controllers/ChatController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;
use App\Events\NewMessageEvent;

class ChatController extends Controller
{
    public function sendMessage(Request $request)
    {
        // Validate incoming message
        $request->validate([
            'user' => 'required|string',
            'message' => 'required|string',
            'room' => 'required|string',
        ]);

        // Prepare the message data
        $message = [
            'user' => $request->user,
            'message' => $request->message,
            'room' => $request->room,
            'timestamp' => now()->toDateTimeString()
        ];

        // Publish the message to the Redis channel
        Redis::publish('chat-room:' . $request->room, json_encode($message));

        // Fire the event to broadcast the message
        broadcast(new NewMessageEvent($message));

        return response()->json(['status' => 'Message Sent']);
    }
}
```

In this method:
- The message is sent via the Redis channel.
- The event `NewMessageEvent` is fired to broadcast the message via WebSockets.

### **Define the Route for Sending Messages:**

```php
// routes/api.php
Route::post('chat/send', [ChatController::class, 'sendMessage']);
```

---

## **5. Set Up WebSocket Listener on the Frontend**

Now, on the frontend, you'll listen for WebSocket messages using **Laravel Echo**.

### **Install Laravel Echo and Socket.IO Client**

Install the necessary dependencies in your frontend:

```bash
npm install --save laravel-echo socket.io-client
```

### **Set Up Echo in JavaScript:**

In your `resources/js/bootstrap.js`, configure Echo to connect to the WebSocket server:

```javascript
import Echo from 'laravel-echo';
import { io } from 'socket.io-client';

window.Pusher = require('pusher-js');
window.Echo = new Echo({
    broadcaster: 'pusher',
    client: io, // Use socket.io for connection
    key: 'your-pusher-app-key',  // From .env
    cluster: 'your-pusher-app-cluster',  // From .env
    forceTLS: true,
});
```

### **Frontend: Listening to WebSocket Events**

Now, in your chat view, you’ll use Laravel Echo to listen for new messages:

```javascript
document.addEventListener('DOMContentLoaded', function () {
    const roomName = 'room1'; // The room you're chatting in

    Echo.channel('chat-room.' + roomName)
        .listen('NewMessageEvent', (event) => {
            console.log(event.messageData); // Display message data
            // You can append the new message to your chat UI here
            let chatBox = document.getElementById('messages');
            let newMessage = document.createElement('li');
            newMessage.textContent = `${event.messageData.user}: ${event.messageData.message}`;
            chatBox.appendChild(newMessage);
        });
});
```

### **Blade View for Chat:**

```php
<!-- resources/views/chat.blade.php -->
<script src="{{ mix('js/app.js') }}"></script>

<div id="chat">
    <ul id="messages">
        <!-- Chat messages will appear here -->
    </ul>

    <input type="text" id="messageInput">
    <button id="sendMessage">Send</button>
</div>

<script>
    document.getElementById('sendMessage').addEventListener('click', function () {
        var message = document.getElementById('messageInput').value;

        axios.post('/api/chat/send', {
            user: 'user1',
            message: message,
            room: 'room1'
        }).then(response => {
            document.getElementById('messageInput').value = '';
        }).catch(error => {
            console.log(error);
        });
    });
</script>
```

---

## **6. Run WebSocket Server**

Start the WebSocket server provided by **Laravel WebSockets**. You can run it using:

```bash
php artisan websockets:serve
```

This will start the WebSocket server, which listens on the default WebSocket port (6001).

---

## **7. Testing the Chat System**

- Run your Laravel WebSocket server with `php artisan websockets:serve`.
- Open two browser windows or tabs (or use multiple devices) to simulate multiple users in the same chat room.
- When one user sends a message via the API, the other connected users will see the message in real time.

---

### **Final Notes:**

- **Redis:** Redis still acts as the message broker to publish messages in the background.
- **WebSockets:** Laravel WebSockets handles the WebSocket connections for real-time communication.
- **Echo:** Laravel Echo listens to WebSocket events on the frontend and automatically updates the chat interface.

This approach uses WebSockets (via Laravel WebSockets and Socket.IO) instead of Pusher, which provides full control and is more cost-effective for larger applications.
