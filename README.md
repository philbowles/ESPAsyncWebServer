# ESPAsyncWebServer 

This is a patched and cut-down fork of [ESPAsyncWebserver](https://github.com/me-no-dev/ESPAsyncWebServer).

It is *required* for the author's other libraries:

1. [Pangolin MQTT](https://github.com/philbowles/PangolinMQTT)
2. [H4Plugins](https://github.com/philbowles/h4plugins)

---
# Pre-requisites

Several of the obvious, long-reported problems with the original library relating to the use of WebSockets and Server Sent Events (SSE) are actually caused by a bug in the same author's ESPAsyncTCP library. ESPAsyncWebserver doesn't help the matter because it contains its own bugs (which is what *this* repo fixes) that ought to spot the underlying issue, but don't, and cause one of:

* Inexplicable unexpected behaviour
* Timeout error
* Crash / reboot

Take your pick from the above almost at random: it's half the fun of using the original...

Therefore there is absolutely *no point whatsoever* downloading and installing this lib unless you also replace the buggy original ESPAsyncTCP with [this author's fixed version:](https://github.com/philbowles/ESPAsyncTCP-master)

---

# What's fixed

1. various random lockups, slo-o-o-o-o-w page loading and random crashes related to the underlying ESPAsyncTCP lib

2. The totally unworkable hard-coded limit of 8 SSE messages "in-flight", with no feedback, warning, notification or error-code to alert the user to the fact that numerous messages are randomly and silently discarded.

The same (surprising) "solution" is used in the Websockets code, rendering both that and SSE as worse than useless for any real-world application.

The removal of this arbitrary limit does however put the burden of flow control back to the user. Unless there is *some* degree of *sensible and practical* flow control, unlimited rapid messages from the user will cause a not-so-gradual depletion of the heap to the point of exhaustion / crash.

What constitutes "*sensible and practical*" is up to the user, but some guidance may be derived from the the way [H4Plugins](https://github.com/philbowles/h4plugins) manages the issue.

## The H4Plugins way

The critical scarce resource in the SSE/Websockets issue is the amount of free heap. Messages can only be despatched / cleared by the underlying TCP lib at a certain rate which depends on a number of factors beyond your control:

* Network speed / quality / latency
* Implementation buffer size / quantity
* User compile-time options re LwIP (mainly chaging the above point)
  
Irrespective of the limiting network factor, if the user sends SSE/socket messages faster than TCP can clear / ACK them, a "backlog" will build up. (This is the hardcoded limit mentioned above). The original lib just basically chopped / ignored / discarded anything over 8 with no error-code or notification, i.e your messages just "disappear" a) without trace b) without you even knowing it has happened, which is - in this author's view - totally unworkable and a very poor design / programming choice.

The real culprit is not the rate of sending, nor the rate of despatch, its the amount of free heap available to hold the "backlog" of unsent messages. If you run out of heap - even if sending a lot of large messages slowly - you will crash unless you limit the queue growth in some way.

Since the free heap is the limiting resource, [H4Plugins](https://github.com/philbowles/h4plugis) uses this to "throttle" the flow of messages. It has two (user-configurable) limits on the heap. When the heap drops below the lower limit, it stops sending messages and stays in this "fingers-in-ears" state until the heap has risen back above the higher limit.

The difference between the LO and HI limits is to give a "hysteresis window" in which the underlying code can safely despatch its queue, freeing up the heap and making room for new messages.

Two factors allow this to work and must be considered if implementing your own solution:

1. You cannot do any of this in a "tight loop". The underlying TCP and AsyncWebserver code must be allow to run - or it can't clear the queue - making things worse. Thus *your* code needs to yield or exit or "do stuff" in a background timer / separate task etc to give those other libs "breathing space". [H4Plugins](https://github.com/philbowles/h4plugins) has this fuctionality as an inherent part of its architecture, whcih is why this is a simple and "natural" solution for it, as long as...

2. ...there exists a strategy to cope with / replace / restransmit / othwerwise manage the ignored messages. This will - by defintion - be app-dependent. In [H4Plugins](https://github.com/philbowles/h4plugins), SSE messages are used solely to keep the web UI up-do-date / in sync with the state of the app. Therefore When the heap has recovered back above the (presumably "safe") higher limit, any code that previously sent an ignored SSE now receives an event message telling it to send/re-sync its *current* state.
   
This means that while, say , a flashing LED may not have displayed several on-offs...once SSE messages can be sent again it will correctly show the way things *are now* even though it may not have accurately represented the way things *were* in the last couple of seconds. In the real world, unless a user happens to have been watching both like a hawk at the exact moment, no-one is going to notice anyway.

On the other hand they usually ***do*** notice things like:

* Only the first 8 of a series of things that *ought to* happen *actually do* happen, which is fine when the "things" are visible: when they are not, then...
* Inexplicably odd behaviour occurs "out-of-the-blue"
* The interface seems to "hang"
* The app crashes randomly

# What's missing

Be warned that this is ***not*** a straightforward drop-in replacement for a buggy library. It is specifically tailored to allow [H4Plugins](https://github.com/philbowles/h4plugins) to function correctly. As such a number of subsections have been removed to "reduce weight"

* All references to `PROGMEM`: an outdated hangover from Arduino code: totally unnecessary when proper memory management techniques are used. 
* All reference to JSON: waaaaaaaaay too heavy a lib.
* All references to WebSockets

I feel a little guilty about the last one (but only *a little*) as I know many people would love a fully-working async WebSockets server. Ironically (and even amusing after a couple of years of trying to work round these problems) [H4Plugins](https://github.com/philbowles/h4plugins) only uses SSE as this author simply couldn't get WebSockets working all those years back without crashing the MCU :). Subsequent investigation (and much swearing and tearing out of hair) reveals it's the same high-school-newbie-quality hardcoded "fix" that causes the problem with both features. The [H4Plugins](https://github.com/philbowles/h4plugins) flow-control method outlined above would also work fine for webSockets, once the compounding ESPAsyncTCP bug is also factored out.

The sad truth is that I abandoned WebSockets long ago, have no use for them myself (I use AJAX/SSE) and simply do not have the time to "save the world" by providing a public working replacement for *all* the bugs in ESPAsynWebserver. Sorry and all...I'm available for hire though. :)

---

## Find me daily in these FB groups

* [Pangolin Support](https://www.facebook.com/groups/pangolinmqtt/)
* [ESP8266 & ESP32 Microcontrollers](https://www.facebook.com/groups/2125820374390340/)
* [ESP Developers](https://www.facebook.com/groups/ESP8266/)
* [H4/Plugins support](https://www.facebook.com/groups/h4plugins)

I am always grateful for any $upport on [Patreon](https://www.patreon.com/esparto) :)

(C) 2021 Phil Bowles