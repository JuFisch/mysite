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

Notice how Pushie is (affectionately) labeled as "DUMB in the diagram? That's because Pushie doesn't know much about anything, and that's how we want it, because the Web App is smart. The Web App knows who (i.e. which clients) are known and allowed to access Pushie messages. It also knows *which* Pushie messages a client will see based on which channels they have access to and are currently 'logged in' to. All Pushie has to do is ask questions.

When a client reaches out to Pushie to ask to see messages (via some javascript in the browser, probably), Pushie looks at them and says, "Hey, Web App, is this person allowed to connect?" The Web App tells Pushie yes or no based on earlier actions between the Client and the Web App (imagine signing up for Slack, you identify yourself for the Web App so it can allow you to connect). If Web App says no, the connection to the client is closed. If Web App says *yes*, it also tells Pushie which channels this user is allowed to see. 

After a 'yes, known user' reply from Web App, Pushie opens a websocket for the client. Now, any messages being sent via the Web App to any of the channels that client belongs to get 'pushed' down to the client by Pushie. Whee!

<b>Server Routing</b>

Pushie is a server. Writing a basic server in Go is really easy, so I was able to quickly move on to figuring out how that server should behave in more detail. To do that, I'm using the <a href="https://github.com/gorilla/mux" target="_blank">Gorilla Mux</a> URL router tool. 

Pushie has to be capable of all sorts of actions, not just sending messages. It needs to:

<ul>
<li> add and remove users from channels
<li> log users out, or kick them out
<li> add and remove whole channels, and check their health
<li> handle authentication tokens
<li> right, also publish messages

You can do that by establishing a particular url route for each action you need to take, and telling Pushie to run a particular bit of code when reading Web App messages from that URL.

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

When the incoming web request from Web App includes the URL extension "/foo" (argument one in each HandleFunc call), Pushie is told to execute the correlation function "BarHandler" (argument two in HandleFunc). That's it, just a simple map!

<b>Reading Messages from Web App</b>

As you can see in the router code, Pushie has a lot of functions. I chose to get started with PublishHandler, Pushie's core function: publishing those incoming messages from Web App. 

So how is Pushie going to do this? JSON decoding! 

When a message is sent from Web App to Pushie, it's an http POST request (POST is just a certain type of http request). That comes in the form of a stream of JSON bytes. In order for Pushie to read that message and understand how to handle it, it needs to convert the JSON bytes to something it can use: a Go object.

Check out this <a href="https://blog.golang.org/json-and-go" target="_blank">Go Blog</a> for a good overview of Go encoding & decoding for JSON. 

To get started, I have to do a lot of fake outs and mock ups of how this will work. For instance, while Pushie is being developed locally, I don't actually have any incoming web requests from a Web App. To fake it, I actually *encode* a Go object into JSON, and then pass that encoded piece into a decode_message function. Eventually, that argument won't be a fake, it'll be an actual stream of JSON bytes coming in as a web request:

<pre><code>package main

import (
  "encoding/json"
  "fmt"
)

//create mockup Message object
type Message struct {
   Channel string
   Data    map[string]interface{}
}

//create mockup message
var decoded_message = Message{
  Channel: "bunno",
  Data: map[string]interface{} {
    "message": map[string]string {"text": "oh hai", "username": "jufisch"},
  },
}

//'dummy' encode message to mimic JSon POST request
var encoded_message, err = json.Marshal(decoded_message)

//function to decode incoming messages
func decode_message(encodedmessage []byte) Message {
  var decodedmessage Message
  json.Unmarshal(encodedmessage, &decodedmessage)

  return decodedmessage
}</code></pre>

<b>Hash Lookups & Sending Messages</b>

I spent the last part of Pushie Day One: Publish Message fleshing out the lookup function that tells Pushie which clients it should push an incoming message to, based on which channel that message is for. 

