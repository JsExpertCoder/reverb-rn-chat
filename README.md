# Let´s explore how to use Laravel Reverb to establish a WebSocket connection between your Laravel backend and a React Native application built with Expo. I think I covered the necessary steps to set up Laravel Reverb, configure the WebSocket connection, subscribe to channels using Laravel Echo and chat with others!

## 1 - [aravel (Backend)](https://github.com/JsExpertCoder/lara-chat)

#### 1.1 - Create a new laravel project

[Instructions in the official documentation](https://www.laravel.com/docs/11.x/installation)

#### 1.2 - Install Broadcasting Reverb

[Instructions in the official documentation](https://www.laravel.com/docs/11.x/reverb)

#### 1.3 - Install Sanctum for API communication

[Instructions in the official documentation](https://www.laravel.com/docs/11.x/sanctum#main-content)

#### 1.4 - Additional configurations

[Based in "cors-and-cookies" in the docs](https://www.laravel.com/docs/11.x/sanctum#cors-and-cookies)

```
php artisan config:publish cors
```

##### config/cors.php

```
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | Cross-Origin Resource Sharing (CORS) Configuration
    |--------------------------------------------------------------------------
    |
    | Here you may configure your settings for cross-origin resource sharing
    | or "CORS". This determines what cross-origin operations may execute
    | in web browsers. You are free to adjust these settings as needed.
    |
    | To learn more: developer.mozilla.org/en-US/docs/Web/HTTP/CORS
    |
    */

    'paths' => ['api/*', 'sanctum/csrf-cookie'],

    'allowed_methods' => ['*'],

    'allowed_origins' => ['*'],

    'allowed_origins_patterns' => [],

    'allowed_headers' => ['*'],

    'exposed_headers' => [],

    'max_age' => 0,

    'supports_credentials' => true,

];
```

##### resources/js/bootstrap.js

```
import axios from "axios";
window.axios = axios;

window.axios.defaults.headers.common["X-Requested-With"] = "XMLHttpRequest";

// Enable credentials and XSRF token handling
window.axios.defaults.withCredentials = true;
window.axios.defaults.withXSRFToken = true;

/**
 * Echo exposes an expressive API for subscribing to channels and listening
 * for events that are broadcast by Laravel. Echo and event broadcasting
 * allow your team to quickly build robust real-time web applications.
 */

import "./echo";
```

##### bootstrap/app.php

[Based in "authorizing-private-broadcast-channels" in the docs](https://www.laravel.com/docs/11.x/sanctum#authorizing-private-broadcast-channels)

```
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__ . '/../routes/web.php',
        api: __DIR__ . '/../routes/api.php',
        commands: __DIR__ . '/../routes/console.php',
        // channels: __DIR__.'/../routes/channels.php',
        health: '/up',
    )
    ->withBroadcasting(
        __DIR__ . '/../routes/channels.php',
        ['prefix' => 'api', 'middleware' => ['api', 'auth:sanctum']],
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

#### 1.5 - Setup a Channel

##### routes/channels.php

```
<?php

use Illuminate\Support\Facades\Broadcast;

use App\Models\User;

Broadcast::channel('chat.{receiverId}', function (User $user, int $receiverId) {
    return (int) $user->id === (int) $receiverId;
});
```

#### 1.6 - "chat_messages" table migration

```
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('chat_messages', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained();
            $table->integer('from');
            $table->text('message');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('chat_messages');
    }
};

```

### 1.7 Database seeder

#### Database/seeders/DatabaseSeeder.php

```
<?php

namespace database/Seeders/DatabaseSeeder.php;

use App\Models\User;
// use Illuminate\Database\Console\Seeds\WithoutModelEvents;
use Illuminate\Support\Str;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the application's database.
     */
    public function run(): void
    {
        $roles = [
            'Software Engineer', 'DevOps', 'Database Analyst', 'Cyber Security Consultant', 'Project Manager',
            'System Architect', 'QA Engineer', 'Product Owner', 'UI/UX Designer', 'Scrum Master'
        ];
        shuffle($roles);

        foreach ($roles as $role) {
            User::create([
                'name' => fake()->name(),
                'role' => $role,
                'email' => fake()->unique()->safeEmail(),
                'email_verified_at' => now(),
                'password' => Hash::make('password'),
                'remember_token' => Str::random(10),
            ]);
        }
        // User::factory(10)->create();

        // User::factory()->create([
        //     'name' => 'Test User',
        //     'email' => 'test@example.com',
        // ]);
    }
}
```

### 1.8 - Create an Event do broadcast

```
php artisan make:event MessageSent
```

#### app/Events/MessageSent.php

```
<?php

namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class MessageSent implements ShouldBroadcastNow
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     */
    public $receiver;
    public $sender;
    public $message;

    public function __construct($receiver, $sender, $message)
    {
        $this->receiver = [
            'id' => $receiver->id,
            'name' => $receiver->name,
            'role' => $receiver->role,
        ];

        $this->sender = [
            'id' => $sender->id,
            'name' => $sender->name,
            'role' => $sender->role,
        ];

        $this->message = $message;
    }

    /**
     * Get the channels the event should broadcast on.
     */
    public function broadcastOn(): Channel
    {
        return new PrivateChannel('chat.' . $this->receiver['id']);
    }
}
```

### 1.9 - Create Controllers

```
php artisan make:Controller API/Auth/_Auth
```

```
php artisan make:Controller API/ChatMessageController
```

#### app/Http/Controllers/API/Auth/\_Auth.php

```
<?php

namespace App\Http\Controllers\API\Auth;

use App\Http\Controllers\Controller;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class _Auth extends Controller
{
    public function login(Request $request)
    {
        try {
            $request->validate([
                'email' => 'required|email',
                'password' => 'required',
            ]);

            $user = User::where('email', $request->email)->first();

            if (!$user || !Hash::check($request->password, $user->password)) {
                return response()->json([
                    'status' => 401,
                    'type' => 'error',
                    'title' => 'Feedback 👋',
                    'message' => '❌ The provided credentials are incorrect'
                ]);
            }

            return [
                'status' => 200,
                'type' => 'success',
                'title' => 'Feedback 👋',
                'message' => '✅ Everything is ok',
                'data' => [
                    'user' => $user,
                    'token' => $user->createToken('login-token')->plainTextToken

                ]
            ];
        } catch (\Throwable $th) {
            return response()->json([
                'status' => 500,
                'type' => 'error',
                'title' => 'Feedback 👋',
                'message' => '❌ Internal server error'
            ]);
        }
    }
}
```

#### app/Http/Controllers/API/ChatMessageController.php (dispatch the event on "store" function)

```
<?php

namespace App\Http\Controllers\API;

use App\Events\MessageSent;

use App\Http\Controllers\Controller;
use App\Models\ChatMessage;
use App\Models\User;

use Illuminate\Database\Eloquent\Collection;
use Illuminate\Http\Request;
use Illuminate\Http\Response;

class ChatMessageController extends Controller
{
    public function store(Request $request): Response
    {
        $validatedData = $request->validate([
            'user_id' => 'required|exists:users,id',
            'from' => 'required|exists:users,id',
            'message' => 'required|string',
        ]);

        $message = ChatMessage::create($validatedData);
        $receiver = User::find($request->user_id);
        $sender = User::find($request->from);

        // broadcast(new MessageSent($receiver, $sender, $request->message));
        MessageSent::dispatch($receiver, $sender, $message);

        return response([
            'status' => 200,
            'type' => 'success',
            'title' => 'Feedback 👋',
            'message' => '✅ message sent',
            'data' => [
                'sender' => $sender,
                'message' =>  $message,
                'receiver' => $receiver
            ]
        ]);
    }
}
```

### 1.10 - Setup API routes (login and "send-message")

#### routes/api.php

```
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

use App\Http\Controllers\API\Auth\_Auth;
use App\Http\Controllers\API\ChatMessageController;

Route::post('login', [_Auth::class, 'login']);

Route::get('user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');

Route::middleware(['auth:sanctum'])->group(function () {

    // Route::get('get-unread-messages', [ChatMessageController::class, 'getUnreadMessages']);

    // Route::post('/get-team-members', [TeamController::class, 'index']);

    Route::post('send-message', [ChatMessageController::class, 'store']);
});

```

## 2 - [React Native](https://github.com/JsExpertCoder/rnative-chat)

### 2.1 Create a new React Native app (I used Expo)

[Instructions in the official documentation](docs.expo.dev/)

### 2.2 Install the dependencies

##### check the dependencies I used (package.json)

```
{
  "name": "YOUR APP NAME",
  "version": "1.0.0",
  "main": "expo-router/entry",
  "scripts": {
    "start": "expo start",
    "android": "expo start --android",
    "ios": "expo start --ios",
    "web": "expo start --web"
  },
  "dependencies": {
    "@react-native-community/netinfo": "11.3.1",
    "axios": "^1.7.2",
    "expo": "~51.0.11",
    "expo-constants": "~16.0.2",
    "expo-linking": "~6.3.1",
    "expo-router": "~3.5.15",
    "expo-status-bar": "~1.12.1",
    "laravel-echo": "^1.16.1",
    "pusher-js": "^8.4.0-rc2",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "react-native": "0.74.2",
    "react-native-safe-area-context": "4.10.1",
    "react-native-screens": "3.31.1",
    "react-native-web": "~0.19.10",
    "socket.io-client": "^4.7.5",
    "twrnc": "^4.2.0"
  },
  "devDependencies": {
    "@babel/core": "^7.20.0",
    "react-native-dotenv": "^3.4.11"
  },
  "private": true
}
```

[check this to configure react-native-dotenv](https://www.github.com/goatandsheep/react-native-dotenv)
I just made this (babel.config.js):

```
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ["babel-preset-expo"],
    plugins: [
      ["module:react-native-dotenv"],
      // {
      // envName: "APP_ENV",
      // moduleName: "@env",
      // path: ".env",
      // blocklist: null,
      // allowlist: null,
      // blacklist: null, // DEPRECATED
      // whitelist: null, // DEPRECATED
      // safe: false,
      // allowUndefined: true,
      // verbose: false,
      // },
    ],
  };
};
```

### 2.3 Setup .env

#### .env

```
# YOUR-SERVER-HOST:YOUR-SERVER-PORT
# (YOU CAN GET IT WHEN YOU RUN php artisan server --host=YOUR-LOCAL-IP)

