---
title: "Investigate Audio Station API"
tags: ["Google Assistant", AudioStation, Synology]
---

# Synology Voice Assistant API for Audio Station

There was very little information available from Synology as to how the Alexa skill integration with AudioStation worked, so I had to do a little digging. I knew that there were essentially two pieces in play here. First the local server would need to participate in an OAuth token exchange to allow the external commands from the Amazon to control the API services running locally on the NAS.

Second I would need to figure out the format of the messages that were transferred by API that was used to command and control AudioStation when using the voice commands from Alexa.

## Synology OAuth Service

Looking at the applications available within the NAS I can see there is application labeled 'OAuth Service', hmm interesting.

To login to Synology via SSH you'll need to enable SSH in the control panel of DSM under the terminal setting page.
note: Set a custom port number for SSH now or else the Synology security adviser will nag you endlessly.

To Login, open a command terminal and use your OS' built-in SSH command to establish a session on the Synology machine. `ssh admin@ipaddress -p 22`

In order to elevate privilege from admin to root on Synology DSM enter the following command. When prompted for a password enter the admin account password again. `sudo -i`

We will need to register a client entry in the server to allow google assistant to request an OAuth token exchange.

Add an OAuth app entry for our purposes, credits to Dominik for this [blog post](https://medium.com/@4c.dmnk/take-control-over-synologys-oauth-service-f96114be3707) where he digs into the OAuth services.

There are several system utilities designed to modify the records contained within the Synology OAuth client registration database in the folder `/usr/local/packages/@appstore/OAuthService/tools`.

## AudioStation Voice Assistant API

Synology has scarce development documentation available at the [developer site](https://www.synology.com/en-global/support/developer)

Due to the lack of documentation most of the commands will need to be inferred by watching the traffic sent between Alexa and DSM as the various functions  are exercised.

There is a document that details the voice command structure for Alexa on this page [How to enable Audio Station skill on Amazon Alexa](https://www.synology.com/en-global/knowledgebase/DSM/tutorial/Multimedia/How_to_enable_Audio_Station_skill_on_Amazon_Alexa)

essentially all English commands begin with the phrase "Alexa, ask AudioStation ..."

```
to [play/start/search]
            [song name]
            the [music/track/songs/audio] by [artist]
            the album [album name]
            the playlist [playlist name]
what's playing
what song is the song
```

Most of the verbs around Alexa integration appear to use the prefix SYNO.AudioStation.VoiceAssistant

I don't actually own an Alexa device, however I was able to simulate an Alexa client using the excellent [echosim tool](https://echosim.io/welcome).

I then setup NGINX on the Synology to log all relevant details of the HTTP traffic to a log file while I asked Alexa to do various functions, this [post by Rob D.](https://community.synology.com/enu/forum/14/post/124369) was helpful in setting up NGINX to perform the logging.

### Recipe for logging requests to NGINX

Based on that post I performed the following actions:

- Edit NGINX mustache file to allow access, Comment out the "access_log off" line and uncomment the line below it
`vi /usr/syno/share/nginx/nginx.mustache`
Added the request_body / accept,authorization,content-type headers to the log:

```bash
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent $request_body "$http_referer"'
        '"$http_user_agent" "$http_x_forwarded_for" "$http_authorization" "$content_type" "$http_accept"';
```

- Fix the permissions of the /var/log/nginx directory by editing the NGINX startup file:
` sudo vi /usr/share/init/nginx.conf `
Look for the line that starts with "MakeDirectory /var/log/nginx" and change the access permission to 0755

- Fix file permissions for the access log and error log (add global user read/write) in the syslog-ng nginx config.  (NGINX is configured to send its logs through syslog-ng.)
` sudo vi /etc.defaults/syslog-ng/patterndb.d/nginx.conf `
Find the string 'file("/var/log/nginx/access.log"' and add the string "perm(0666)" immediately after.  Do the same with the error.log entry.  (Or you can use 0644 to only allow read permissions.) Full line should look like this:

- Restart NGINX, syslog-ng
`sudo synoservice --restart nginx`
`sudo synoservice --restart syslog-ng`

Logs are located in the directory `/var/log/nginx/access.log`

note: log files can be truncated using `truncate -s 0 access.log`

All functions encode inputs using application/x-www-form-urlencoded, results are returned in JSON encoded format.
All POST functions expect a bearer token to be supplied.
GET functions use the query string parameter '_oat' to pass authentication tokens.

## Voice Assistant API methods

### count_search()

 Used to return the number of results for a query, any attribute in the track maybe queried [album/artist/title/etc...]

```HTTP
POST /webapi/entry.cgi/SYNO.AudioStation.VoiceAssistant.Browse HTTP/1.1
Host: server
Content-Type: application/x-www-form-urlencoded
Authentication: Bearer 123ABCDEF...
Accept: application/json

title=%22songname%22&api=SYNO.AudioStation.VoiceAssistant.Browse&method=count_search&version=1 
```

```HTTP
Server: nginx
Date: _now_
Content-Type: application/json

{
    "data": {
        "count": 9
    },
    "success": true
}
```

### search()

 Used to return the details of tracks that match a query attribute [album/artist/title/etc...]

```HTTP
POST /webapi/entry.cgi/SYNO.AudioStation.VoiceAssistant.Browse HTTP/1.1
Host: server
Content-Type: application/x-www-form-urlencoded
Authentication: Bearer 123ABCDEF...
Accept: application/json

offset=0&limit=10&title=%22songname%22&sort_by=%22album%22&api=SYNO.AudioStation.VoiceAssistant.Browse&method=search&version=1
```

```HTTP
Server: nginx
Date: _now_
Content-type: application/json

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
```

### stream()

 Returns the audio stream to play, the optional parameter ['method=transcode'] may be added to transcode non mp3 files on the fly.

```HTTP
GET /webapi/entry.cgi/SYNO.AudioStation.VoiceAssistant.Stream?api=SYNO.AudioStation.VoiceAssistant.Stream&method=stream&version=1&track_id=1234&_oat=%22_bearer_token_here_%22 HTTP/1.1
```

```HTTP
Server: nginx
Date: _now_
Content-type: application/binary

.. binary file stream response body ...
```

Hope that helps.