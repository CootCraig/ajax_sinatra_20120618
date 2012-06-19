# Linked Servers Distributing AJAX events from multiple sources

A presentation to Not Just Ruby

## Project Overview
## VB.net 0mq Event Injector
```
    Sub Main(ByVal args As String())
        Try
            log4net.Config.XmlConfigurator.Configure()
            logger = log4net.LogManager.GetLogger("app")
            Dim zmq_context As New ZMQ.Context
            Dim zsock As ZMQ.Socket = Nothing
            Dim msg As New Dictionary(Of String, String)
            Dim key As String = Nothing
            Dim val As String = Nothing

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

