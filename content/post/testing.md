---
title: "Testing with httptest"
date: 2017-08-05T12:31:12-04:00
---

I'm forging ahead with Pushie, my real-time push notification server. The first step was setting up Gorilla Mux to handle routing. Each route then has an associated handler function, which, when called, executes one of Pushie's main functions.

I decided to start by working out PublishHandler, the function called when a web app sends a request to pushie at the URL path "/publish." Guess what it does? It publishes a message from the web app to any clients listening on that channel. I thought I would just blow through each handler function one by one until Pushie was complete. But once I *really* got into the weeds of how each function worked, I discovered that a better second step to "just jump in with development" was "just jump in with tests that will help guide your development." 

Ok, ok, in actual fact, I'm incredibly stubborn. I was initially *very* hesitant to stop the clear forge-ahead work of jamming through each handler function. Writing tests!? More work instead of less!? Surely that's going to slow me down and be fruitless. Thankfully, someone wiser than me encouraged me to get over my stubborn gut-instinct, and oh how right that person was!

Writing tests alongside your main code provides you with something akin to a built-in sounding board. You must enumerate your goals in the test, and the test will - if written well - let you know if your code is meeting those goals or not. I like to think that if I was attacking the code without tests, I would do so by taking a systematic, step-by-step approach to what any given function needs to do. But instead of simply trusting that I'll do this, I can do my step-by-step thinking through of functionality "out loud" in the test file instead. 

Testing also sets up a natural barrier for something I've discussed before: the need to 'fake' things that your code will eventually have access to - a hardcoded hash instead of something generated dynamically, or a web request I've written by hand rather than one sent by an actual web app. Establishing a test file gives you a natural place to store all these items. You can 'fake the world' in your test, so your code is much more aligned with the truth of what it will actually need to be. Then, you can slowly chip away at those 'fake' things in your test, or move away from the test paradigm at all and simply run your code in the final environment you're anticipating... if you've been testing well, the switch from running tests to running live should be far more of a table-cloth trick maneuver than a mess of broken dishes and dropped cutlery.

![Gotta love a good tablecloth trick](img/tablecloth.gif)

Using the test tools included in Go's standard library has been delightful and very easy. I thought I would give a quick rundown here.

<b>Go's Package "Testing"</b>

Go automates a lot of the testing machinery in its "Testing" package. The setup is incredibly simple: just create a file with name ending in "_test.go" and put it in the same package being tested (i.e. if your Go code is in package main, open your _test.go file with package main as well). Then import the "testing" package. Now you can use a particular function signature to test functions in your main code. That signature looks like:

<pre><code>func TestXxx(*testing.T)</code></pre>

Here's an example of how these things pair up from Pushie. This is a bit of code from Pushie.go, my main project file. The relevant details here are: I'm in package main, and I'm working on a function called PublishHandler.

<pre><code>package main

...

func PublishHandler(w http.ResponseWriter, req *http.Request) {
...
}</code></pre>

My test file is named Pushie_test.go, following the "_test.go" convention. It's in the *same* package as my code to test (package main), and I've named a testing function TestPublishHandler to communicate with PublishHandler, again following the Testing package's convention:

<pre><code>package main

import (
  ...
  "testing"
  ...
)

...

func TestPublishHandler(t *testing.T) {

...

}</code></pre>

That's it! Now the code inside TestPublishHandler can be implemented to test PublishHandler. Remember the function signature of PublishHandler above? That tells me at some point I'll have to create an http.ResponseWriter and an *http.Request pointer, and then pass those to PublishHandler inside TestPublishHandler. Whee.

<b>Migrating Fakes to Tests</b>

