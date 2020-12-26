---
title: "Using HTTP Content-Range headers to resume file uploads"
tags: [Http]
---

Resumable file transfers are sometimes necessary in situations where a large amount of data must be sent over a network that experiences intermittent outages. This could be the case when a remote worker at a rural job site has tethered their laptop via a weak cellular signal and needs to upload a large file package back to home office. In many cases using an FTP server may not be an option due to corporate policy or perhaps you are building a service that must perform addiational processing at the ment that a file is completely transfered.

Utilizing content-range support of the HTTP protocol can solve the problem. A content-range header is a standard HTTP header that informs the receiver of our intention to provide only a part of a larger set of bytes. [RFC 7233](https://tools.ietf.org/html/rfc7233) contains the detail of how this header is expected to behave.


Content ranged transfers allow the client application to upload a large file in smaller chunks. To do so we need to ensure that the file that we are uploading will not be modified during the time that the transfer is occuring, this makes sense as we are effictively creating a promise to the receiver that we are uploading pieces of a whole, which when assembled on the receiving side can be pieced back together.

The client can also elect to send file chunks of anysize, this may be useful as a tuning parameter for the system, balancing between overall speed of transfer and resiliance against timeouts or network interruption.


This sequence diagram outlines the process:

![Http Resumable Sequence Diagram](/assets/images/2017/03/21/System%20Sequence%20Diagram.png)

Metadata is sent from the client to the server in the initial upload header, here we have chosen to send a content-length of 524K out of our intended 25MB file. The body of this message contains the actual bits of the first part of this file. 

``` HTTP
PUT https://server.com/api/upload?file=abc.zip HTTP/1.1
Content-Length: 524288
Content-Type: application/zip
Content-Range: bytes 0-524287/25000000

[BYTES 0-524287]
```

If the upload is successful then the server will respond back with a 308 Resume incomplete message. The response indicates from the server how much of the content was received at the given offset, in this case the file was uploaded beginning at byte marker 0, and completed with 524288 bytes total.

```HTTP
HTTP/1.1 308 Resume Incomplete
Content-Length: 0
Range: bytes=0-524287
```

We take the upper byte range returned from the server and use this to paginate into our file by the byte offset -> 524287. Rinse and repeat the process, moving through the file in chunks until the server respond back with a 200 OK. Note that we need to be careful to calculate the content-length correctly for our PUT messages, this should always reflect the actual number of bytes placed in the message body, and will need to be adjusted as we put the last chunk for upload to the server.

If the upload is interrupted we can ask the server to provide a status to us before resuming the upload. This message is similar to the one used when we began the transfer but importantly it has a content-length == 0, as we do not know which chunk to send yet. And the content range header uses an asterix '*' to indicate that the content range is unknown, however the total file size is sent, the server should take this opportunity to validate that the file size matches what it expected given the earliest part of this conversation with this particular client.  

```HTTP
PUT https://server.com/api/upload?file=abc.zip HTTP/1.1
Content-Length: 0
Content-Range: bytes */25000000
```

The expected response back from the inquiry message is a 308 Resume that denotes the size of the message thus far received by the server.

```HTTP
HTTP/1.1 308 Resume Incomplete
Content-Length: 0
Range: bytes=xxxxx-yyyyy
```