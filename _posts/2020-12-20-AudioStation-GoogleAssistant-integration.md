---
tags: ["Google Assistant", AudioStation, Synology]
---

# How-to use google assistant to call AudioStation

## Problem

Like many folks who own a Synology NAS I watched with envy as folks in the Alexa universe received love from the development team at Synology to command and control playback of the media that they had placed into the AudioStation app, while those of us within the Googleverse got nada.

Time to take matters into my own hands.

## Investigate Alexa Skill

There was very little information available from Synology as to how the Alexa skill integration with AudioStation worked, so I had to do a little digging. I knew that there were essentially two pieces in play here. First the local server would need to participate in an OAuth token exchange to allow the external commands from the Amazon cloud to control the api services running locally on the NAS.

Second I would need to figure out the format of the API that was used to command and control AudioStation when using the voice commands.


### Synology OAuth Service

Looking at the applications available within the NAS I can see there is application labeled 'OAuth Service', hmm interesting.

To login to synology via SSH you'll need to enable ssh in the control panel of DSM under the terminal setting page.
note: Set a custom port number for ssh now or else the synology security adviser will nag you endlessly.


To Login, open a command terminal and use your OS' builtin ssh command to establish a session on the Synology machine. `ssh admin@ipaddress -p 22`


In order to elevate privelege from admin to root on synology dsm enter the following command. When prompted for a password enter the admin account passwor again. `sudo -i`



We will need to register a client entry in the server to allow google assistant to request an OAuth token exchange.

Add an oauth app entry for our purposes, credits to Dominik for this blog post https://medium.com/@4c.dmnk/take-control-over-synologys-oauth-service-f96114be3707 where he digs into the Oauth services. 

There are several system utilities designed to modify the records contained within the synology OAuth client registration database in the folder `/usr/local/packages/@appstore/OAuthService/tools`.





### AudioStation Voice Assistant API

Synology has scarce development documentation available at the [developer site](https://www.synology.com/en-global/support/developer)

Due to the lack of documentation most of the commands will need to be inferred by watching the traffic sent between Alexa and DSM as the various functions  are exercised.

There is a document that details the voice command structure for Alexa on this page [How to enable Audio Station skill on Amazon Alexa](https://www.synology.com/en-global/knowledgebase/DSM/tutorial/Multimedia/How_to_enable_Audio_Station_skill_on_Amazon_Alexa)

essentially all english commands begin with the phrase "Alexa, ask AudioStation ..."
```
to	[play/start/search]
            [song name]
            the	[music/track/songs/audio] by [artist]
            the album [album name]
            the playlist [playlist name]
what's playing
what song is the song
```

Most of the verbs around Alexa integration appear to use the prefix SYNO.AudioStation.VoiceAssistant

I was able to simulate an Alexa client using the excellent [echosim tool](https://echosim.io/welcome)

I then setup nginx on the synology to log all relevant details of the HTTP traffic to a log file while I asked Alexa to do various functions, this [post by Rob D.](https://community.synology.com/enu/forum/14/post/124369) was helpful in setting up nginx to perform the logging.


All functions encode inputs using application/x-www-form-urlencoded, results are returned in json format. 
All POST functions expect a bearer token to be supplied, GET functions use the query string param '_oat' to pass tokens. 

Voice Assistant API methods:

count_search; Used to return the number of results for a query, any attribute in the track maybe queried [album/artist/title/etc...]
example:
```HTTP
POST /webapi/entry.cgi/SYNO.AudioStation.VoiceAssistant.Browse
title=%22songname%22&api=SYNO.AudioStation.VoiceAssistant.Browse&method=count_search&version=1

{
    "data": {
        "count": 9
    },
    "success": true
}
```

search; Used to return the details of tracks that match a query attribute [album/artist/title/etc...]
```HTTP
POST /webapi/entry.cgi/SYNO.AudioStation.VoiceAssistant.Browse
offset=0&limit=10&title=%22songname%22&sort_by=%22album%22&api=SYNO.AudioStation.VoiceAssistant.Browse&method=search&version=1

{
    "data": {
        "track": [
            {
                "album": "Song Album XXX",
                "artist": "Artist YYY",
                "codec": "mp3",
                "file_extension": "mp3",
                "id": 1234,
                "title": "Title ZZZ"
            }
        ]
    }
    "success": true
}


stream; Returns the audio stream to play, the optional parameter ['method=transcode'] may be added to transcode non mp3 files on the fly.
```HTTP
GET /webapi/entry.cgi/SYNO.AudioStation.VoiceAssistant.Stream?api=SYNO.AudioStation.VoiceAssistant.Stream&method=stream&version=1&track_id=1234&_oat=%22_bearer_token_here_%22

[..binary file stream response...]
```



# Create Google Action project

In order for commands received by your google assistant to do anything you'll need to setup a google actions project. 

Goto https://console.actions.google.com/ 

Creating a new project for english language (or your language of choice), gave it a unique project name.

Gave the action the name of Audio Station, as it had to be two words.

The goolge action consists of two major parts that need to be setup, first there is the action that defines how spoken phrases are mapped to actions. The template for this is available at [my Github](https://github.com/RaysceneNS/AudioStation-GoogleAssistant/Action/action.json) You will need to replace the server url and oauth linking information.

Secondly, the fulfillment project contains the Google functions that are executed by the action that we setup. This is where the actual code lives that communicates with the API that we saw earlier. Requests such as play song invoke a web search method and then return the first matching track as an Media stream.




## Register OAuth client information

Now we move back to the OAuth service on the Synology NAS, this time we will add the actual database entry that allows google assistant to receive an Authorization code using auth code flow.

### Arguments

uri: Also we supply a redirect url to call, this again must match the location that we setup earlier. 
scope: Note here that we are asking for the same scope as the Alexa assistant uses, this is important as it limits what functions are available to be called.
displayname: Any name you want here is fine.


Use the following command to add a client to the OAuth list. Make note of the client id and secret returned from this function as we will require these later on in order to complete the service registration with Google. 

```
# ./oauth_clientinfo --client-add oauth-redirect.googleusercontent.com/r/[some-project-name] AudioStation.voiceassistant GoogleAssistant 
```

Now you should see something like the following in your OAuth service blade.
![Alternate Product](/assets/images/2020/12/20/oauth_setup.png)

### Linking the google action:

In the account linking form enter the following information:

Linking Type - Select Oauth & Authorization Code

OAuth Client Information - 

Client ID: This is the client id returned from the OAuth registration step
Client Secret: This is the client secreet returned from the OAuth registration step.


Authorization URL: Enter publicly available url of your DSM (you must have a valid SSL Certificate installed on DSM) https://[my.synology.com]:5001/oauth/oauth_login.cgi

Token URL: https://[my.synology.com]:5001/oauth/oauth_token.cgi

# Testing

Once the action and fulfillment are setup, you can begin a conversation with Audio Station using Google. Utterances must explicitly include 'Ask AudioStation' or else Google will take this to mean use the default music provider to satisfy the request. Try 'Ask Audio Station to play song ...'