As part of the Authentication and Authorization processes (yet another future post), Pushie will keep track of which users are currently logged in on each channel. It stores that data in a hash, or in Go speak, a map. They're very exciting data structures. You can try them out in the <a href="https://tour.golang.org/moretypes/19" target="_blank"> Tour of Go</a> or read more at the <a href="https://blog.golang.org/go-maps-in-action" target="_blank">Go Blog</a>. 

Web App sends messages to Pushie, either because a user told it to (ie a chat message was sent from a client) or code told it to (ie a program is using Pushie to send "Water your plants" reminders to users everyday at 12pm). The data in the message includes which *channel* it's going to be sent to.

Remember the decoding of the incoming message JSON data? Once Pushie reads the message and gets the target channel for that message, it looks in its hash of channels until it finds channel "bunno." The hash or map is a key-value store, and the channel names are our keys. For key "bunno," Pushie can now find what values are stored, and in this case, it's been storing those clients as values. Now it knows, for channel "bunno," users "foo" and "bar" are logged in. *This* is how Pushie now fulfills it's beautiful promise: sending the message to those users. Take a bow, Pushie.

Like the fake web request in the encoding discussion, here I'm faking the hash of channels by hard-coding it. When Pushie is further along, that will be a dynamic hash reflecting which users are currently logged in on which channels. 

<pre><code>package main

import (
  "encoding/json"
)

//create Message type
type Message struct {
   Channel string
   Data    map[string]interface{}
}

//create Websocket type -- these are the 'values' being stored in the hash, referred to as clients in the blog post.
type Websocket struct {
  id int
}

//create some dummy socket objects -- IRL these will be generated based on who is currently listening on Pushie channels
var Socket1 = Websocket{
  id: 1,
}

var Socket2 = Websocket{
  id: 2,
}

var Socket3 = Websocket{
  id: 3,
}

// create send method for Websocket -- this gives each value stores an action to send the message, for now it's also pretty fake
func (s Websocket) Send(message []byte) error {
  return nil
}

//create dummy Message
var decoded_message = Message{
  //this is the important part for Pushie
  Channel: "bunno",
  //this is the rest of the data that gets sent with a message
  Data: map[string]interface{} {
    "message": map[string]string {"text": "oh hai", "username": "jufisch"},
  },
}

//encode dummy message to JSON to mimic POST request
var encoded_message, err = json.Marshal(decoded_message)

//function to decode encoded messages (ie JSON bytes in POST requests)
func decode_message(encodedmessage []byte) Message {
  var decodedmessage Message
  //err := json.Unmarshal(encodedmessage, &decodedmessage)
  json.Unmarshal(encodedmessage, &decodedmessage)

  return decodedmessage
}

//finally, our lookup function. It takes the current hash or map as its first argument, and the incoming message as its second argument. It returns all the values for the given key (ie all the clients for a given channel).
func channel_lookup(currentmap map[string][]Websocket, message Message) []Websocket{

  target_sockets, ok := currentmap[message.Channel]
  if !ok {
    return []Websocket{}
  }

  return target_sockets
  //fmt.Println("Sockets in target channel: ", target_sockets)
}

func main() {

  //json bytes, our incoming message

  message_json := []byte (`{"Channel":"bunno","Data":{"message":{"text":"oh hai","username":"jufisch"}}}`)

  //hash for channels & sockets, hardcoded for now
  channels_hash := make(map[string][]Websocket)
  channels_hash["bunno"] = []Websocket{Socket1, Socket2}
  channels_hash["quadrupal"] = []Websocket{Socket1, Socket2, Socket3}
  channels_hash["thisisfine"] = []Websocket{}

  message := decode_message(message_json)

  target_sockets := channel_lookup(channels_hash, message)

  //now that we know our 'target sockets', we iterate over them and call the send method on each one, to send the message to each client with an open socket for the channel!

  for _, socket := range target_sockets {

    socket.Send(encoded_message)

  }

}</code></pre>

And, scene.     
