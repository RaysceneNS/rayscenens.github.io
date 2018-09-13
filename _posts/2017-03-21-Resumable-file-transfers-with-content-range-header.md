---
layout: post
title: Resumable file transfers.
excerpt: "Resumable file transfers are sometimes necessary in situations where a large amount of data must be sent over a network that experiences intermittent outages."
tags: [Http]
---

# Http Content-Range header for resuming file uploads

Resumable file transfers are sometimes necessary in situations where a large amount of data must be sent over a network that experiences intermittent outages. This could be the case when a remote worker at a rural job site has tethered their laptop via a weak cellular signal and needs to upload a large file package home. In many cases using an FTP server may not be an option due to corporate restrictions or maybe the service that is communicated to has a need to perform further processing on the files as they come in.

Utilizing content-range support of the HTTP protocol can solve the problem. A content-range header is a standard HTTP header that informs the receiver of our intention to provide only a part of a larger set of bytes. See [RFC 7233](https://tools.ietf.org/html/rfc7233) for more detail on this standard.

```HTTP
PUT https://server.com/api/upload?file=abc.zip HTTP/1.1
Content-Length: 524288
Content-Type: application/zip
Content-Range: bytes 0-524287/25000000
[BYTES 0-524287]
```

If the upload is successful then the server will respond back with a 308 Resume incomplete message.

```HTTP
HTTP/1.1 308 Resume Incomplete
Content-Length: 0
Range: bytes=0-524287
```

In this case we take the upper byte range returned from the server and use this to paginate into our file by the byte offset -> 524287. Rinse and repeat the process, moving through the file in chunks until the server respond back with a 200 OK. Note that we need to be careful to calculate the content-length correctly for our PUT messages, this should always reflect the actual number of bytes placed in the message body, and will need to be adjusted as we put the last chunk for upload to the server.

If the upload is interrupted we can ask the server to provide a status to us before resuming the upload.

```HTTP
PUT https://server.com/api/upload?file=abc.zip HTTP/1.1
Content-Length: 0
Content-Range: bytes */2000000
```

The expected response back from the inquiry message is a 308 Resume that denotes the size of the message thus far received by the server.
![Http Resumable Sequence Diagram](/assets/images/2017/03/21/System%20Sequence%20Diagram.png)