BACKEND_API_URL=http://YOUR-SERVER-HOST:YOUR-SERVER-PORT/api

REVERB_APP_ID= # FIND IT IN .env FILE OF YOU LARAVEL APPLICATION
REVERB_APP_KEY=  # FIND IT IN .env FILE OF YOU LARAVEL APPLICATION
REVERB_APP_SECRET= # FIND IT IN .env FILE OF YOU LARAVEL APPLICATION
REVERB_HOST= # FIND IT IN .env FILE OF YOU LARAVEL APPLICATION
REVERB_PORT= # FIND IT IN .env FILE OF YOU LARAVEL APPLICATION
REVERB_SCHEME= # FIND IT IN .env FILE OF YOU LARAVEL APPLICATION
```

### 2.4 Setup services (API)

#### services/api

```
import axios from "axios";
import { BACKEND_API_URL } from "@env";

const api = axios.create({
  baseURL: BACKEND_API_URL,
  headers: {
    "Content-Type": "application/json",
  },
});

const setBearerToken = (token) => {
  api.defaults.headers.common["Authorization"] = `Bearer ${token}`;
};

export { api, setBearerToken };

```

#### services/auth

```
import { api } from "./api";

export const Auth = {
  async login(email, password) {
    const options = {
      method: "POST",
      url: "/login",
      data: {
        email,
        password,
      },
    };

    try {
      const response = await api.request(options);
      return response.data;
    } catch (error) {
      throw error;
    }
  },
};

