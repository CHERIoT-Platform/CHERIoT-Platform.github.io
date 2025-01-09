---
layout: post
title: "Using capabilities instead of trusting 3rd party code"
date:   2025-01-08
categories: rtos cheri
author: Phil Day
---

When getting started with CHERI it's easy to think of capabilities as just a better form of pointer that the compiler will set up and manage on your behalf.
And of course there are a lot of memory safety benefits that come for free from that, but the real value is unlocked when you start to manipulate capabilities directly to capture intent.
Let me illustrate with a simple example.

The publish method in the MQTT library in [cheriot-network](https://github.com/CHERIoT-Platform/network-stack.git) takes pointers for both the topic and the payload:

```c++
int __cheri_compartment("MQTT") mqtt_publish(Timeout    *t,
                                             SObj        mqttHandle,
                                             uint8_t     qos,
                                             const char *topic,
                                             size_t      topicLength,
                                             const void *payload,
                                             size_t      payloadLength,
                                             bool        retain = false);
```

and has the following comment about their scope:

_Both the topic and payload buffers must remain valid during the execution of this function._
_If the caller frees them during the execution of this function, the publish may leak application data to the broker through the topic and payload._

Ok, so it seems clear that I only need to make sure that they remain valid and unchanged until the call returns, but I was hitting a bug that seemed to suggest they might be being used outside of that by another thread in the network stack.
_It turns out I was wrong about that BTW, but anyway I'm a suspicious kind of guy when it comes to trusting external modules._
A quick look through the code quickly persuaded me that a) I'm not very good at reading other peoples code and b) it was probably not the issue but I still wanted to rule it out.

Luckily by thinking of these in terms of capabilities rather than pointers I can directly implement my intent rather that trust what the library is telling me.
Specifically I can derive new capabilities that are read only and can't be captured.
So I can be guaranteed that at the end of the call nothing in the network stack can have modified the message or saved a reference to it.
And for good measure I can also limit the bounds so that I don't have to trust that the MQTT code will only read within topicLength and payloadLength.
This is what the resulting code looks like in my demo:

```c++
// Publish a string to the status topic
void publish(SObj        mqtt,
             std::string topic,
             void       *status,
             size_t      statusLength,
             bool        retain)
{
	Timeout t{MS_TO_TICKS(5000)};

	// Use capabilities with only Load
	// permission so we can be sure the MQTT stack
	// doesn't capture a pointer to our buffer.
	CHERI::Capability roTopic{topic.data()};
	roTopic.permissions() &= {CHERI::Permission::Load};
	roTopic.bounds().set_inexact(topic.size());

	CHERI::Capability roStatus{status};
	roStatus.permissions() &= {CHERI::Permission::Load};
	roStatus.bounds().set_inexact(statusLength);

	auto ret = mqtt_publish(&t,
	                        mqtt,
	                        1, // QoS 1 = delivered at least once
	                        roTopic,
	                        topic.size(),
	                        roStatus,
	                        statusLength,
	                        retain);

	if (ret < 0)
	{
		Debug::log("Failed to send status: error {}", ret);
	}
}
```

Note that when setting the bounds of a capability there may be rounding considerations to account for alignment.
Using `set_inexact()` will return a valid capability that will be rounded up as needed.
Using the following would instead return an invalid capability if `statusLength` could not be exactly represented in the bounds of a capability. 

```c++
   ...
   roStatus.bounds() = statusLength;
   ...
```


So by explicitly using the extra controls that CHERIoT provides we can eliminate any trust relationship between our code and some third party module and its compilation environment.
The CHERI processor will guarantee that the MQTT library can only use the pointers we supply in the way we intend it to, even if the library is implemented to make use of compiler escape hatches such as "unsafe".
And that's a significant step forwards in creating safe and reliable systems.
