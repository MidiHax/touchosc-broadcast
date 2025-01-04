# touchosc-broadcast

This repo contains TouchOSC Lua script code that enables support for sending messages (via TouchOSC's notify mechanism) to one or more "subscribers". Subscribers are controls within the panel that listen for broadcast messages via the _onReceiveNotify()_ handler.

The broadcast logic is implemented as a _topic-based_ pub/sub bus, similar to RabbitMQ but not nearly as sophisticated. For more information on message topics, see [this helpful documentation](https://www.rabbitmq.com/tutorials/tutorial-five-python).

> _Usage of this code assumes familiarity with TouchOSC's Lua scripting model._

## Use Case

Why would you need this? One important goal of messaging-based architectures is to reduce coupling. In other words, the controls within your panel should be, generally speaking, independent and not have "knowledge" of other controls or their behaviors. Now, this is very academic and reality often has different ideas but if you strive to reduce coupling you gain some benefits around reducing code complexity and interdependence and you can often improve performance.

Let's consider a practical example. Say you are building a panel to control a sequencer. Your panel might have Play, Stop, and Pause buttons. When clicked, the panel will take many actions. The Play button might need to send a Start MIDI message, might need to enable or disable other controls in the panel, and might need to reset variables and counters used by script. You can cram all that logic in the Play button... and then often have to duplicate with slight variations in the Stop and Pause as well. The code starts to get messy very quickly. If you change something in your panel design, you have to remember to go into the Play, Stop, and Pause button code and make corresponding changes there too!

Now imagine that your panel is also receiving MIDI ticks in order to update the bar/beat counter and other UI elements. Your MIDI handler callback will have a large amount of code to control various UI elements and, as above, if you need to change it you'll have to remember to update all your MIDI handlers as well.

One solution to this problem is to use a messaging architecture. Now, when the Play button is pressed, the button will simply broadcast to the panel "Hey! We're in Play mode now! Everyone do your thing!". Any controls or script logic that care about Play mode (or any other mode) can update themselves accordingly and don't care what all the other controls are doing. Coupling is greatly (or completely) reduced and everything is simpler to understand and maintain.

The same applies to MIDI or OSC event handlers or any control value change events. When you get an event, you can perform any handling logic and then simply broadcast to the panel what happened and include any relevant data (for example, the bar/beat count in the case of a sequencer) along with the message.

## Getting Started

The main logic for the pub/sub system should be placed at the root of your TouchOSC panel. You will need to include 2 methods, a local variable, and two standard callback handlers (_init()_ and _onReceiveNotify()_):

```
local broadcastSubs = {}

function init()
  broadcastSubs = {}
end

function onReceiveNotify(key, data)
  if (key == "broadcast") then
    broadcast(data.topic, data.message)
  end
  
  if (key == "broadcast-subscribe") then
    broadcastSubscribe(data.topic, data.controlName)
  end
end

function broadcastSubscribe(topicPattern, controlName)
  local ctrl = self:findByName(controlName, true)
  
  if (ctrl == nil) then
    print("broadcastSubscribe: Unknown control: " .. controlName)
    return
  end
  
  table.insert(broadcastSubs, {topicPattern = topicPattern, control = ctrl})
  
  print("Control '" .. ctrl.name .. "' has subbed to to topic: " .. topicPattern)
end

function broadcast(topic, message)
  print("broadcast", topic, message)
  for i = 1, #broadcastSubs do
    local idx = string.find(topic, broadcastSubs[i].topicPattern)
    
    if (idx ~= nil) then
      print (broadcastSubs[i].control.name, idx)
      broadcastSubs[i].control:notify(topic, message)
    end
  end
end
```

Next, we need to setup some listeners, which are just the controls in your panel. Without listeners, any messages you broadcast are simply sent to the void and are never received by anyone.

Setting up a listener is easy:

```
function init()
  root:notify("broadcast-subscribe", { controlName = self.name, topic = "test" })
end
```

All that is necessary is sending a _notify()_ event to the root indicating the topic we want to listen for (more on that later) and the relevant control name.

Subcribe should only be done once and should generally be done during the _init()_ handler since it is an expensive operation and would impede performance if done elsewhere.

Now that we're subscribed to the "test" topic, how do we get the messages? All messages are delivered to controls via the _onReceiveNotify()_ handler:

```
function onReceiveNotify(key, data)
  print("Received message: " .. key, data)
end
```

This simple function doesn't really do anything useful, but here is where you would implement logic in response to messages. For example, if you had a fader that controlled some synth parameter, you might want to enable/disable that fader if it isn't relevant for certain patch editing modes. Here is where you would implement that logic in response to messages about patch edit mode updates.

## Broadcasting

The last piece of the puzzle is actually broadcasting a message. This is very simple to do:

```
root:notify("broadcast", { topic = "transport|stop", message = {} })
```

This will broadcast a message with the topic "transport|stop". Notice the pipe character. That is a topic delimeter. It is used to group messages into a logical heirarchy so that listeners can only receive the messages they actually care about. A complex panel might have thousands of messages firing off. You don't want every control getting every message! The topic specifies what the actual message is about, so controls can subscribe only to the ones they are concerned about.

The _message_ property can be any Lua data (usually a table) and will be sent along to the _data_ parameter of the _onReceiveNotify()_ handlers for all subscribers to this specific message topic. This is how you include a message payload (message data) as part of the broadcast.

## Message Topic Wildcards

Wildcards are where the real power and flexibility of topic-based message routing are apparent. With wildcards, you can subscribe to a particular pattern of messages so that your listener only gets messages it cares about. Using our sequencer example, we might format our Play, Stop, and Pause messages using the following strings:

```
Sequencer|Transport|Play
Sequencer|Transport|Stop
Sequencer|Transport|Pause
```

If we wanted to listen to all the sequencer transport control messages, we would subscribe using the Lua wildcard *%w* - which matches all alphanumeric characters:

```
function init()
  root:notify("broadcast-subscribe", { controlName = self.name, topic = "Sequencer|Transport|%w" })
end
```

With this subscription, our control will get notified of all Sequencer Transport messages.

Now consider our sequencer has another subsystem, the playhead display. We could setup messages for that as well:

```
Sequencer|PlayHead|CurrentBeat
Sequencer|PlayHead|CurrentBar
Sequencer|PlayHead|CurrentTick
```

Now we could subscribe to all the PlayHead messages (_Sequencer|PlayHead|%w_) so that we can update our display as playback position changes. Or, if we had a control that displayed only the current Bar/Measure number, we could subscribe to just that one specific message and ignore the rest.

When creating subscription patterns, the matching wildcards available are standard in Lua and are defined [here](https://www.lua.org/pil/20.2.html).

# Links
 * [Hexler TouchOSC](https://hexler.net/touchosc)
 * [TouchOSC Scripting API](https://hexler.net/pub/touchosc/scripting-api.html)