When I first started, I was 'faking' or 'mocking up' a bunch of hard-coded things like hashes of Pushie's channels and sockets, pretend socket objects, and an example message, because PublishHandler will eventually need to refer to those things but I didn't have them implemented yet (most of that info comes from the Web App, which doesn't exist yet.) As mentioned above, one of the nice things about testing is that I can move all my 'fakes' or examples into the test file. It provides an obvious and useful way to keep my code organized: get Pushie.go running just as it would in the live production scenario, and keep fake-out data in the test file. 

Another reason this is helpful is that you can more rigorously use your tests to *test* because you can change some of the hardcoded data values you're passing. Perhaps you've hardcoded something to pass a value of "1." You *expect* it to be the same if you pass it "-1," but instead of futzing with your code file, you can alter some of the values in the test file to check those assumptions.  by changing some of those hard-coded items. In Pushie's case, the fake message now lives in my test file. I can run the test to make sure the message sent by my test request is the same one mirrored back to me by Pushie in it's response, and then I can change the text of the message and run my test again to make sure changing messages don't break my Pushie code.

<b>fmt.Println Is Your Friend</b>

The Go compiler offers a lot of helpful (let's say "helpful;" sometimes it makes you want to scream, but ultimately it is helpful) information when you've got something wrong in your code. Even so, I've found there's no substitute for adding debugging information to your code and printing it to the screen so you can follow along with the action. I used to code this way in Ruby and Javascript and it's just so informative when you're looking for issues. 

In Go, that command is fmt.Println (this is akin to console.log in javascript or puts in Ruby). I use fmt.Println *all* the time - just remember to import "fmt". At one point I was printing out the full story of what was going on in Pushie, and all of the variable values, just so I could walk through my terminal output and get a feel for what was happening and why something wasn't working.

The same holds true when you have tests set up: I just output that much more extra info from Pushie_test.go as I worked from failing to passing tests. I'm not sure this is a good practice or not, but I started indicating where the printed line was coming from to make it easier to follow along. Any fmt.Println statements in my main code began "Pushie.go says:" and those in the test files were labeled "Pushie_test.go says:..."

<b>State of Pushie Update</b>

So what is Pushie actually able to do now? Not tons, honestly. But I'm excited to have a good clean test for PublishHandler that passes. I want to walk through the code a bit here. 
Here's a peek inside PublishHandler:

<pre><code>func PublishHandler(w http.ResponseWriter, req *http.Request) {

	decoder := json.NewDecoder(req.Body)
	for {
		var m Message
		if err := decoder.Decode(&m); err == io.EOF {
			break
		} else if err != nil {
			log.Fatal(err)
		}
		// Get sockets for the channel from channelsHash.

		sockets := ChannelLookup(channelsHash, m.Channel)

		// Count the number of sockets listening.
		sentCount := len(sockets)

		// Build a response object.

		responseObject := PublishResponse{
			NumSent: sentCount,
			MesSent: m.Data,
		}

		// Marshal the response object into a byte slice of JSON data.
		responseByte, err := json.Marshal(responseObject)
		if err != nil {
			fmt.Println("error:", err)
		}

		//fmt.Println("This is the response as bytes: ", responseByte)

		// For socket in sockets
		for _, socket := range sockets {

			// To be replaced with socket.Send
			fmt.Println("Sending to socket ", socket)
			// Check for error (which disrupts sentCount)
		}

		// Send response to web app.
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		w.Write(responseByte)
		w.WriteHeader(http.StatusOK)
	}
}

</code></pre>

Right now, I have PublishMessage build two pieces of information into its response (alongside the basic Content-Type header): the message text that was sent (m.Data) and the *number* of sockets that it sent the message to (sentCount). 

Warning: There's a big cheat here, which I'm going to have to come back to later. Right now, sentCount is just set equal to len(sockets), the length of the list of socket objects stored as a value to the name of the channel (the key) in the channels hash. So basically, channels hash says "Hey there are three sockets in this channel" and PublishHandler says "Ok, sentCount equals three!" Why is that cheating? Because sentCount doesn't *actually* count the number of sends. (Heyo!) When this is complete, sentCount will truly be a counter of the for loop that iterates through each socket and calls socket.Send. That way, if I try to send a message to Socket #2 of 5, and I get an error back, I can (a) return an error (b) use sentCount to show that it wasn't equal to the socket number in the channels hash, i.e. that it failed to send the message to the correct number of sockets.

But I digress. Let's focus on the fact that PublishHandler sends a response including number of sockets it sent to and the message content. That let's me implement a test of PublishHandler that says "Make sure that we hear a response from PublishHandler that says it returned the # of sockets we know is in the channel, and the text of the message we asked it to send."

Here's how I get to that "make sure" step with Pushie_test.go:

<pre><code>func TestPublishHandler(t *testing.T) {

	var Socket1 = Websocket{
		id: 1,
	}

	var Socket2 = Websocket{
		id: 2,
	}

	var Socket3 = Websocket{
		id: 3,
	}

	fmt.Println("running TestRoutes")

	// Set up body with message.
	message := Message{
		Channel: "bunno",
		Data: map[string]interface{}{
			"message": map[string]string{"text": "oh POOP", "username": "jufisch"},
		},
	}

	// This pulls out the content of the example message and sets them as variables named for our *expected* response content.
	expectedResMesText := message.Data["message"].(map[string]string)["text"]
	expectedResMesUser := message.Data["message"].(map[string]string)["username"]

	// Encode message to messagebytes ([]byte).
	mbytes, _ := json.Marshal(message)
	//fmt.Println(mbytes)
	//message_bytes := []byte(`{"blah": "blah"}`)
	mbuffer := bytes.NewBuffer(mbytes)

	// Set up the request with the path and the message.
	req, err := http.NewRequest("POST", "/publish", mbuffer)
	if err != nil {
		t.Fatal()
	}

	// Set up the channels hash.
	channelsHash["bunno"] = []Websocket{Socket1, Socket2, Socket3}
	channelsHash["quadrupal"] = []Websocket{Socket1, Socket2, Socket3}
	channelsHash["thisisfine"] = []Websocket{}

	// Create a recorder to pass in as our ResponseWriter.
	rr := httptest.NewRecorder()

	PublishHandler(rr, req)

	// Create a response object to store the unmarshaled JSON from response received.
	var res PublishResponse

	// Unmarshal the json from responseRecorder into var res
	err = json.Unmarshal(rr.Body.Bytes(), &res)
	if err != nil {
		fmt.Println("error:", err)
	}

	// Assign number of sockets in called channel to socketsInChannel.
	socketsInChannel := len(channelsHash[message.Channel])

	// Assert that the number of socket.Send calls in the response is equal to # of sockets in appopriate channelHash channel.
	assert.Equal(t, res.NumSent, socketsInChannel, "These should be equal ints.")

	// Assert that responseObject.MesSent == message.Data

	
	// Get the message text out of the response.
	resMesText := res.MesSent["message"].(map[string]interface{})["text"]
	// Get the message username out of the response.
	resMesUser := res.MesSent["message"].(map[string]interface{})["username"]

	//Assert that the message content (text and username) is equal in original message & message mirrored in response.
	assert.Equal(t, expectedResMesText, resMesText, "These should be equal message text values.")
	assert.Equal(t, expectedResMesUser, resMesUser, "These shoudl be equal message username values.")
}</code></pre>


Ok! I learned a ton while setting this up. It's important to note that I'm using two additional testing packages for this:

<ul>
  <li>Package <a href="https://golang.org/pkg/net/http/httptest/" target="_blank">"httptest"</a>, a test suite for Go's http package in the Go standard library</li>
  <li><a href="https://github.com/stretchr/testify" target="_blank">Testify</a>, a toolkit for testing that works well with Go's standard library</li>
</ul>

The TestPublishHandler function creates some socket objects and a message object, the example content I was talking about earlier. It also assigns values into the channelsHash map, which populates the main channelsHash variable in Pushie.go for Pushie to refer to. 

Then it creates a "NewRecoder," which is a handy item from the httptest suite. The recorder can be passed into an http handler func that's looking for an http.ResponseWriter. The recorder simply stores (or records) whatever response is being written by PublishHandler's response writer, without truly *being* a response writer. We also create the second argument to be passed to PublishHandler, an http.Request object. What is the request that will trigger the PublishHandler function? Right! A POST request sent to the URL path "/publish." Huzzah! 

Once we have those things contstructed and established, we're ready to call PublishHandler, but we want to implement the logic of the test. I use Testify's "assert.Equal" method to make sure that what I expect in the test suite is what I get back (i.e. is equal) from Pushie's response (remember, this will eventually come from a true ResponseWriter, but right now the response is stored in my Recorder). I have three assert.Equal statements, checking that:

<ol>
  <li>Pushie sent a message to the correct number of sockets</li>
  <li>Pushie's response message data included the correct message text</li>
  <li>Pushie's response message data included the correct message username</li>
</ol>

<br>
Whew. One thing you might be wondering: why do I check both message text and message username, instead of simply checking all the message data stored in the hash at one time? To be honest, this is a bit of a hack, because I was getting a slight mismatch of the type of message data between PublishHandler and TestPublishHandler (one was a map[string]interface{} and one was a map[string]string, so even though they had the same inner content, which is what we care about, the type was different and caused the tests to fail.)

So that, in what would never even remotely fit inside a nutshell, is what I've been working on this week. 

<b>TL;DR</b>

I'm now able to pass a test request and response recorder to PublishHandler, and ensure that the JSON data comes back correctly! Yay! 

One handler function down, about 13 to go ;)
  
