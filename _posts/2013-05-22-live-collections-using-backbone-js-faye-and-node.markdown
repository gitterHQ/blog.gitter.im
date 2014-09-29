---
layout: post
title: Live Collections using Backbone.js, Faye and Node
date: '2013-05-22 12:00:00'
---

### Why LiveCollections?


[Gitter](https://gitter.im) is a realtime collaborative application. Having a real-time, push-based application, where updates are automatically reflected in the client, is essential. We were keen to leverage the power of Backbone.js, so we decided to extend Backbone with
real-time Live Collection capabilities to enable realtime push in Backbone collections.

__Our aim: create a real-time drop-in replacement for Backbone collections that plays nicely with Node.js__

We were already using Mongoose.js for Mongo persistence, so it made sense for us to leverage the Mongoose middleware concept to generate CRUD notifications.

Our initial implementation used the now-defunct now.js for realtime client notifications, but we switched to Faye as our requirements became clearer.

For this demo, we'll show you how to build your own LiveCollection implementation.

Take a look at the source on [GitHub](http://github.com/Troupe/faye-live-demo).

### TL;DR


With only a few moving parts, and minimal changes to your Backbone application, it's easy to drop in a real-time push-based live collection component.


####Technologies Used

This demo uses __[Faye](http://faye.jcoglan.com/)__, __[Backbone.js](http://backbonejs.org/)__, __[Mongoose.js](http://mongoosejs.com/)__, __[Baucis](https://github.com/wprl/baucis)__, __[Node.js](http://nodejs.org)__, __[Express.js](http://expressjs.com/)__ and __[MongoDB](http://www.mongodb.org/)__ (phew!). Baucis generates REST interfaces from Mongoose schemas, and we used it to keep the demo small. There is no requirement for it.

For a front-end, we're using [TodoMVC's](http://todomvc.com/architecture-examples/backbone/) Backbone.js example, with a few changes:

* LocalStorage has been switched for a REST backend.
* The REST backend is implemented as a Node.js http server connecting to a Mongodb backend, using the technologies mentioned above.

*We'll take a fairly standard stack, like this:...*
![The basic stack that we'll be adapting for LiveCollections](/content/images/2014/May/live-collections-before.png)

*... and extend it with additional client and server components, like this:*
![The final technology stack](/content/images/2014/May/live-collections-after.png)

#### Why Faye?


> "Faye is a publish-subscribe messaging system based on the Bayeux protocol. It provides message servers
> for Node.js and Ruby, and clients for use on the server and in all major web browsers."
> -- [http://faye.jcoglan.com/](http://faye.jcoglan.com/)

Alternatives to Faye include [Engine.io](https://github.com/LearnBoost/engine.io), [Socket.io](http://socket.io/), [straight sockets](http://caniuse.com/#search=websockets), but here are some reasons we feel Faye is right for us:

* _A well documented network protocol_: Faye is built on top of the well-established and open [Bayeux](http://svn.cometd.com/trunk/bayeux/bayeux.html) protocol. Being open means ObjectiveC, Android, Ruby and Javascript implementations exist and should be able to interoperate.
* _High-level messaging interface_: Faye provides a right-level API that allows us to focus on publishing messages to channels, while Faye deals with client handshaking, reconnections, etc
* _Fallback_: in the real-world of outdated browsers, websocket implementations are notoriously flakey. Faye falls back to comet-style implementations when web sockets aren't behaving correctly.
* _Easy_: Faye is easy to understand and use. See below.

```
// Start a server
var Faye = require('faye'),
server = new Faye.NodeAdapter({mount: '/'});
server.listen(8000);
 
// Create a client and subscribe to a channel
var client = new Faye.Client('http://localhost:8000/');
client.subscribe('/messages', function(message) { alert('Got a message: ' + message.text);
});
 
// Publish a message to a channel
client.publish('/messages', {
  text: 'Hello world'
});
```


#### Using a REST-like scheme for Pub/Sub Channels


Backbone Collections use REST URL endpoints and CRUD operations are performed using `POST`, `PUT` and `DELETE` operations. Faye uses channels, not URL endpoints, and passes messages. We need a way of mapping between these two domains. Luckily, it's quite easy to do:

For each LiveCollection model, we will use a separate channel. The name of the channel matches the relative REST base-url. For example, for the resource `http://server/api/todos/`, updates will be published on the channel `/api/todos/`.

The messages passed over the channel are simple JSON messages. Messages take the form:

	{
		method: 'POST/PUT/DELETE',
		body: { ... }
	}

The method attribute indicates create (POST), update (PUT) or delete (DELETE) and the body is the object associated with the event.

<table>
	<thead>
		<tr>
			<th>Operation</th>
			<th>REST (client to server)</th>
			<th>Faye (broadcast, server to clients)</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td></td>
			<td><code>GET /api/todos/</code></td>
			<td><code>client.subscribe(‘/api/todos/’, ...)</code></td>
		</tr>

		<tr>
			<td>Create</td>
			<td><code>POST /api/todos/</code></td>
			<td><code>{ method:“POST”, body: {...} }</code></td>
		</tr>

		<tr>
			<td>Update</td>
			<td><code>PUT /api/todos/id</code></td>
			<td><code>{ method:“PUT”, body: {...} }</code></td>
		</tr>

		<tr>
			<td>Delete</td>
			<td><code>DELETE /api/todos/id</code></td>
			<td><code>{ method:“DELETE”, body: {...} }</code></td>
		</tr>

	</tbody>
</table>

#### Changes to the Client


On the client, we'll extend the `Backbone.Collection` class to listen to events from Faye. The class subscribes to its Faye channel in its constructor, and messages are handled in the `_fayeEvent()` method.

```
  var fayeClient = new Faye.Client('/faye');
 
  var LiveCollection = Backbone.Collection.extend({
    constructor: function(models, options) {
      Backbone.Collection.prototype.constructor.call(this, models, options);
 
      this.subscription = fayeClient.subscribe(this.url, this._fayeEvent, this);
    },
 
    _fayeEvent: function(message) {
      var method = message.method;
      var body = message.body;
 
      switch(method) {
        case 'POST':
          this._createEvent(body);
          break;
 
        case 'PUT':
          this._updateEvent(body);
          break;
 
        case 'DELETE':
          this._removeEvent(body);
          break;
 
        default:
          ...
      }
    },
 
    ...
  });
```

Then we change the base class of the Todo collection to extend the LiveCollection, like this:

```
var TodoList = app.LiveCollection.extend({
   ...
});
```

#### Changes to the Server to Implement Push


Firstly, we'll need to attach a Faye server instance to our httpServer instance, like so:

```
var fayeServer = new faye.NodeAdapter({mount: '/faye', timeout: 45});
fayeServer.attach(httpServer);
```

The next step is to publish messages to the appropriate channel whenever the mongo collection is modified. Since we're already using Mongoose, the easiest way to generate notifications is by attaching a mongoose middleware component to the `Todo` schema.

> Middleware are functions which are passed control of flow during execution of init, validate, save and remove methods. There are two types of middleware, pre and post.

Unfortunately, in the `post('save')` middleware you can't easily tell if the save operation was for a create or an update. Therefore we use a small utility, `attachListenersToSchema()`, to help us out. Dig into the source if you're interested in how we do this.

```
// This method will send realtime notifications to the faye clients
function notifyClients(method, model, url) {
  bayeux.getClient().publish(url, {
    method: method,
    body: model.toJSON()
  });
}
 
// Attach events to the mongoose schema. Since the default mongoose middleware makes it difficult to
// distinguish between a create and an update event, we use a small utility that helps us to do that.
mongooseUtils.attachListenersToSchema(todoSchema, {
  onCreate: function(model, next) {
    notifyClients('POST', model, '/api/todos');
    next();
  },
 
  onUpdate: function(model, next) {
    notifyClients('PUT', model, '/api/todos');
    next();
  },
 
  onRemove: function(model) {
    notifyClients('DELETE', model, '/api/todos');
  }
});
```


And we're done!


## Here Be Dragons


Obviously we've had to gloss over a few details; here's a few things to look out for:

1. _Views need to respond to non-UI-driven change events_: You may need to make changes if your view code responds to UI changes instead of collection/model changes. In the TodoMVC example, the remove model code needed to be modified in order for push deleted to show in the UI. We highly recommend [Marionette.js](http://marionettejs.com/)  to ease pain here.

1. _Event ordering_: Sometimes the client will receive Faye events before the RESTful operation that caused the event has completed, other times not. Don't make any assumptions about the order and timing of events. Read how we deal with some of these problems in the next section.

1. _Security_: Obviously this demo doesn't use security, but it's something you'll need to consider carefully.

1. _Partitioning_: In this demo all clients get all change events. In a real-world system, this is usually not the case. Follow RESTful principals so that different partitions have different endpoints and this problem is fairly easy to workaround.

1. _Dodgy websockets_: Websockets are awesome when they work. Faye deals with much of the pain, but not all of it. Expect weird edge-cases when dealing with bad network connections (or mobile connections), iOS, computers that have awoken from sleep and other scenarios.

1. _Testing_: creating a solid test environment when working with push technologies is hard.

### Drama with Duplicates: Dealing with unexpected timing issues


The LiveCollection's `_createEvent()`, `_updateEvent()` and `_removeEvent()` methods are fairly straightforward, but one issue that we'll discuss here is the problem of duplicates that occurs sometimes when saving a new object (create), and how we deal with this issue.

As mentioned, sometimes the order of events is not what you might expect. For example, take a look at the following sequence when a client saves a new model to the collection:

1. HTTP POST request is initiated with `{ id: null, title: 'Hello' }`, client (asynchronously) awaits a response.
1. On Faye, the client receives create event from server with `{ id: X, title: 'Hello' }`
1. HTTP response is received:  `{ id: X, title: 'Hello' }`

At step 2, the client does not know whether the event is related to the outstanding POST event, or is an event from another client. It's only once the HTTP POST response is received that the client is able to
tell that the event was in fact for the corresponding operation, by matching on the id. If the create was for the same object, the event can safely be ignored. If not, the created object should be inserted into the collection.

We use the following strategy to deal with this situation:
* If there are no outstanding POST operations, always insert the object immediately.
* Otherwise, wait until all the outstanding POST operations have completed.
* Once they have completed, check if the id of the event matches one of the newly sync'ed objects.
* If it does, ignore the event
* If not, insert the event's body into the collection - the create must have been from another client.

This is how we do that in code:

```
  var LiveCollection = Backbone.Collection.extend({
    ...
    
    _createEvent: function(body) {
      ...
      
      // Look to see if this collection has any outstanding creates...
      var idAttribute = this.model.prototype.idAttribute;
      var unsaved = this.filter(function(model) {
        return !model.id;
      });
 
      // If there are unsaved items, monitor them and if one of them turns out to be the matching object
      // then simply update that
      if(unsaved.length) {
        console.log('live: awaiting syncs of unsaved objects');
 
        var listener = new UpdateEventListener(id, unsaved);
 
        listener.once('notfound', function() {
          this.add(body, { parse: true });
        }, this);
 
      } else {
        this.add(body, { parse: true });
 
      }
    }
    ...
  });
  
  
  function UpdateEventListener(outstandingId, models) {
    this._outstandingId = outstandingId;
    this._awaitCount = models.length;
 
    _.each(models, function(model) {
      this.listenToOnce(model, 'sync error', this._onModelSync);
    }, this);
  }
 
  _.extend(UpdateEventListener.prototype, Backbone.Events, {
    _onModelSync: function(model) {
      if(model.id === this._outstandingId) {
        // Great, this is the model we're waiting for!
        this.trigger('found', model);
        this.stopListening();
 
      } else if(--this._awaitCount === 0) {
        // All the models are done and we haven't found the id we received from the realtime stream
        this.stopListening();
        this.trigger('notfound');
      }
    }
  });
```

## LNUG Slide Deck



<iframe src="http://www.slideshare.net/slideshow/embed_code/21755606" width="427" height="356" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC;border-width:1px 1px 0;margin-bottom:5px"> </iframe>

We presented this as a talk at [LNUG](http://lanyrd.com/2013/lnug-may/schcxq/).


## In Summary


With a small amount of effort you can adapt your existing Backbone.js application to use Live Collections. Just watch out for gotchas. Give it a try and let us know how it goes.


## About the Authors


<img alt="Andrew Newdigate" src="http://www.gravatar.com/avatar/a8b4506ba466eecadc9f4c1b46d400d0.jpg" style="float:left; padding-right: 1em">

__Mike Bartlett__ is co-founder of Gitter. Product guy with enough technical knowledge to be useful yet dangerous. Brought video calling to the world at Skype. When he's not dabbling with technology he's either cooking, pontificating over fine wine, tasting ales or snowboarding. Normally not at the same time.

<a href="https://twitter.com/mydigitalself" class="twitter-follow-button" data-show-count="false" data-lang="en">Follow @mydigitalself</a>

<p style="clear:both"><br></p>

<img alt="Andrew Newdigate" src="http://www.gravatar.com/avatar/2644d6233d2c210258362f7f0f5138c2.png" style="float:left; padding-right: 1em">

__Andrew Newdigate__ is co-founder of Gitter, where he spends most of his days writing Node.js, Javascript apps and Objective-C. He's worked as a software engineer since 1996. Loves travel, photography and snowboarding and is a new dad.

<a href="https://twitter.com/suprememoocow" class="twitter-follow-button" data-show-count="false" data-lang="en">Follow @suprememoocow</a>
