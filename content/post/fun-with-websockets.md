---
title: "Fun With Websockets"
date: 2017-08-09T17:30:08-04:00
---

Another Pushie update! I'm now in Week 4 of learning Go, and attempting to implement a Real-time Push Notification Server. Previously, I gave a quick overview of the project and discussed writing tests for the project with Go's testing tools. 

Today, I'm excited to report I've entered the wild world of websockets - real ones, not Websocket objects which I shamelessly faked in order to focus on the logic of my handler functions. 

If you're new to websockets, you might enjoy perusing <a href="https://tools.ietf.org/html/rfc6455" target="_blank">The WebSocket Protocol</a> over at the IETF site. This helped me get a handle on the basic concept: websockets are initiated with a client / server handshake, and can then be used to send messages back and forth (websockets allow bidirectionality) between client and server. Great.

In order to start working with websockets, you'll need to decide which websocket package to use. While Go has its own <a href="https://godoc.org/golang.org/x/net/websocket" target="_blank">'package websocket'</a> at golang.org/x/net/websocket, the documentation for it actually recommends using <a href="https://godoc.org/github.com/gorilla/websocket" target="_blank">Gorilla's more actively maintained package</a>.

After a few days of futzing with code and the concepts behind websocket communication, I've arrived at a kind of exciting situation. Cue diagram! 

![This is a new Pushie diagram](/img/Pushie02.png)

Here's a lovely new rendition of my original Pushie diagram to help me illustrate what I have now. That is:

<b>Pushie's HandlerFuncs are Working!</b>

Over in Pushie (top left), I now have a WebsocketHandler that looks like this. 

<pre><code>func WebsocketHandler(w http.ResponseWriter, r *http.Request) {

	// Get a connection to a websocket by calling Upgrade on the web request r
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		fmt.Println("ERROR ->", err)
		return
	}

	// Establish a Websocket object (defined above) to work with.

	socket := Websocket{
		connection: conn,
	}

	// Create loop to check for control messages.
	for {
		msgType, msg, err := conn.ReadMessage()

		if err != nil {
			conn.Close()
			return
		}

		if msgType == websocket.PingMessage {
			fmt.Println(msg)
			continue
		}
		// Message type is invalid, we don't expect clients to send us anything.
		conn.Close()
		return
	}
}</code></pre>

Cool! And how does WebsocketHandler get called? I'm glad you asked! 

<b>Javascript Client</b>

I quickly did up an Index.html page with Javascript that mimics the request behavior I need. (You can read up on how to work with Websockets in Javascript <a href="https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_client_applications" target="_blank">here</a> - it's incredibly easy!)

My script lives inside a myWebsocketStart function, which is called with a button onclick action. Here's the script tag contents:

<pre><code>function myWebsocketStart() {
      // Using ws: instead of http: implicitly requests an upgrade to websocket
      var ws = new WebSocket("ws://localhost:8080/websocket");

      ws.onopen = function (evt) {

        var myTextArea = document.getElementById("textarea1");
        myTextArea.value = myTextArea.value + "You're connected to Pushie!"
      };

      ws.onmessage = function (evt) {
        var myTextArea = document.getElementById("textarea1");
        myTextArea.value = myTextArea.value + "\n" + evt.data
      };

      ws.onclose = function (evt) {
        var myTextArea = document.getElementById("textarea1");
        myTextArea.value = myTextArea.value + "\n" + "Pushie connection closed."
      };

    }
  </code></pre>

When I hit "Subscribe," the myWebsocketScript sends out a websocket request (denoted with "ws://..." instead of "http(s)://...") to the URL path for WebsocketHandler. As you can see in the ws.onopen function, right now I have it print "You're connected to Pushie!" in a text area on the page to signal that the websocket was opened successfully.

<b>The Joy of Curling</b>

And last but not least, it's a throwback to remembering the way Pushie is meant to function. The client asks for a websocket connection to Pushie, but Pushie does nothing other than open that connection *until* the <i>Web App</i> provides Pushie with something to do. (Close readers may notice that I've skipped the Authentication step that I'll want to do to allow sockets to open. I'll be tackling that out of order, a little bit later on.)

I could certainly have set up a Ruby on Rails app or something like that (I hope to get Pushie working with RoR and Drupal down the road), in order to send messages to Pushie that Pushie then pushes down the socket to the client. But instead of side tracking myself for a day creating something like that, I can turn to my best friend, The Terminal.

With the cURL command, I can craft a single command that sends out a web request to my URL of choice. That URL is my path for PublishMessage, .../publish.I want to send JSON data to PublishMessage just as a real web app would, and cURL's data flag (-d) allows you to handcraft your own data. You also need to express the Content-Type header, which, when you want JSON, is going to be application/json. All in all, the cURL command looks like this:

<pre><code>curl -d '{"Channel": "bunno", "Data": {"message": {"text":"oh hai", "username": "jufisch"}}}' -H "Content-Type: application/json" -X POST http://localhost:8080/publish</code></pre>

Sweet.

<b>Houston We Have Lift-off</b>

Once all three things are in place, I walk through the following process:

(1) I run ./pushie, the compiled Pushie binary, in Terminal Window 1. That runs the ListenAndServe server in Func Main, waiting for action. 

(2) I open the Index.html file that contains my websockets javascript in my browser. To initiate a websocket connection, I hit the "Subscribe" button and send out the ws:// request to localhost:8080/websocket. This calls my WebsocketHandler function. "You're connected to Pushie!" appears in my browser page, powered by javascript. There's some debugging comments that print out in Terminal Window 1 as well, that I embedded in WebsocketHandler to follow along as it's called.

(3) In Terminal Window 2, I load up my cURL command and run it. I see more debugging info in Terminal Window 1, which is now coming from PublishHandler, which was called by the cURL request. And then, the real moment of excitement, I get the data that I place in PublishHandler's response writers printed out in Terminal Window 2, to let me know that my request was succesful and yieled a response.

Boom! That is surprisingly thrilling. And that's also all for now.
