---
title: "Push Notification Server: Part 01"
date: 2017-07-19T15:05:08-04:00
draft: true
---

My first portfolio project is a real-time push notification server, written in Golang. It's modeled on the Node.js project by Beejeebus. And as luck would have it, I'm married to Beejeebus! (Yay!) So while this is probably more of an advanced project, I have a live-in mentor and occassional responsive rubber duck kindly supporting me on this one.

<b>Basic Concept</b>

To get started, I had to understand what exactly Pushie does and how it works. Here's a quick sketch that helps lay it out:

![This is an image](/img/pushie_overview.jpg)

Pushie sits between the client (let's call it a web browser) and the server (let's call it a web app). The goal is to enable real-time notifications to be received by the client from the web app. Real-time here means non-persistent, so messages sent to a particular client aren't saved for later - if the client isn't around at the moment, the messages get dropped.

Notice how Pushie is (affectionately) labeled as "DUMB" in the diagram? That's because Pushie doesn't know much about anything, and that's how we want it, because the Web App is smart. The Web App knows who (i.e. which clients) are known and allowed to access Pushie messages. It also knows *which* Pushie messages a client will see based on which channels they have access to and are currently 'logged in' to. All Pushie has to do is ask questions.

When a client reaches out to Pushie to ask to see messages (via some javascript in the browser, probably), Pushie looks at them and says, "Hey, Web App, is this person allowed to connect?" The Web App tells Pushie yes or no based on earlier actions between the Client and the Web App (imagine signing up for Slack, you identify yourself for the Web App so it can allow you to connect). If Web App says no, the connection to the client is closed. If Web App says *yes*, it also tells Pushie which channels this user is allowed to see. Pushie now knows this user is currently logged in on their available channels. 

Now Pushie can perform its core feature as a *push notification* server. When messages come in from Web App for channels the client is in, Pushie automatically pushes the message down to the client. Whee!

<b>Server Routing</b>

Pushie is a server. Writing a basic server in Go is really easy, so I was able to quickly move on to figuring out how that server should behave in more detail. To do that, I'm using the <a href="https://github.com/gorilla/mux" target="_blank">Gorilla Mux</a> URL router library. 

Pushie has to be capable of all sorts of actions, not just sending messages. It needs to:

<ul>
<li> add and remove users from channels</li>
<li> log users out, or kick them out</li>
<li> add and remove whole channels, and check their health</li>
<li> handle authentication tokens</li>
<li> right, also publish messages</li>
</ul>

You can do that by establishing a unique url route for each action you need to take, and telling Pushie to run some code when reading Web App messages from that URL.

Here's my router function using Gorilla Mux:

<pre><code>func main() {

  r := mux.NewRouter()

  r.HandleFunc("/", HomeHandler)
  r.HandleFunc("/publish", PublishHandler)
  r.HandleFunc("/user/kick/:uid", KickHandler)
  r.HandleFunc("/user/logout/:authtoken", LogoutHandler)
  r.HandleFunc("/user/channel/add/:channel/:uid", AddUserChannelHandler)
  r.HandleFunc("/user/channel/remove/:channel/:uid", RemoveUserChannelHandler)
  r.HandleFunc("channel/add/:channel", AddChannelHandler)
  r.HandleFunc("health/check", HealthCheckHandler)
  r.HandleFunc("channel/check/:channel", CheckChannelHandler)
  r.HandleFunc("channel/remove/:channel", RemoveChannelHandler)
  r.HandleFunc("user/presence-list/:uid/:uidList", SetUserPresenceHandler)
  r.HandleFunc("debug/toggle", DebugToggleHandler)
  r.HandleFunc("content/token/users", GetContentTokenHandler)
  r.HandleFunc("content/token", SetContentTokenHandler)
  r.HandleFunc("content/token/message", PublishMessageToContentChannelHandler)
  r.HandleFunc("authtoken/channel/add/:channel/:authToken", AddAuthTokenChannelHandler)
  r.HandleFunc("authtoken/channel/remove/:channel/:authToken", RemoveAuthTokenChannelHandler)

  http.Handle("/", r)

  http.ListenAndServe(":8080", nil)
}</code></pre>

See arguments one and two in the various calls to HandleFunc? Argument one is the extension of the incoming web request (eg "/publish") and argument two is the correlated function Pushie is told to run for this request (eg "PublishHandler").

<b>Reading Messages from Web App</b>

As you can see in the router code, Pushie has a lot of functions. I chose to get started with PublishHandler, Pushie's core function: publishing those incoming messages from Web App. 

So how is Pushie going to do this? JSON decoding! 

The body of the incoming http request is JSON, and we'll need to decode that into something Pushie can use: a Go object. 

Check out this <a href="https://blog.golang.org/json-and-go" target="_blank">Go Blog</a> for a good overview of Go encoding & decoding for JSON. 