```

### 2.5 Create Echo instance (our websocket client)

#### hooks/echo

```
import { useEffect, useState } from "react";
// import axios from "axios";
import { api as axios } from "../services/api";
import Echo from "laravel-echo";
import Pusher from "pusher-js/react-native";
import { REVERB_APP_KEY, REVERB_HOST, REVERB_PORT, REVERB_SCHEME } from "@env";

const useEcho = () => {
  const [echoInstance, setEchoInstance] = useState(null);

  useEffect(() => {
    //  Setup Pusher client
    const PusherClient = new Pusher(REVERB_APP_KEY, {
      wsHost: REVERB_HOST,
      wsPort: REVERB_PORT ?? 80,
      wssPort: REVERB_PORT ?? 443,
      forceTLS: (REVERB_SCHEME ?? "https") === "https",
      enabledTransports: ["ws", "wss"],
      disableStats: true,
      cluster: "mt1",
      authorizer: (channel, options) => {
        return {
          authorize: (socketId, callback) => {
            axios
              .post("/broadcasting/auth", {
                socket_id: socketId,
                channel_name: channel.name,
              })
              .then((response) => {
                callback(false, response.data);
              })
              .catch((error) => {
                callback(true, error);
              });
          },
        };
      },
    });

    // Create Echo instance
    const echo = new Echo({
      broadcaster: "reverb",
      client: PusherClient,
    });

    setEchoInstance(echo);

    // Cleanup on unmount
    return () => {
      if (echo) {
        echo.disconnect();
      }
    };
  }, []);

  return echoInstance;
};

export default useEcho;

```

### 2.6 Create chat component

#### components/chatInterface

```
// src/components/ChatInterface.js
import React, { useState, useEffect } from "react";
import {
  SafeAreaView,
  View,
  TextInput,
  TouchableOpacity,
  Text,
  FlatList,
  KeyboardAvoidingView,
  Platform,
} from "react-native";
import { style as tw } from "twrnc";
import useEcho from "../hooks/echo";
import { api } from "../services/api";

