# Websocket-Rails

[![Build Status](https://secure.travis-ci.org/DanKnox/websocket-rails.png)](https://secure.travis-ci.org/DanKnox/websocket-rails)

If you haven't done so yet, check out the [Project Page](http://danknox.github.com/websocket-rails/) to get a feel for the project direction. Feedback is very much appreciated. Post an issue on the issue tracker or [shoot us an email](mailto://support@threedotloft.com) to give us your thoughts.

## Overview

Plug and play WebSocket support for ruby on rails with streaming HTTP fallback for improved cross browser compatibility. Includes event router for mapping javascript events to controller actions. There is no need for a separate WebSocket server process. Requests to `/websocket` will be passed through to the `ConnectionManager` class which is a simple Rack based WebSocket server developed using the `Faye::WebSocket` library.

*Important Note*

This is mostly a proof of concept as of right now. Please try it out and let me know what you like or dislike. We will be adding much more soon including a development road map.

*Update*

We are finally very close to the first production release. Any comments or suggestions would be appreciated. The test coverage is now solid and the `ConnectionManager` class has been completely rewritten. There were a few bugs in the connection management on the first release that have been eliminated. Please upgrade to the latest version if you have not yet done so.

## Installation

Check out the [Example Application](https://github.com/DanKnox/websocket-rails-Example-Project) for additional information.

1. Add the gem to your Gemfile
3. Create a WebsocketRails controller - [See Documentation](http://rdoc.info/github/DanKnox/websocket-rails/master/frames/WebsocketRails/BaseController)
4. Create an `events.rb` initializer file to map events to your controller - [See Documentation](http://rdoc.info/github/DanKnox/websocket-rails/master/frames/WebsocketRails/EventMap)
5. Launch the web server and connect a WebSocket client to `ws://yourserver:port/websocket`

*Important Note About Web Servers*

Thin is the only web server currently supported. Use the `thin-socketrails` executable to override the Thin connection timeout setting. The full command to start the server in development is `thin-socketrails -p 3000 start`. Be sure to enable config.threadsafe! in your rails application and use the Rack::Fiberpool middleware to take advantage of Thin's asynchronous request processing.

````ruby
gem 'websocket-rails'
````

## Event Router

Map WebSocket events to controller actions by creating an `events.rb` file in your app/config/initializers directory

There are two built in events that are fired automatically by the dispatcher. The built in events are `:client_connected` and `:client_disconnected`. They are triggered when a new WebSocket client connects or disconnects to the server. You can handle them however you like by subscribing to the appropriate event in your `events.rb` file.

You can subscribe multiple controllers and actions to the same event to provide very clean event handling logic. The new message will be available in each controller using the `message` method discussed in the *Controllers* section below. The example event router below demonstrates subscribing to the `:new_message` event with one controller action to rebroadcast the message out to all connected clients and another controller action to log the message to a database.

The Event Map now supports namespacing events. Events triggered on the
client as `namespace.event_name` will now be dispatched to the action
subscribed to the `event_name` event under the `namespace` namespace.
See the [EventMap
Documentation](http://rdoc.info/github/DanKnox/websocket-rails/master/frames/WebsocketRails/EventMap) for more details.

````ruby
# app/config/initializers

WebsocketRails::EventMap.describe do
  # The :client_connected method is fired automatically when a new client connects
  subscribe :client_connected, to: ChatController, with_method: :client_connected
	
  # You can subscribe any number of controller actions to a single event
  subscribe :new_message, to: ChatController, with_method: :new_message
  subscribe :new_message, to: ChatLogController, with_method: :log_message
	
  subscribe :new_user, to: ChatController, with_method: :new_user
  subscribe :change_username, to: ChatController, with_method: :change_username

  namespace :product do
    subscribe :new, to: ProductController, with_method: :new_product
  end

  # The :client_disconnected method is fired automatically when a client disconnects
  subscribe :client_disconnected, to: ChatController, with_method: :delete_user
end
````

The `subscribe` method takes the event name as the first argument, then a hash where `:to` is the Controller class and `:with_method` is the action to execute.

## WebsocketRail JavaScript Client

There is an accompanying JavaScript client in the [lib/assets/javascripts/websocket_rails](https://github.com/DanKnox/websocket-rails/tree/master/lib/assets/javascripts/websocket_rails) directory. The client detects support for WebSockets in the browser and falls back to HTTP streaming if it is unavailable. The client currently works in every major browser except for internet explorer. If you are using the current master branch of this repository, you can require the client in your application.js manifest directly: `//= require websocket_rails/main`. The client will be released to rubygems soon. 

````javascript
// Setting up the client and connecting to the server:
// Do not pass the prefix of the URL, 'ws://' will be
// added automatically when the client uses WebSockets
var dispatcher = new WebSocketRails("localhost:port/websocket");

dispatcher.on_open = function() {
	// trigger a server event immediately after opening connection
	dispatcher.trigger('new_user',{user_name: 'guest'});
}

//Triggering a new event on the server
dispatcher.trigger('event_name',object_to_be_serialized_to_json);

//Listening for new events from the server
dispatcher.bind('event_name', function(data) {
	console.log(data.user_name);
})
````

Check out the [example application](https://github.com/DanKnox/websocket-rails-Example-Project) for a working implementation.  

## Controllers

There are a few differences between WebSocket controllers and standard Rails controllers. The biggest of which, is that each event will be handled by the same, continually running instance of your controller class. This means that if you set any instance variables in your methods, they will still be available when the next event is processed. On top of that, every single client that is connected will share these same instance variables. This can be an advantage if used properly, but it can also lead to bugs if not expected. We provide our own `DataStore` object accessible in a WebsocketRails controller to make it easier to store data isolated from each connected client. This is explained further below.

Do not override the `initialize` method in your class to set up. Instead, define an `initialize_session` method and perform your set up there. The `initialize_session` method will be called the first time a controller is subscribed to an event in the event router. Instance variables defined in the `initialize_session` method will be available throughout the course of the server lifetime.

````
class ChatController < WebsocketRails::BaseController
  def initialize_session
    # perform application setup here
    @message_count = 0
  end
end
````

The WebsocketRails::BaseController class provides methods for working with the WebSocket connection. Make sure you extend this class for controllers that you are using. The two most important methods are `send_message` and `broadcast_message`. The `send_message` method sends a message to the client that initiated this event, the `broadcast_message` method broadcasts messages to all connected clients. Both methods take two arguments, the event name to trigger on the client, and the message that accompanies it.

````ruby
new_message = {:message => 'this is a message'}
broadcast_message :event_name, new_message
send_message :event_name, new_message
````

Here is an example controller for handling the `:new_message` event for a basic chat application.

````ruby
class ChatController < WebsocketRails::BaseController
  def new_message
    # Print the new message and client id to the console
    puts "Message from client #{client_id} received: #{message.inspect}"
    
    # Broadcast the new message to all connected clients
    broadcast_message :new_message, message  
  end
end
````

We are using several of the methods provided by `WebsocketRails::BaseController` here, two of which are in the `puts` statement. 

The first method used is the `client_id` method. This method contains the ID of the current WebSocket client that initiated this event. Each connected client is randomly assigned an ID upon connecting to the server. You can keep track of who is who by storing the `client_id` associated with each user somewhere. You can also use the provided `DataStore` (explained later) to make keeping track of users easier.

The next method used is the `message` method. This method will always return the message, if any, that was received along with the event initiated by the client. These messages are JSON decoded by the dispatcher automatically so you can serialize objects in your javascript client and send them along with events.

Lastly, the `broadcast_message` method is called, triggering the `:new_message` event on every connected client and sending the `message` received from the client along with it.

## Data Store

The `DataStore` object is a private Hash. When you access it in a controller, you will be accessing a Hash that is private to the client that initiated the event currently executing. You can use it exactly as you would a regular Hash, except you do not have to worry about it being overridden by the next client that triggers an event. This means that unlike instance variables in WebsocketRails controllers which are shared amongst all connected clients, the data store is an easy place to temporarily persist data for each user between events.

You can access the `DataStore` object by using the `data_store` controller method.

````ruby
class ChatController < WebsocketRails::BaseController
  def new_user
    # The instance variable would be overwritten when the next user joins
    @user = User.new(name: message[:user_name])  # No Good!
    
    # This will be private for each user
    data_store[:user] = User.new(name: message[:user_name])  # Good!
    broadcast_user_list
  end
end
````

If you wish to output an Array of the assigned values in the data store for every connected client, you can use the `each_<key>` method, replacing `<key>` with the hash key that you wish to collect. 
  
Given our ongoing chat server example, we could collect all of the current `User` objects like so:
d
````ruby
data_store[:user] = 'User3'
data_store.each_user
=> ['User1','User2','User3']
````

A simple method for broadcasting the current user list to all clients would look like this:

````ruby
def broadcast_user_list
  users = data_store.each_user
  broadcast_message :user_list, users
end
````

## Message Format

The message can be a string, hash, or array. The message is serialized as JSON before being sent to the client. The message arrives at the client as a two element serialized array with the `event_name` string as the first element, and the message object you passed to the `message` parameter of the `send_message` method as the second element. The example JavaScript dispatchers decode the JSON for you as well as provide a few conveniences for event subscribing and dispatching.

If you executed this code in your controller:

````ruby
new_message = {:message => 'this is a message'}
send_message :new_message, new_message
````

The message that arrives on the client would look like:

````javascript
['new_message',{message: 'this is a message'}]
````

## Development

This gem is created and maintained by Dan Knox and Kyle Whalen under the MIT License.

Brought to you by:

Three Dot Loft LLC