A quick note for n00bs like myself: I'm building Pushie from scratch, and often the function I'm working on is meant to engage with other functions or incoming data that I haven't written, built, or set up yet. For instance, IRL Pushie will receive messages from Web App, but I don't have a Web App sending messages yet! To get around this when you're working on an incomplete app, you can hard code or 'fake' certain thingsuntil you're further along. These serve as helpful placeholders for the real inputs you'll have later. In the code snippet below, you'll see for example I *create* a Go object called Message, then *encode* it to masquerade as an incoming http request, *then* I write a decoding function to respond to it. That's the function that'll eventually decode real incoming POST requests. It sounds pretty basic, but I didn't have a sense of this part of the process before I started writing my own code. I'll probably grow out of finding it so charming, but right now, I'm going to spend my newbie points talking about what a great part of the programming process this is.

So, here's my decoding function:

<pre><code>package main

import (
  "encoding/json"
  "fmt"
)

// Create a mockup Message object.

type Message struct {
   Channel string
   Data    map[string]interface{}
}

// Create mockup message.

var decoded_message = Message{
  Channel: "bunno",
  Data: map[string]interface{} {
    "message": map[string]string {"text": "oh hai", "username": "jufisch"},
  },
}

// Here's where I encode the mockup message to get something I can work with for now that mimics a JSon POST request.

var encoded_message, err = json.Marshal(decoded_message)

// This is the function that decodes incoming messages.

func decode_message(encodedmessage []byte) Message {
  var decodedmessage Message
  json.Unmarshal(encodedmessage, &decodedmessage)

  return decodedmessage
}</code></pre>

<b>Hash Lookups & Sending Messages</b>

I spent the last part of <i>Pushie Day One: Publish Message</i> fleshing out Pushie's channel lookup function. After reading the message to find out what channel it's for, Pushie looks up that channel and finds out which clients are in there. 

As part of the Authentication and Authorization processes (yet another future post), Pushie will keep track of which users are currently logged in on each channel. It stores that data in a hash, or in Go speak, a map. They're very exciting data structures. You can try them out in the <a href="https://tour.golang.org/moretypes/19" target="_blank"> Tour of Go</a> or read more at the <a href="https://blog.golang.org/go-maps-in-action" target="_blank">Go Blog</a>. 

Events happen in the Web App. Maybe a user sent a chat message or a timer went off. When the event happens, a message gets sent to Pushie. As you know, the data in the message includes which channel it's for.

Remember the decoding of the incoming message JSON data? Once Pushie reads the message and gets the target channel for that message, it looks in its hash of channels until it finds channel "bunno." For each *channel*, Pushie knows which clients are currently there. Quick n00b diversion: you'll hear maps referred to as key-value stores. In this case, "bunno" (ie channel name) is our key. We find the key and can then see its associated *values,* in this case clients. 

Like the fake web request in the encoding discussion, here I'm faking the hash of channels by hard-coding it. When Pushie is further along, that will be a dynamic hash reflecting which users are currently logged in on which channels. 

<pre><code>package main

import (
  "encoding/json"
)

// Here I create a Websocket type -- these are the 'values' being stored in the hash, referred to as clients in the blog post.

type Websocket struct {
  id int
}

// This creates some dummy socket objects -- IRL these will be generated based on who is currently listening on Pushie channels.

var Socket1 = Websocket{
  id: 1,
}

var Socket2 = Websocket{
  id: 2,
}

var Socket3 = Websocket{
  id: 3,
}

// Here I create a send method for Websocket obejcts. 

func (s Websocket) Send(message []byte) error {
  return nil
}

// Same as before, our function to decode encoded messages (ie JSON bytes in POST requests).

func decode_message(encodedmessage []byte) Message {
  var decodedmessage Message
  json.Unmarshal(encodedmessage, &decodedmessage)

  return decodedmessage
}

// Finally, here's our lookup function. It takes the current hash or map as its first argument, and the incoming message as its second argument. Next it reads the message object to get the correct channel, and looks that channel up in the hash.

func channel_lookup(currentmap map[string][]Websocket, message Message) []Websocket{

  target_sockets, ok := currentmap[message.Channel]
  if !ok {
    return []Websocket{}
  }

  return target_sockets
}

// This is fairly messy right now, so let's comment inside func main.
 
func main() {

  // I create a message using some JSON I cooked earlier.

  message_json := []byte (`{"Channel":"bunno","Data":{"message":{"text":"oh hai","username":"jufisch"}}}`)

  // I hardcode a hash of channels and sockets. Eventually this will be dynamically stored by Pushie as it keeps track of which clients are logging in.

  channels_hash := make(map[string][]Websocket)
  channels_hash["bunno"] = []Websocket{Socket1, Socket2}
  channels_hash["quadrupal"] = []Websocket{Socket1, Socket2, Socket3}
  channels_hash["thisisfine"] = []Websocket{}

  // Here I call my decode message function. 
  message := decode_message(message_json)

  // Here's the lookedup. I call my channel_lookup function, passing it the hardcoded hash and the decoded message.

  target_sockets := channel_lookup(channels_hash, message)

  // Now that we know our 'target sockets', we iterate over them and call the send method on each one, to send the message to each client with an open socket for the channel!

  for _, socket := range target_sockets {

    socket.Send(encoded_message)

  }

}</code></pre>

And, scene.