const ChatInterface = ({ user }) => {
  const [messages, setMessages] = useState([]);
  const [inputText, setInputText] = useState("");
  const [userData, setUserData] = useState(user);

  const echo = useEcho();

  const handleSend = async () => {
    if (inputText.trim()) {
      setInputText("");
      const options = {
        method: "POST",
        url: "/send-message",
        data: {
          /**I logged in with user 1 and 7.
           * then you configure this to be dynamic
           * (giving the user the power to choose who they want to chat with)
           * */
          user_id: userData.id === 1 ? 7 : 1, // receiver
          from: userData.id,
          message: inputText,
        },
      };

      try {
        const response = await api.request(options);
        if (response.status === 200) {
          const data = response.data.data;
          console.log("response", data);
          setMessages((prevMessages) => [
            ...prevMessages,
            {
              id: data?.message?.id,
              from: userData.id,
              message: data?.message?.message,
            },
          ]);
        }
      } catch (error) {
        console.error(error);
        // Handle error appropriately
      }
    }
  };

  function subscribeToChatChannel() {
    if (echo) {
      echo.private(`chat.${user?.id}`).listen("MessageSent", (e) => {
        console.log("real-time-event", {
          id: e.message?.id,
          message: e.message?.message,
          from: e.message?.from,
        });
        setMessages((prevMessages) => [
          ...prevMessages,
          {
            id: e.message?.id,
            message: e.message?.message,
            from: e.message?.from,
          },
        ]);
      });
    }
  }

  useEffect(() => {
    if (echo) {
      console.log("user", userData.id);
      subscribeToChatChannel();
    }
    return () => {
      if (echo && user) {
        echo.leaveChannel(`chat.${user?.id}`);
      }
    };
  }, [echo, user]);

  const renderItem = ({ item }) => (
    <View
      style={tw(
        `flex flex-row  my-2`,
        item.from == userData.id ? `justify-end` : "justify-start"
      )}
    >
      <View
        style={tw(
          `rounded-lg p-6 max-w-3/4`,
          item.from == userData.id ? "bg-[#1EBEA5]" : "bg-[#8696a0]"
        )}
      >
        <Text style={tw`text-white`}>{item.message}</Text>
      </View>
    </View>
  );

  return (
    <SafeAreaView style={tw`flex-1 bg-[#0b141a]`}>
      <FlatList
        data={messages}
        renderItem={renderItem}
        keyExtractor={(item) => item.id}
        contentContainerStyle={tw`p-4`}
      />
      <KeyboardAvoidingView
        behavior={Platform.OS === "ios" ? "padding" : "height"}
      >
        <View style={tw`p-4 border border-gray-700 gap-2`}>
          <View style={tw`bg-blue-500 p-6 rounded-lg`}>
            <Text style={tw`text-white`}>{userData?.name}</Text>
          </View>
        </View>
        <View style={tw`flex flex-row p-4 border-t border-gray-700 gap-2`}>
          <TextInput
            style={tw`flex-1 bg-gray-800 text-white p-6 rounded-lg`}
            value={inputText}
            onChangeText={setInputText}
            placeholder="Type a message..."
            placeholderTextColor="gray"
          />
          <TouchableOpacity
            onPress={handleSend}
            style={tw`bg-blue-500 p-6 rounded-lg`}
          >
            <Text style={tw`text-white`}>Send</Text>
          </TouchableOpacity>
        </View>
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
};

export default ChatInterface;
```

### 2.7 Login users

#### index.js

```
import { StatusBar } from "expo-status-bar";
import { Text, TouchableOpacity, View } from "react-native";

import { style as tw } from "twrnc";

import ChatInterface from "../components/chatInterface";
import { useState } from "react";
import { setBearerToken } from "../services/api";
import { Auth } from "../services/auth";

export default function App() {
  const [userData, setUserData] = useState(null);

  async function handleLogin(userCredentials) {
    try {
      const response = await Auth.login(...userCredentials);
      if (response.status == 200) {
        const data = response.data;
        setBearerToken(data.token);
        setUserData(data.user);
      }
    } catch (error) {
      console.error("Error:", error);
    }
  }

  return userData ? (
    <ChatInterface user={userData} />
  ) : (
    <View style={tw(`bg-gray-300 flex-1 items-center justify-center gap-16`)}>
      <TouchableOpacity
        onPress={() =>
          handleLogin(["FIRST-USER-EMAIL-HERE", "FIRST-USER-PASSWORD-HERE"])
        }
        style={tw`w-80 bg-blue-500 p-7 rounded-lg`}
      >
        <Text style={tw`text-white`}>login 1</Text>
      </TouchableOpacity>
      <TouchableOpacity
        onPress={() =>
          handleLogin(["SECOND-USER-EMAIL-HERE", "SECOND-USER-PASSWORD-HERE"])
        }
        style={tw`w-80 bg-blue-500 p-7 rounded-lg`}
      >
        <Text style={tw`text-white`}>login 2</Text>
      </TouchableOpacity>
    </View>
  );
}
```

## 3 - Notes

**[One of the bases was this video on YouTube](https://www.youtube.com/watch?v=xEV7ruVUEvs&t=1516s)**

> Feel free to ask, agree or complain anything
