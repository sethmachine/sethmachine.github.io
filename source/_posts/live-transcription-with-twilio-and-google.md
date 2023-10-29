---
title: Live Transcription with Twilio and Google using Java
date: 2023-09-09 15:05:27
tags: 
- Twilio
- Transcription
- AI
- Calling
- Google
- Gcloud
- Speech to Text
- Java
- Dropwizard
- Websockets
---

![Placeholder for live speech to text image](placeholder.jpg)

## Introduction

This article provides a detailed guide on how to create your own Java live transcription service with Twilio.  There already exist guides for [Python](https://www.twilio.com/blog/transcribe-phone-calls-text-real-time-twilio-vosk) and [Node.js](https://www.twilio.com/blog/live-transcribing-phone-calls-using-twilio-media-streams-and-google-speech-text) but no such equivalent guide in Java exists.  Live speech to text (or live transcription) is the capability to turn spoken human language into text on demand as each person talks.  This capability powers many well known products today like Amazon's Alexa or Apple's Siri.  Live transcription contrasts with batch or offline transcription, which operates on already made recordings.  The online (live) nature of this makes it a harder problem and requires a different software engineering approach than the usual [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) approach applicable to simpler systems.

Before getting started, you'll need the following accounts or tools installed:
* [A Twilio account](https://www.twilio.com/try-twilio)
* [A Google Cloud account](https://cloud.google.com/free)
* [IntelliJ Java IDE or equivalent](https://www.jetbrains.com/idea/download/)
* [Maven](https://maven.apache.org/)
* [ngrok](https://ngrok.com/docs/getting-started/)
* [PieSocket WebSocket Tester](https://chrome.google.com/webstore/detail/piesocket-websocket-teste/oilioclnckkoijghdniegedkbocfpnip)

## WebSockets Approach

Live transcription's real time nature is challenging and requires using real time communication protocols that vanilla HTTP cannot accomplish easily.  End users or applications consuming the live transcription output will expect results nearly simultaneously as a person speaks, otherwise it is not really live transcription at all.  Enter [WebSockets](https://en.wikipedia.org/wiki/WebSocket), which allows two way communication between a server and client in real time with arbitrary data.  Using WebSockets will allow for meeting the demanding latency requirements for a live speech to text application.  Thankfully, Twilio already uses the same WebSockets technology to provide live streams of audio bytes through [Media Streams](https://www.twilio.com/docs/voice/api/media-streams).  The Java web service we write will capture these audio bytes and send them to Google Cloud Speech To Text to get written natural language output.   

At a high level, the approach consists of the following steps:
1. When a phone call comes in, open a WebSocket connection to Twilio Media Streams
2. Create a separate thread to stream Twilio audio bytes to Google Cloud Speech to Text
3. Create a new WebSocket connection and begin publishing Google Cloud transcription results
4. End user or application listens to the previous WebSocket connection and displays transcription results

Each phone call will create 2 separate WebSocket connections: one connection for receiving the audio bytes from Twilio Media Streams, and a second WebSocket connection for publishing the Google Cloud transcription results.  This guide will omit [security/identity/authentication issues](https://en.wikipedia.org/wiki/WebSocket#Security_considerations) and focus primarily on functional live transcription: from phone call speech to natural language text in real time.  

![Placeholder for systems diagram](placeholder.jpg)

## Setting up Java web server

In this section we will set up foundations for the Java web server that will handle incoming phone calls, the creation of the WebSocket connections, and streaming live transcription results.  We will use the [Dropwizard framework](https://www.dropwizard.io/en/latest/getting-started.html) for the Java web server, as it has extensions that allow it to act as a WebSocket server.  It is possible to substitute Dropwizard with a different framework as long as it supports WebSockets. 

To simplify the initial setup and boilerplate, I have created a repo, [dropwizard-guice-template](https://github.com/sethmachine/dropwizard-guice-template) that already has much of these completed in order to keep the focus on the business logic for live transcription.  The steps below assume using IntelliJ IDEA Community Edition.  

1. Clone the template repo: `git clone https://github.com/sethmachine/dropwizard-guice-template`
2. Open the project `dropwizard-guice-template` in IntelliJ.  It should appear as shown in the screenshot below:
   ![Dropwizard Guice Template in IntelliJ](dropwizard-guice-template-in-intellij.png)
    There are several files here and packages.  Some of the key ones are:
    * `DropwizardGuiceTemplateApplication.java`: the main entry point for starting the server 
    * `dropwizard-guice-template.yml`: YAML runtime configuration for the server (ports, logging, etc.)
    * `resources/SampleHttpResource.java`: a simple GET endpoint
    * `resources/SampleWebsocketResource.java`: a websocket endpoint handling connections, messages, and disconnects
3. Select Eclipse Adoptium 17.03 as the project Java SDK under File > Project Structure > Project Settings > Project.  Hit Apply and OK after selecting.
   ![Select Eclipse Java SDK](select-java-runtime-for-dropwizard-app.png)
4. Run the server:
    1.  Open the Modify Run Configuration menu next to `DropwizardGuiceTemplateApplication`
        ![Open Run Configuration](open-run-configuration.png)
   2. Add the following to the CLI arguments: `server $ProjectFileDir$/dropwizard-guice-template.yml`
      ![Edit Run Configuration](edit-server-run-config.png)
   3. Click Apply then OK to save the changes.  
   4. Run the server in IntelliJ (open the same menu as Modify but click Run instead)
      ![Run Server Button](run-server.png)
   5. Verify the server runs in the console.  You should see output like below:
      ![Server Running in the Console](running-server-in-console.png)
5.  Using an HTTP client like `curl`, verify the sample HTTP resource is working:
    ```shell
    curl -X GET localhost:8080/sample/http
    Hello world!
    ```
6.  (Optionally) Rename any of the files to have the `TwilioLiveTranscriptionDemo` prefix.  For the remainder of the guide I will be referencing files and packages like `TwilioLiveTranscriptionDemoApplication.java` instead of `DropwizardGuiceTemplateApplication.java`


## Handle Inbound Calls

With the basic server now running, we will need to configure it to handle incoming phone calls from Twilio and respond with instructions to stream the audio bytes to a websocket server.  Every purchased phone number in Twilio can be assigned an external webhook endpoint which is triggered when certain events happen, such as an incoming phone call.  We will use this webhook to instruct Twilio how to respond to incoming calls, and ultimately to get for live transcription output.

First, create a new HTTP resource to handle incoming calls from Twilio called `TwilioInboundCallWebhookResource` as shown below:

```java
package io.sethmachine.twiliolivetranscriptiondemo.resources;

import javax.inject.Inject;
import javax.ws.rs.Consumes;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.HttpHeaders;
import javax.ws.rs.core.MediaType;

@Path("/twilio/webhooks/inbound-call")
@Produces(MediaType.TEXT_XML)
@Consumes(MediaType.TEXT_XML)
public class TwilioInboundCallWebhookResource {

  private static final String WEBSOCKET_CONNECT_PATH = "twilio/websocket/audio-stream";

  @Inject
  public TwilioInboundCallWebhookResource() {}

  @GET
  public String getTwiml(@Context HttpHeaders httpHeaders) {
    String websocketUri = buildWebsocketUri(httpHeaders);

    return String.format(
      "    <Response>\n" +
      "      <Start>\n" +
      "        <Stream url=\"%s\"/>\n" +
      "      </Start>\n" +
      "      <Say>This calling is being recorded.  Streaming 60 seconds of audio for live transcription.</Say>\n" +
      "      <Pause length=\"60\" />\n" +
      "    </Response>",
      websocketUri
    );
  }

  private static String buildWebsocketUri(HttpHeaders httpHeaders) {
    String hostName = httpHeaders.getRequestHeader("Host").get(0);
    return String.format("wss://%s/%s", hostName, WEBSOCKET_CONNECT_PATH);
  }
}
```

This HTTP resource does the following:

* Returns [TwiML](https://www.twilio.com/docs/voice/twiml) to provide instructions back to Twilio on what to do with the incoming call.  The TwiML tells Twilio to do the following:
  * Stream the audio to a specified websocket URI (constructed in `TwilioInboundCallWebhookResource#buildWebsocketUri`)
  * Play a text to speech message informing the caller that 60 seconds of the call will be recorded and transcribed
  * End the call after 60s have passed.  
* Constructs a URI to `"twilio/websocket/audio-stream"` websocket endpoint.  We will create this in the next section.  

We can verify this endpoint works as expected by running the server locally and hitting it with `curl -X GET localhost:8080/twilio/webhooks/inbound-call`:

```xml
<Response>
  <Start>
    <Stream url="wss://localhost:8080/twilio/websocket/audio-stream"/>
  </Start>
  <Say>This calling is being recorded.  Streaming 60 seconds of audio for live transcription.</Say>
  <Pause length="60" />
</Response>
```

In order to use it with a real phone call, we will first need to expose our local service to Twilio via ngrok and then purchase a phone number from Twilio.

## Handle Websocket audio stream

Now we will create the first websocket endpoint in our server to handle incoming audio streams from Twilio.  The nature of the stream is the audio byte representation of any sounds or voice made during the phone call.  Ultimately we will be implementing a websocket endpoint that implements the contract of [TwiML™️ Voice: \<Stream\>](https://www.twilio.com/docs/voice/twiml/stream).  

The initial websocket resource looks like the following:

```java
package io.sethmachine.twiliolivetranscriptiondemo.resources;

import com.codahale.metrics.annotation.ExceptionMetered;
import com.codahale.metrics.annotation.Metered;
import com.codahale.metrics.annotation.Timed;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.sethmachine.twiliolivetranscriptiondemo.guice.GuiceWebsocketConfigurator;
import java.io.IOException;
import javax.inject.Inject;
import javax.websocket.CloseReason;
import javax.websocket.OnClose;
import javax.websocket.OnMessage;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.ServerEndpoint;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Metered
@Timed
@ExceptionMetered
@ServerEndpoint(value = "/websocket", configurator = GuiceWebsocketConfigurator.class)
public class TwilioAudioStreamWebsocketResource {

  private static final Logger LOG = LoggerFactory.getLogger(
    TwilioAudioStreamWebsocketResource.class
  );

  private ObjectMapper objectMapper;

  private Session session;

  @Inject
  public TwilioAudioStreamWebsocketResource(ObjectMapper objectMapper) {
    this.objectMapper = objectMapper;
  }

  @OnOpen
  public void myOnOpen(final Session session) throws IOException {
    LOG.info(
      "[sessionId: {}] Websocket session connection opened: {}",
      session.getId(),
      session
    );
    session.getAsyncRemote().sendText("Ready to receive live transcription results");
    this.session = session;
  }

  @OnMessage
  public void myOnMsg(final Session session, String message) {
    LOG.info("[sessionId: {}] Got message: {}", session.getId(), message);
  }

  @OnClose
  public void myOnClose(final Session session, CloseReason cr) {
    LOG.info("Closed connection! reason: {}, session: {}", cr, session);
  }
}
```

We will also need to update `TwilioLiveTranscriptionDemoApplication.java` and change the `ServerEndpointConfig config` to match the new websocket resource path like so:

```java
    // NOTE: supplier is required to allow for lazy initialization of the guice injection
    final ServerEndpointConfig config = ServerEndpointConfig.Builder
      .create(TwilioAudioStreamWebsocketResource.class, "/twilio/websocket/audio-stream")
      .configurator(new GuiceWebsocketConfigurator(() -> guiceBundle.getInjector()))
      .build();
```

This websocket resource does the following:

* Exposes a websocket endpoint via `"/twilio/websocket/audio-stream"`, which can be accessed locally via `ws://localhost:8080/twilio/websocket/audio-stream`.  
* Implements `@OnOpen`: when a first websocket connection is made, this block of code is triggered.  
* Implements `@OnMessage`: whenever a message is sent (from the client to the server), this block of code is triggered.  In our case, Twilio send over [Media Messages](https://www.twilio.com/docs/voice/twiml/stream#message-media) which include raw audio bytes.  Right now this is represented as a raw String but we'll change this later in the guide to a more meaningful object representation.  
* Implements `@OnClose`: executed when the connection is closed.  This can be useful for logging and cleaning up resources.  
* Declares a private `Session` object.  Each session is stateful and represents an entire websocket connection from its beginning to end.  This object will be key to send back transcription results to the client/end user. 

We can verify the websocket resource is working as expected by using the Chrome extension [PieSocket WebSocket Tester](https://chrome.google.com/webstore/detail/piesocket-websocket-teste/oilioclnckkoijghdniegedkbocfpnip):
1. Run the Java server locally (via `TwilioLiveTranscriptionDemoApplication`)
2. Open PieSocket WebSocket Tester in Chrome browser.  
3. Enter the websocket URL `ws://localhost:8080/twilio/websocket/audio-stream` and hit Connect:
   ![Connect Pie WebSocket to localhost](connect-pie-websocket.png)
4.  Observe the following in the PieSocket console:
   ![Pie WebSocket Initial Connection Basic](pie-websocket-connected-basic.png)
5. At the same time, there should be output in the Java server console:
   ![Server Initial Output when connecting via websocket](server-websocket-connection-initial-output.png)

Later on we will add proper Java objects to model all the messages coming in from Twilio, and then build logic to send these to Google Cloud Speech to Text for live transcription.  

## Expose server with ngrok

[ngrok](https://ngrok.com/docs/getting-started/) allows for exposing a local service to the public internet.  This step is necessary because Twilio needs to hit publicly exposed webhooks when an incoming phone call comes in to get further instructions.  While this may seem insecure, the URLs that ngrok generates for forwarding are hard to guess and thus provide security through obfuscation.  In addition, each time ngrok is run, a different public URL will be generated, making it more secure.  

After ngrok is installed, run ngrok and verify it is correctly exposing the server:

1.  First, run the Java server (via running `TwilioLiveTranscriptionDemoApplication` in IntelliJ)
2. On a separate command line, run `ngrok http 8080`.  This will expose port 8080 publicly.  
3. From the ngrok output, find the public forwarding URL (see below image):
   ![Ngrok running with forwarding URL](ngrok-running-first-time.png)
4. In this example, the URL is `https://31d0-2601-189-8000-c3f0-7c5c-caaa-3da3-78f1.ngrok-free.app`.  Note your actual forwarding URL will be different (but have a similar pattern of alphanumeric characters).  
5. Verify the endpoint is now exposed, either via copying in a browser URL or using `curl -X GET https://31d0-2601-189-8000-c3f0-7c5c-caaa-3da3-78f1.ngrok-free.app/twilio/webhooks/inbound-call`:
   ```xml
   <Response>
  <Start>
    <Stream url="wss://31d0-2601-189-8000-c3f0-7c5c-caaa-3da3-78f1.ngrok-free.app/twilio/websocket/audio-stream"/>
  </Start>
  <Say>This calling is being recorded.  Streaming 60 seconds of audio for live transcription.</Say>
  <Pause length="60" />
</Response>
   ```
   Note this response the stream URL also points to the exposed ngrok endpoint.  This will make sure Twilio sends audio byte streams to the same server.  

## Buy and configure a phone number

In this section we will verify the server is working with a real phone call.  First we need to purchase a phone number from Twilio and then configure it to communicate with the Java server for inbound calls.  For this part of the guide, the Java server needs to be running with ngrok forwarding enabled.  Keep the ngrok forwarding URL ready for copying.  See [Exposing server with ngrok](#exposing-server-with-ngrok) for how to do this.  Note that each time ngrok is turned off, the forwarding URL will be different than last time, so make sure you have copied the most recent one.    

1.  In the Twilio console, navigate to Develop > Phone Numbers > Manager > Buy a number
    ![Buy Twilio Phone Number part 1](buy-twilio-phone-number-part-1.png)
2. (Optional) Search for any particular phone number that has voice capability.  In this case I recommend any cheap phone number in the U.S.
3. Buy the phone number.  This will cost actual money but it is a very small monthly charge (e.g. 25 cents a month) and you can later delete the phone number once finished to avoid future monthly costs.  Some Twilio trial accounts come with free credits so you may not end up paying for anything at all until the credits are used up.    
   ![Buy Twilio Phone Number part 2](buy-twilio-phone-number-part-2.png)
4. After purchasing successfully, open the configuration for the newly purchased phone number
   ![Buy Twilio Phone Number part 3](buy-twilio-phone-number-part-3.png)
5. Under Voice Configuration, replace the URL for the webhook **A call comes in** with the ngrok forwarding URL and the server endpoint path `/twilio/webhooks/inbound-call`.  E.g. it should look like `https://31d0-2601-189-8000-c3f0-7c5c-caaa-3da3-78f1.ngrok-free.app/twilio/webhooks/inbound-call`.  Your particular ngrok forwarding URL will be unique and different from the one in the guide.  
6. Set the HTTP select box method to HTTP GET.
   ![Buy Twilio Phone Number part 4](buy-twilio-phone-number-part-4.png)
7.  Scroll to the bottom of the page and click Save Configuration.
    ![Buy Twilio Phone Number part 5](buy-twilio-phone-number-part-5.png)
8.  Keep a copy of the purchased phone number, we will be calling it regularly to test the live transcription functionality.  

Congratulations!  You've purchased your first phone number and configured it to send webhooks to a Java server running locally.  One gotcha: each time ngrok is stopped and started again, the phone number webhook will need to be reconfigured with the most recent ngrok forwarding URL.  

Now to test the server.  For this next part you'll need a way to dial the purchased phone number.  I use my personal cell phone but any method of dialing it should work.  While dialing be sure to have the Java server console open to verify the incoming web traffic.  

1.  Dial the purchased phone number!  
2. You should hear a message saying "This calling is being recorded.  Streaming 60 seconds of audio for live transcription."
3. Observe the console output of the server:
   ![Console output for Twilio incoming call](console-output-twilio-connection-part-1.png)
   Several key parts are annotated here:
    * the initial inbound call webhook being hit by Twilio
    * the websocket connection being opened
    * the first media stream message being sent over the websocket by Twilio
4.  Afterwards, you should see a continuous stream of messages being received by the websocket.  These messages contain a payload of the captured audio bytes of the phone call.  We will soon send these bytes to Google Cloud Speech To Text to get natural language text back.    
    ![Console output for Twilio incoming call](console-output-twilio-connection-part-2.png)
    * The `"payload"` key of each message looks like gibberish but it's the base 64 representation of the audio bytes.
    * We will soon use speech to text to transcribe each payload into readable human language text.
5. Hang up the call or wait 60 seconds for the call to end.  

Note if instead you hear "An application error has occurred", this means something went wrong with the Twilio webhook.  Double check the server is running, ngrok is forwarding port 8080, and the ngrok forwarding URL on the purchased phone number webhook matches the current ngrok forwarding URL.  For more troubleshooting see: [Troubleshooting Voice Calls](https://www.twilio.com/docs/voice/troubleshooting#an-application-error-has-occurred-on-your-call).  



## Model Twilio Stream messages

In the previous sections we were able to successfully open a websocket audio stream from an incoming call.  However, the current representation of the messages are plain Java `String` objects.  We will need to deserialize these into actual typed objects in order to reference various fields such as the `"payload"`, as well as understand what overall is happening in the stream.  Thankfully Twilio has clear documentation on the schema of each type of Media Stream message, which we'll use to build our Java models: [Twilio Media Stream Messages](https://www.twilio.com/docs/voice/twiml/stream#message-connected).  

1.  Create a new `core.model.twilio.stream` package under the root package.  E.g. the full package path on my implementation would be: `package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream`.  
2. Create two additional subpackages, `messages` for the modeling the stream messages, and `mediaformat` for modeling the audio format of each message.
3.  Create another subpackage under `core.model.twilio.stream.messages` called `payloads`.  This will model information specific to the audio payload.  
4. Create an immutable class `MediaFormatIF` in the `mediaformat` package.  This class will provide information on the audio format of each media message, so we can tell Google Speech To Text how to interpret the audio bytes.
    ```java MediaFormatIF.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.mediaformat;
    
    import com.hubspot.immutables.style.HubSpotStyle;
    import org.immutables.value.Value.Immutable;
    
    @Immutable
    @HubSpotStyle
    public interface MediaFormatIF {
    String getEncoding();
    String getSampleRate();
    String getChannels();
    }
    ```
5.  Create immutable classes to model the nested payloads of incoming messages: `MediaMessagePayloadIF.java`, `StartMessagePayloadIF`, and `StopMessagePayloadIF`.  Create these under the `core.model.twilio.stream.messages.payloads` package:
    ```java StartMessagePayloadIF.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.payloads;
    
    import com.hubspot.immutables.style.HubSpotStyle;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.mediaformat.MediaFormat;
    import java.util.List;
    import java.util.Map;
    import org.immutables.value.Value.Immutable;
    
    @Immutable
    @HubSpotStyle
    public interface StartMessagePayloadIF {
    String getStreamSid();
    String getAccountSid();
    String getCallSid();
    List<String> getTracks();
    Map<String, String> getCustomParameters();
    
    MediaFormat getMediaFormat();
    }
    ```
    ```java MediaMessagePayloadIF.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.payloads;

    import com.hubspot.immutables.style.HubSpotStyle;
    import org.immutables.value.Value.Immutable;
    
    @Immutable
    @HubSpotStyle
    public interface MediaMessagePayloadIF {
    String getTrack();
    String getChunk();
    String getTimestamp();
    String getPayload();
    }
    ```
    ```java StopMesssagePayloadIF.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.payloads;

    import org.immutables.value.Value.Immutable;
    
    import com.hubspot.immutables.style.HubSpotStyle;
    
    @Immutable
    @HubSpotStyle
    public interface StopMessagePayloadIF {
    String getAccountSid();
    String getCallSid();
    }
    ```
6.  Create `MessageEventType.java` enum class to model the four kinds of messages Twilio can send through the audio stream.  Knowing the type of the message will allow for deserializing the message with the right message model.  
    ```java MessageEventType.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages;

    import com.fasterxml.jackson.annotation.JsonCreator;
    import com.fasterxml.jackson.annotation.JsonValue;
    import java.util.Arrays;
    import java.util.Map;
    import java.util.Objects;
    import java.util.function.Function;
    import java.util.stream.Collectors;
    
    public enum MessageEventType {
    CONNECTED("connected"),
    START("start"),
    MEDIA("media"),
    STOP("stop");
    
    private static final Map<String, MessageEventType> EVENT_TO_ENUM_MAP = Arrays
    .stream(MessageEventType.values())
    .collect(
    Collectors.toUnmodifiableMap(MessageEventType::getEventName, Function.identity())
    );
    private final String eventName;
    
    MessageEventType(String eventName) {
    this.eventName = eventName;
    }
    
    @JsonValue
    public String getEventName() {
    return eventName;
    }
    
    @JsonCreator
    public static MessageEventType fromEventName(String eventName) {
    MessageEventType maybeEntry = EVENT_TO_ENUM_MAP.get(eventName);
    if (Objects.isNull(maybeEntry)) {
    throw new IllegalArgumentException(
    String.format("Unknown value for MessageEventType enum: %s", eventName)
    );
    }
    return maybeEntry;
    }
    }
    ```
7.  Create a `StreamMessageCore.java` interface class.  This interface includes fields present in all types of media messages, namely the sequence number and the stream SID (a unique Twilio identifier for the stream).
    ```java StreamMessageCore.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages;

    public interface StreamMessageCore extends StreamMessage {
    String getSequenceNumber();
    String getStreamSid();
    }
    ```
8.  Create models for each of the four media message types: connected, start, message, and stop:
    ```java ConnectedMessageIF
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages;

    import com.hubspot.immutables.style.HubSpotStyle;
    import org.immutables.value.Value.Immutable;
    
    @HubSpotStyle
    @Immutable
    // See: https://www.twilio.com/docs/voice/twiml/stream#message-connected
    public interface ConnectedMessageIF extends StreamMessage {
    String getProtocol();
    String getVersion();
    }
    ```
    ```java StartMessageIF
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages;

    import com.fasterxml.jackson.annotation.JsonAlias;
    import com.hubspot.immutables.style.HubSpotStyle;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.payloads.StartMessagePayload;
    import org.immutables.value.Value.Immutable;

    @HubSpotStyle
    @Immutable
    // See: https://www.twilio.com/docs/voice/twiml/stream#message-start
    public interface StartMessageIF extends StreamMessageCore {
    @JsonAlias("start")
    StartMessagePayload getStartMessagePayLoad();
    }

    ```
    ```java MediaMessageIF
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages;

    import com.fasterxml.jackson.annotation.JsonAlias;
    import com.hubspot.immutables.style.HubSpotStyle;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.payloads.MediaMessagePayload;
    import org.immutables.value.Value.Immutable;
    
    @HubSpotStyle
    @Immutable
    public interface MediaMessageIF extends StreamMessageCore {
    @JsonAlias("media")
    MediaMessagePayload getMediaMessagePayload();
    }

    ```
    ```java StopMessageIF
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages;

    import org.immutables.value.Value.Immutable;
    
    import com.hubspot.immutables.style.HubSpotStyle;
    
    @HubSpotStyle
    @Immutable
    // See: https://www.twilio.com/docs/voice/twiml/stream#example-stop-message
    public interface StopMessageIF extends StreamMessageCore {
    String getCallSid();
    }
    ```
9.  Finally, to allow Java to know how to serialize each stream message into the appropriate message type, introduce a `StreamMessage.java` class that provides JSON subtyping information.
    ```java StreamMessage.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages;

    import com.fasterxml.jackson.annotation.JsonAlias;
    import com.fasterxml.jackson.annotation.JsonSubTypes;
    import com.fasterxml.jackson.annotation.JsonTypeInfo;
    
    @JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.EXISTING_PROPERTY,
    property = "event",
    visible = true
    )
    @JsonSubTypes(
    {
    @JsonSubTypes.Type(value = ConnectedMessage.class, name = "connected"),
    @JsonSubTypes.Type(value = StartMessage.class, name = "start"),
    @JsonSubTypes.Type(value = MediaMessage.class, name = "media"),
    @JsonSubTypes.Type(value = StopMessage.class, name = "stop"),
    }
    )
    public interface StreamMessage {
    @JsonAlias("event")
    MessageEventType getMessageEventType();
    }
    ```
    This uses the media message event type to determine how to deserialize each incoming websocket message.  

Now we need to make the websocket resource class, `TwilioInboundCallWebhookResource` aware of these models so it will automatically deserialize incoming Strings into its proper `StreamMessage` object.  

1.  Create a new top level package `service.twilio.stream` (the full package being `package io.sethmachine.twiliolivetranscriptiondemo.service.twilio.stream;`).  
2. Create the `StreamMessageDecoder.java` decoder class in the new subpackage.  
    ```java StreamMessageDecoder.java
    package io.sethmachine.twiliolivetranscriptiondemo.service.twilio.stream;

    import java.util.Optional;
    
    import javax.websocket.DecodeException;
    import javax.websocket.Decoder;
    import javax.websocket.EndpointConfig;
    
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    
    import com.fasterxml.jackson.databind.ObjectMapper;
    
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.StreamMessage;
    
    public class StreamMessageDecoder implements Decoder.Text<StreamMessage> {
    
    private static final Logger LOG = LoggerFactory.getLogger(StreamMessageDecoder.class);
    
    private ObjectMapper objectMapper;
    
    @Override
    public StreamMessage decode(String s) throws DecodeException {
    return decodeString(s)
    .orElseThrow(() -> {
    String msg = String.format("Failed to parse string into StreamMessage: %s", s);
    return new DecodeException(s, msg);
    });
    }
    
    @Override
    public boolean willDecode(String s) {
    return decodeString(s).isPresent();
    }
    
    @Override
    public void init(EndpointConfig config) {
    this.objectMapper = new ObjectMapper();
    }
    
    @Override
    public void destroy() {}
    
    private Optional<StreamMessage> decodeString(String s) {
    try {
    return Optional.of(objectMapper.readValue(s, StreamMessage.class));
    } catch (Exception e) {
    LOG.error("Failed to decode string into StreamMessage: {}", s);
    return Optional.empty();
    }
    }
    }
    ```
3.  Update the `TwilioInboundCallWebhookResource.java` class to use the new stream decoder, as well as use `StreamMessage streamMessage` instead of `String streamMessage`.
    ```java TwilioInboundCallWebhookResource.java
    package io.sethmachine.twiliolivetranscriptiondemo.resources;

    import com.codahale.metrics.annotation.ExceptionMetered;
    import com.codahale.metrics.annotation.Metered;
    import com.codahale.metrics.annotation.Timed;
    import com.fasterxml.jackson.databind.ObjectMapper;
    
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.StreamMessage;
    import io.sethmachine.twiliolivetranscriptiondemo.guice.GuiceWebsocketConfigurator;
    import io.sethmachine.twiliolivetranscriptiondemo.service.twilio.stream.StreamMessageDecoder;
    
    import java.io.IOException;
    import javax.inject.Inject;
    import javax.websocket.CloseReason;
    import javax.websocket.OnClose;
    import javax.websocket.OnMessage;
    import javax.websocket.OnOpen;
    import javax.websocket.Session;
    import javax.websocket.server.ServerEndpoint;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    
    @Metered
    @Timed
    @ExceptionMetered
    @ServerEndpoint(
    value = "/twilio/websocket/audio-stream",
    configurator = GuiceWebsocketConfigurator.class,
    decoders = { StreamMessageDecoder.class }
    )
    public class TwilioAudioStreamWebsocketResource {
    
    private static final Logger LOG = LoggerFactory.getLogger(
    TwilioAudioStreamWebsocketResource.class
    );
    
    private ObjectMapper objectMapper;
    
    private Session session;
    
    @Inject
    public TwilioAudioStreamWebsocketResource(ObjectMapper objectMapper) {
    this.objectMapper = objectMapper;
    }
    
    @OnOpen
    public void myOnOpen(final Session session) throws IOException {
    LOG.info(
    "[sessionId: {}] Websocket session connection opened: {}",
    session.getId(),
    session
    );
    session.getAsyncRemote().sendText("Ready to receive live transcription results");
    this.session = session;
    }
    
    @OnMessage
    public void myOnMsg(final Session session, StreamMessage streamMessage) {
    LOG.info("[sessionId: {}] Got message: {}", session.getId(), message);
    }
    
    @OnClose
    public void myOnClose(final Session session, CloseReason cr) {
    LOG.info("Closed connection! reason: {}, session: {}", cr, session);
    }
    }
    ```
4.  Update `TwilioLiveTranscriptionDemoApplication.java` to provide the decoder to the websocket endpoint.  Update the `ServerEndpointConfig config` variable so it looks like the following:
    ```java
       final ServerEndpointConfig config = ServerEndpointConfig.Builder
      .create(TwilioAudioStreamWebsocketResource.class, "/twilio/websocket/audio-stream")
      .configurator(new GuiceWebsocketConfigurator(() -> guiceBundle.getInjector()))
      .decoders(ImmutableList.of(StreamMessageDecoder.class))
      .build();
    ```

To verify the media messages are being correctly deserialized, re-run the server and place another phone call to the purchased phone number.  The console output should be equivalent but there should be no serialization errors, confirming the model decoding is working.  We can now work with proper typed models for writing the Speech To Text business logic.  The console output now shows each message as a properly deserialized object:

![Console output with deserialized media messages](console-output-modeling-twilio-messages.png)

## Set up Google Cloud Speech to Text

In this section we will now set up API access to [Google Cloud Speech To Text](https://console.cloud.google.com/marketplace/product/google/speech.googleapis.com).  This will enable the server to access speech to text APIs to turn the audio bytes into natural language text.  The goal here is to create and download JSON API credentials.

### Create credentials

1.  [Create a free Google Cloud account](https://cloud.google.com/free)
2. [In the Google Cloud console, create a new project](https://developers.google.com/workspace/guides/create-project).  I named mine "Twilio Live Transcription"
3. Search for speech to text in the top search bar.  Select Cloud Speech-to-Text API.  
   ![Search for speech to text](setup-google-cloud/search-for-speech-to-text.png)
4. Click Enable to enable the speech to text API
   ![Enable speech to text](setup-google-cloud/enable-speech-to-text-api.png)
    Note you may be prompted to provide billing information to enable this.  If this is your first Google Cloud account, it should come with a large amount of free credits, so you will not be charged until these are exhausted.  
5. Click Create Credentials for the speech to text API to start the process to create a JSON key file.   
   ![Create credentials part 1](setup-google-cloud/create-credentials-part-1.png)
6. Select Application Data in Credential Type setup.
   ![Create credentials part 2](setup-google-cloud/create-credentials-part-2.png)
7. In the Service account details, name the API service account.  I chose "twilio-live-transcription-demo".  Leave the rest of the fields as is.  
8. Grant the Owner role to the service account.  This will let it have access to all available APIs in cloud speech to text.  Hit Done afterwards.  
   ![Create credentials part 3](setup-google-cloud/create-credentials-part-3.png)
9. Under the newly created service account, create a new key.  For key type, select JSON.  
   ![Create credentials part 4](setup-google-cloud/create-credentials-part-4.png)  
10. This will immediately download the JSON key file to your computer.  Importantly, do not share this file or upload it online.  It should be stored outside any repository or codebase.  You should see the key listed in the service account now.  Note in the image below I have replaced the key with a made up string of alphanumeric characters.    
    ![Create credentials part 5](setup-google-cloud/create-credentials-part-5.png)

Congratulations, we have now created JSON credentials for Google Cloud Speech To Text.  As a warning, do not store the JSON file anywhere online or version control it to a repository.  Anyone with the key file can begin using the cloud API and eventually rack up actual charges to your account.  

### Add credentials to server

The server will need the credentials to make Google Cloud API calls.  Follow these steps to do this.  

1.  Move or make a copy of the JSON credentials to folder outside of version control.  For example, I have stored mine here: `/Users/sethmachine/cloud/gcloud/keys/cloud-speech-to-text/12837428383-3024afs.json`.
2. In IntelliJ, open the edit configuration for the `TwilioLiveTranscriptionDemoApplication`.
   ![Edit run config part 1](setup-google-cloud/edit-run-config-part-1.png)
3. Open the Environment variables menu.
   ![Edit run config part 2](setup-google-cloud/edit-run-config-part-2.png)
4. Create a new environment variable called `GOOGLE_APPLICATION_CREDENTIALS`.  Set its value to the full path to where the JSON key file is stored on your computer.   
   ![Edit run config part 3](setup-google-cloud/edit-run-config-part-3.png)
5. Hit Apply and then OK to save this configuration change.  

## Live transcription setup

In this section we will add the actual code to send audio bytes from Twilio's media stream messages to the Google Cloud Speech To Text we set up in the previous section.  Because new audio bytes will be constantly streaming in, we cannot use blocking HTTP requests to wait for transcription results.  Thus we will need to do the following:

* Create a new thread pool that listens for incoming media messages and sends these to Google Cloud Speech To Text
* Send transcription results back to the client through the websocket

### Speech To Text thread pool

The thread pool will run workers that listen for Twilio media stream messages and send these to Google Cloud until transcription results are ready.  The worker will run until the websocket connection is closed.  Thankfully Google has provided an example of how to do this "infinite transcription streaming": [Google Cloud Speech To Text Infinite Stream](https://cloud.google.com/speech-to-text/docs/samples/speech-transcribe-infinite-streaming).  I have taken this example and modified it support the websocket use case as shown below in `StreamingSpeechToTextRunnable.java`.  Be sure to create this under a new package like `core.concurrent.speech.google` for project organization.  

1.  Create new subpackages `core.concurrent.speech.google` and `core.model.speech.google`.  
2.  Create a new class for transcription results output `TranscriptOutputMessageIF.java` under the model subpackage.  These are the messages the websocket will send back to the client or user.  
    ```java TranscriptOutputMessageIF.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.model.speech.google;

    import org.immutables.value.Value.Immutable;
    
    import com.hubspot.immutables.style.HubSpotStyle;
    
    @HubSpotStyle
    @Immutable
    public interface TranscriptOutputMessageIF {
    String getText();
    float getConfidence();
    boolean getIsFinal();
    }

    ```
3.  Create the thread runnable class `StreamingSpeechToTextRunnable.java` under the `core.concurrent.speech.google` package.  This is the code that each worker thread will execute for each incoming phone call audio stream.  It is based on the Google infinite stream example but heavily modified to add in the websocket connection and state.  
    ```java StreamingSpeechToTextRunnable.java
    package io.sethmachine.twiliolivetranscriptiondemo.core.concurrent.speech.google;
    
    import java.io.IOException;
    import java.text.DecimalFormat;
    import java.util.ArrayList;
    import java.util.Base64;
    import java.util.concurrent.BlockingQueue;
    import java.util.concurrent.LinkedBlockingQueue;
    import java.util.concurrent.TimeUnit;
    import java.util.concurrent.atomic.AtomicBoolean;
    
    import javax.websocket.MessageHandler;
    import javax.websocket.Session;
    
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    
    import com.fasterxml.jackson.core.JsonProcessingException;
    import com.fasterxml.jackson.databind.ObjectMapper;
    import com.google.api.gax.rpc.ClientStream;
    import com.google.api.gax.rpc.ResponseObserver;
    import com.google.api.gax.rpc.StreamController;
    import com.google.cloud.speech.v1p1beta1.RecognitionConfig;
    import com.google.cloud.speech.v1p1beta1.RecognitionConfig.AudioEncoding;
    import com.google.cloud.speech.v1p1beta1.SpeechClient;
    import com.google.cloud.speech.v1p1beta1.SpeechRecognitionAlternative;
    import com.google.cloud.speech.v1p1beta1.StreamingRecognitionConfig;
    import com.google.cloud.speech.v1p1beta1.StreamingRecognitionResult;
    import com.google.cloud.speech.v1p1beta1.StreamingRecognizeRequest;
    import com.google.cloud.speech.v1p1beta1.StreamingRecognizeResponse;
    import com.google.inject.Inject;
    import com.google.inject.assistedinject.Assisted;
    import com.google.protobuf.ByteString;
    import com.google.protobuf.Duration;
    
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.speech.google.TranscriptOutputMessage;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.MediaMessage;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.StreamMessage;
    
    public class StreamingSpeechToTextRunnable
      implements Runnable, MessageHandler.Whole<StreamMessage> {
    
      private static final Logger LOG = LoggerFactory.getLogger(
        StreamingSpeechToTextRunnable.class
      );
    
      private static final int STREAMING_LIMIT = 290000; // ~5 minutes
    
      public static final String RED = "\033[0;31m";
      public static final String GREEN = "\033[0;32m";
      public static final String YELLOW = "\033[0;33m";
    
      // Creating shared object
      private static volatile BlockingQueue<byte[]> sharedQueue = new LinkedBlockingQueue();
      private static int BYTES_PER_BUFFER = 6400; // buffer size in bytes
    
      private static int restartCounter = 0;
      private static ArrayList<ByteString> audioInput = new ArrayList<ByteString>();
      private static ArrayList<ByteString> lastAudioInput = new ArrayList<ByteString>();
      private static int resultEndTimeInMS = 0;
      private static int isFinalEndTime = 0;
      private static int finalRequestEndTime = 0;
      private static boolean newStream = true;
      private static double bridgingOffset = 0;
      private static boolean lastTranscriptWasFinal = false;
      private static StreamController referenceToStreamController;
      private static ByteString tempByteString;
    
      private Session websocketSession;
    
      private ObjectMapper objectMapper;
    
      private AtomicBoolean stopped = new AtomicBoolean(false);
      private Thread worker;
    
      @Inject
      public StreamingSpeechToTextRunnable(
        @Assisted Session websocketSession,
        ObjectMapper objectMapper
      ) {
        this.websocketSession = websocketSession;
        this.objectMapper = objectMapper;
      }
    
      public void stop() {
        LOG.info("Received request to stop this thread");
        this.stopped.set(true);
      }
    
      @Override
      public void run() {
        ResponseObserver<StreamingRecognizeResponse> responseObserver = null;
        try (SpeechClient client = SpeechClient.create()) {
          ClientStream<StreamingRecognizeRequest> clientStream;
          responseObserver =
            new ResponseObserver<StreamingRecognizeResponse>() {
              ArrayList<StreamingRecognizeResponse> responses = new ArrayList<>();
    
              public void onStart(StreamController controller) {
                referenceToStreamController = controller;
              }
    
              public void onResponse(StreamingRecognizeResponse response) {
                responses.add(response);
                StreamingRecognitionResult result = response.getResultsList().get(0);
                Duration resultEndTime = result.getResultEndTime();
                resultEndTimeInMS =
                  (int) (
                    (resultEndTime.getSeconds() * 1000) + (resultEndTime.getNanos() / 1000000)
                  );
                double correctedTime =
                  resultEndTimeInMS - bridgingOffset + (STREAMING_LIMIT * restartCounter);
    
                SpeechRecognitionAlternative alternative = result
                  .getAlternativesList()
                  .get(0);
                if (result.getIsFinal()) {
                  isFinalEndTime = resultEndTimeInMS;
                  lastTranscriptWasFinal = true;
                  // in actual use we would publish to a specific channel tied to the call ID
                  websocketSession
                    .getOpenSessions()
                    .forEach(session -> {
                      try {
                        session
                          .getAsyncRemote()
                          .sendObject(
                            objectMapper.writeValueAsString(
                              createTranscriptOutputMessage(result.getIsFinal(), alternative)
                            )
                          );
                      } catch (JsonProcessingException e) {
                        throw new RuntimeException(e);
                      }
                    });
                } else {
                  lastTranscriptWasFinal = false;
                  LOG.info(
                    "TRANSCRIPTION RESULT: transcript: {}, confidence {}",
                    alternative.getTranscript(),
                    alternative.getConfidence()
                  );
    
                  // in actual use we would publish to a specific channel tied to the call ID
                  websocketSession
                    .getOpenSessions()
                    .forEach(session -> {
                      try {
                        session
                          .getAsyncRemote()
                          .sendText(
                            objectMapper.writeValueAsString(
                              createTranscriptOutputMessage(result.getIsFinal(), alternative)
                            )
                          );
                      } catch (JsonProcessingException e) {
                        throw new RuntimeException(e);
                      }
                    });
                }
              }
    
              public void onComplete() {}
    
              public void onError(Throwable t) {}
            };
          clientStream = client.streamingRecognizeCallable().splitCall(responseObserver);
    
          RecognitionConfig recognitionConfig = RecognitionConfig
            .newBuilder()
            .setEncoding(AudioEncoding.MULAW)
            .setLanguageCode("en-US")
            .setSampleRateHertz(8000)
            .setModel("phone_call")
            .build();
    
          StreamingRecognitionConfig streamingRecognitionConfig = StreamingRecognitionConfig
            .newBuilder()
            .setConfig(recognitionConfig)
            .setInterimResults(true)
            .build();
    
          StreamingRecognizeRequest request = StreamingRecognizeRequest
            .newBuilder()
            .setStreamingConfig(streamingRecognitionConfig)
            .build(); // The first request in a streaming call has to be a config
    
          clientStream.send(request);
    
          try {
            long startTime = System.currentTimeMillis();
    
            while (!stopped.get()) {
              long estimatedTime = System.currentTimeMillis() - startTime;
    
              if (estimatedTime >= STREAMING_LIMIT) {
                clientStream.closeSend();
                referenceToStreamController.cancel(); // remove Observer
    
                if (resultEndTimeInMS > 0) {
                  finalRequestEndTime = isFinalEndTime;
                }
                resultEndTimeInMS = 0;
    
                lastAudioInput = null;
                lastAudioInput = audioInput;
                audioInput = new ArrayList<ByteString>();
    
                restartCounter++;
    
                if (!lastTranscriptWasFinal) {
                  System.out.print('\n');
                }
    
                newStream = true;
    
                clientStream =
                  client.streamingRecognizeCallable().splitCall(responseObserver);
    
                request =
                  StreamingRecognizeRequest
                    .newBuilder()
                    .setStreamingConfig(streamingRecognitionConfig)
                    .build();
    
                System.out.println(YELLOW);
                System.out.printf(
                  "%d: RESTARTING REQUEST\n",
                  restartCounter * STREAMING_LIMIT
                );
    
                startTime = System.currentTimeMillis();
              } else {
                if ((newStream) && (lastAudioInput.size() > 0)) {
                  // if this is the first audio from a new request
                  // calculate amount of unfinalized audio from last request
                  // resend the audio to the speech client before incoming audio
                  double chunkTime = STREAMING_LIMIT / lastAudioInput.size();
                  // ms length of each chunk in previous request audio arrayList
                  if (chunkTime != 0) {
                    if (bridgingOffset < 0) {
                      // bridging Offset accounts for time of resent audio
                      // calculated from last request
                      bridgingOffset = 0;
                    }
                    if (bridgingOffset > finalRequestEndTime) {
                      bridgingOffset = finalRequestEndTime;
                    }
                    int chunksFromMs = (int) Math.floor(
                      (finalRequestEndTime - bridgingOffset) / chunkTime
                    );
                    // chunks from MS is number of chunks to resend
                    bridgingOffset =
                      (int) Math.floor((lastAudioInput.size() - chunksFromMs) * chunkTime);
                    // set bridging offset for next request
                    for (int i = chunksFromMs; i < lastAudioInput.size(); i++) {
                      request =
                        StreamingRecognizeRequest
                          .newBuilder()
                          .setAudioContent(lastAudioInput.get(i))
                          .build();
                      clientStream.send(request);
                    }
                  }
                  newStream = false;
                }
    
                tempByteString = ByteString.copyFrom(sharedQueue.take());
    
                request =
                  StreamingRecognizeRequest
                    .newBuilder()
                    .setAudioContent(tempByteString)
                    .build();
    
                audioInput.add(tempByteString);
              }
    
              clientStream.send(request);
            }
          } catch (Exception e) {
            System.out.println(e);
          }
        } catch (IOException e) {
          throw new RuntimeException(e);
        }
        LOG.info("Runnable has stopped!");
      }
    
      @Override
      public void onMessage(StreamMessage streamMessage) {
        MediaMessage mediaMessage = (MediaMessage) streamMessage;
        byte[] audioBytes = Base64
          .getDecoder()
          .decode(mediaMessage.getMediaMessagePayload().getPayload());
        try {
          sharedQueue.put(audioBytes);
        } catch (InterruptedException e) {
          LOG.error("Failed to add media message bytes to shared queue", e);
          throw new RuntimeException(e);
        }
      }
    
      public static String convertMillisToDate(double milliSeconds) {
        long millis = (long) milliSeconds;
        DecimalFormat format = new DecimalFormat();
        format.setMinimumIntegerDigits(2);
        return String.format(
          "%s:%s /",
          format.format(TimeUnit.MILLISECONDS.toMinutes(millis)),
          format.format(
            TimeUnit.MILLISECONDS.toSeconds(millis) -
            TimeUnit.MINUTES.toSeconds(TimeUnit.MILLISECONDS.toMinutes(millis))
          )
        );
      }
    
      private static TranscriptOutputMessage createTranscriptOutputMessage(
        boolean isFinal,
        SpeechRecognitionAlternative alternative
      ) {
        return TranscriptOutputMessage
          .builder()
          .setText(alternative.getTranscript().strip())
          .setConfidence(alternative.getConfidence())
          .setIsFinal(isFinal)
          .build();
      }
    }
    ```
4.  Create a factory class `StreamingSpeechToTextRunnableFactory.java` to allow dynamically creating each runnable with different websocket connection each time. 
    ```java
    package io.sethmachine.twiliolivetranscriptiondemo.core.concurrent.speech.google;

    import javax.websocket.Session;
    
    public interface StreamingSpeechToTextRunnableFactory {
    StreamingSpeechToTextRunnable create(Session websocketSession);
    }
    ```
5.  Modify the `TwilioLiveTranscriptionDemoModule.java` (under the `guice` package) to provide a thread pool executor for the server and register the factory class.  
    ```java TwilioLiveTranscriptionDemoModule.java
    package io.sethmachine.twiliolivetranscriptiondemo.guice;
    
    import com.fasterxml.jackson.databind.ObjectMapper;
    import com.google.inject.Provides;
    import com.google.inject.Singleton;
    import com.google.inject.assistedinject.FactoryModuleBuilder;
    import com.google.inject.name.Named;
    import io.dropwizard.Configuration;
    import io.sethmachine.twiliolivetranscriptiondemo.core.concurrent.speech.google.StreamingSpeechToTextRunnableFactory;
    import java.util.concurrent.LinkedBlockingQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    import ru.vyarus.dropwizard.guice.module.support.DropwizardAwareModule;
    
    public class TwilioLiveTranscriptionDemoModule extends DropwizardAwareModule<Configuration> {
    
    @Override
    protected void configure() {
    install(new FactoryModuleBuilder().build(StreamingSpeechToTextRunnableFactory.class));
    
        configuration();
        environment();
        bootstrap();
    }
    
    @Provides
    @Singleton
    @Named("StreamingCloudSpeechToTextThreadPoolExecutor")
    public ThreadPoolExecutor provideThreadPoolExecutorForCloudSpeechToText() {
    return new ThreadPoolExecutor(
    8,
    100,
    60,
    TimeUnit.SECONDS,
    new LinkedBlockingQueue()
    );
    }
    
    @Provides
    @Singleton
    public ObjectMapper provideObjectMapper() {
    return bootstrap().getObjectMapper();
    }
    }
    ```

### Websocket Speech to Text service

In this section we will create a class to manage how the thread pool is started or how existing workers are stopped (e.g. the phone call ends).  

1.  Create a new service class `StreamingSpeechToTextService.java` under subpackage `service.speech.google`.  
    ```java StreamingSpeechToTextService.java
    package io.sethmachine.twiliolivetranscriptiondemo.service.speech.google;

    import java.util.Optional;
    import java.util.concurrent.ThreadPoolExecutor;
    
    import javax.websocket.Session;
    
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    
    import com.google.common.collect.Iterables;
    import com.google.inject.Inject;
    import com.google.inject.name.Named;
    
    import io.sethmachine.twiliolivetranscriptiondemo.core.concurrent.speech.google.StreamingSpeechToTextRunnable;
    import io.sethmachine.twiliolivetranscriptiondemo.core.concurrent.speech.google.StreamingSpeechToTextRunnableFactory;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.ConnectedMessage;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.MediaMessage;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.StartMessage;
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.StreamMessage;
    
    public class StreamingSpeechToTextService {
    
    private static final Logger LOG = LoggerFactory.getLogger(
    StreamingSpeechToTextService.class
    );
    
    private final ThreadPoolExecutor speechToTextThreadPoolExecutor;
    private final StreamingSpeechToTextRunnableFactory streamingSpeechToTextRunnableFactory;
    
    @Inject
    public StreamingSpeechToTextService(
    @Named(
    "StreamingCloudSpeechToTextThreadPoolExecutor"
    ) ThreadPoolExecutor threadPoolExecutor,
    StreamingSpeechToTextRunnableFactory streamingSpeechToTextRunnableFactory
    ) {
    this.speechToTextThreadPoolExecutor = threadPoolExecutor;
    this.streamingSpeechToTextRunnableFactory = streamingSpeechToTextRunnableFactory;
    }
    
    public void handleStreamMessage(Session session, StreamMessage streamMessage) {
    switch (streamMessage.getMessageEventType()) {
    case CONNECTED:
    handleConnectedMessage(session, (ConnectedMessage) streamMessage);
    break;
    case START:
    handleStartMessage(session, (StartMessage) streamMessage);
    break;
    case MEDIA:
    handleMediaMessage(session, (MediaMessage) streamMessage);
    break;
    case STOP:
    handleStreamClose(session);
    break;
    default:
    LOG.error(
    "[sessionId: {}] Unhandled message event type for StreamMessage: {}",
    session.getId(),
    streamMessage
    );
    }
    }
    
    public void handleStreamClose(Session session) {
    getRunnableFromSession(session)
    .ifPresentOrElse(
    StreamingSpeechToTextRunnable::stop,
    () -> LOG.info("Attempted to stop session but no runnable found: {}", session)
    );
    }
    
    private void handleConnectedMessage(
    Session session,
    ConnectedMessage connectedMessage
    ) {
    LOG.info(
    "[sessionId: {}] Received connected message: {}",
    session.getId(),
    connectedMessage
    );
    }
    
    private void handleStartMessage(Session session, StartMessage startMessage) {
    LOG.info("[sessionId: {}] Received start message: {}", session.getId(), startMessage);
    StreamingSpeechToTextRunnable streamingSpeechToTextRunnable = streamingSpeechToTextRunnableFactory.create(
    session
    );
    session.addMessageHandler(streamingSpeechToTextRunnable);
    speechToTextThreadPoolExecutor.execute(streamingSpeechToTextRunnable);
    }
    
    private void handleMediaMessage(Session session, MediaMessage mediaMessage) {
    StreamingSpeechToTextRunnable streamingSpeechToTextRunnable = getRunnableFromSession(
    session
    )
    .orElseThrow();
    streamingSpeechToTextRunnable.onMessage(mediaMessage);
    }
    
    private Optional<StreamingSpeechToTextRunnable> getRunnableFromSession(
    Session session
    ) {
    try {
    return Optional.of(
    (StreamingSpeechToTextRunnable) Iterables.getOnlyElement(
    session.getMessageHandlers()
    )
    );
    } catch (Exception e) {
    LOG.error("Failed to get runnable from session: {}", session, e);
    return Optional.empty();
    }
    }
    }
    ```
2.  Modify the existing `TwilioAudioStreamWebsocketResource.java` websocket resource to use the service class.  In particular, we want to start the worker when a new websocket connection is made `StreamingSpeechToTextService#handleStreamMessage` and stop an existing worker when a phone call ends via `StreamingSpeechToTextService#handleStreamClose`.  
    ```java TwilioAudioStreamWebsocketResource.java
    package io.sethmachine.twiliolivetranscriptiondemo.resources;

    import java.io.IOException;
    
    import javax.inject.Inject;
    import javax.websocket.CloseReason;
    import javax.websocket.OnClose;
    import javax.websocket.OnMessage;
    import javax.websocket.OnOpen;
    import javax.websocket.Session;
    import javax.websocket.server.ServerEndpoint;
    
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    
    import com.codahale.metrics.annotation.ExceptionMetered;
    import com.codahale.metrics.annotation.Metered;
    import com.codahale.metrics.annotation.Timed;
    import com.fasterxml.jackson.databind.ObjectMapper;
    
    import io.sethmachine.twiliolivetranscriptiondemo.core.model.twilio.stream.messages.StreamMessage;
    import io.sethmachine.twiliolivetranscriptiondemo.guice.GuiceWebsocketConfigurator;
    import io.sethmachine.twiliolivetranscriptiondemo.service.speech.google.StreamingSpeechToTextService;
    import io.sethmachine.twiliolivetranscriptiondemo.service.twilio.stream.StreamMessageDecoder;
    
    @Metered
    @Timed
    @ExceptionMetered
    @ServerEndpoint(
    value = "/twilio/websocket/audio-stream",
    configurator = GuiceWebsocketConfigurator.class,
    decoders = { StreamMessageDecoder.class }
    )
    public class TwilioAudioStreamWebsocketResource {
    
    private static final Logger LOG = LoggerFactory.getLogger(
    TwilioAudioStreamWebsocketResource.class
    );
    
    private StreamingSpeechToTextService streamingSpeechToTextService;
    private ObjectMapper objectMapper;
    
    private Session session;
    
    @Inject
    public TwilioAudioStreamWebsocketResource(
    StreamingSpeechToTextService streamingSpeechToTextService,
    ObjectMapper objectMapper
    ) {
    this.streamingSpeechToTextService = streamingSpeechToTextService;
    this.objectMapper = objectMapper;
    }
    
    @OnOpen
    public void myOnOpen(final Session session) throws IOException {
    LOG.info(
    "[sessionId: {}] Websocket session connection opened: {}",
    session.getId(),
    session
    );
    session.getAsyncRemote().sendText("Ready to receive live transcription results");
    this.session = session;
    }
    
    @OnMessage
    public void myOnMsg(final Session session, StreamMessage streamMessage) {
    streamingSpeechToTextService.handleStreamMessage(session, streamMessage);
    }
    
    @OnClose
    public void myOnClose(final Session session, CloseReason cr) {
    LOG.info("Closed connection! reason: {}, session: {}", cr, session);
    streamingSpeechToTextService.handleStreamClose(session);
    }
    }
    ```



## Conclusion