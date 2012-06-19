# Linked Servers Distributing AJAX events from multiple sources

A presentation to Not Just Ruby by the "Coot" Craig Anderson.

## Project Overview

The call center I work at wanted to distribute call progress events to
various pieces of software on computers throughout the company intranet.
There is both inbound and outbound work.

A windows DLL is available to get inbound call events from the phone switch.

No centralized source was available for outbound calls, but messages
are sent to the workstations of outbound agents logged into the dialer.

The shop infracture is Windows OS, MSSQL and Firefox for the browser.

I'm a happy Ruby programmer and management was fine with that so a
developing I went.

![System Diagram](https://github.com/CootCraig/ajax_sinatra_20120618/blob/master/system_diagram.png)

<img src="https://github.com/CootCraig/ajax_sinatra_20120618/blob/master/system_diagram.png" alt="System Diagram"></img>

## The Plan such as it was

1. I've used Sinatra as an HTTP application before, so that's decided.
1. Distributing events sounds like AJAX. I've never worked with AJAX.  Does it work with Sinatra.  I'll find out.
1. There's this DLL for getting events from the phone switch; sure hope I can get this to work.

## Access the Phone Switch DLL from Ruby
There's a gem for that: Here's one that will connect DLLs to Ruby

```ruby
require 'ffi'
```

cvlancli.dll

## Picking the tools

## Collecting the Inbound events

## Omq Overview

[The Intelligent Transport Layer](http://www.zeromq.org/)

[The excellent guide for 0mQ](http://zguide.zeromq.org/page:all)

### From the Guide - ØMQ in a Hundred Words

ØMQ (ZeroMQ, 0MQ, zmq) looks like an embeddable networking library but
acts like a concurrency framework. It gives you sockets that carry whole
messages across various transports like in-process, inter-process, TCP,
and multicast. You can connect sockets N-to-N with patterns like fanout,
pub-sub, task distribution, and request-reply. It's fast enough to be
the fabric for clustered products. Its asynchronous I/O model gives you
scalable multicore applications, built as asynchronous message-processing
tasks. It has a score of language APIs and runs on most operating
systems. ØMQ is from iMatix and is LGPL open source.

I heard of 0MQ because DCell, the distributed extension to Celluloid
uses 0MQ for remote messaging.

2 types of 0mq sockets are used:

## VB.net 0mq Event Injector

I wrote a VB.net console application to send arbitrary events
to a 0mq

```
    Sub Main(ByVal args As String())
            Dim zmq_context As New ZMQ.Context
            Dim zsock As ZMQ.Socket = Nothing
            Dim msg As New Dictionary(Of String, String)

            zsock = zmq_context.Socket(SocketType.REQ)
            zsock.Connect(My.Settings.InjectEventUri)

            For Each arg As String In args
                If key Is Nothing Then
                    key = arg
                Else
                    val = arg
                    msg.Add(key, val)
                    key = Nothing
                    val = Nothing
                End If
            Next arg
            Dim sw As New StringWriter
            sw.Write("{")
            Dim not_first As Boolean = False
            For Each kvp As KeyValuePair(Of String, String) In msg
                If not_first Then
                    sw.Write(",")
                End If
                sw.Write(String.Format("""{0}"" : ""{1}""", kvp.Key, kvp.Value))
                not_first = True
            Next
            sw.Write("}")
            Dim json As String = sw.ToString

            logger.Info(String.Format("v{0} :: {1} :: {2}", My.Settings.version, My.Settings.InjectEventUri, json))

            Dim timeoutAlarm As New System.Threading.Timer(AddressOf sendTimeout, "Send", 10 * 1000, System.Threading.Timeout.Infinite)

            zsock.Send(json, System.Text.Encoding.UTF8)
            Dim response = zsock.Recv(System.Text.Encoding.UTF8)
            logger.Info(String.Format("Response [{0}]", response))
        Catch ex As Exception
            logger.Fatal(String.Format("Exception: {0}", ex.Message))
            Environment.Exit(2)
        End Try
        Environment.Exit(0)
    End Sub
```

