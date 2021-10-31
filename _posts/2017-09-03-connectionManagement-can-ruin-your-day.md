---
title: ".NET &lt;connectionManagement&gt; can ruin your day"
excerpt: "Parallelizing http calls in .NET"
header:
  teaser: "assets/images/connectionManagement-can-ruin-your-day.png"
tags: 
  - .net
  - .netcore
  - .appconfig
  - csharp
  - networking
--- 

## ConnectionManagement

ConnectionManagement is a property found in app.config that allows to put a limit on number of connections to a specific host at tcp level.

This affects http calls, every one of these need established tcp connection "underneath", client needs to reuse or open a new port to listen for the response from the server.[^1]

Together with various http headers it can be misunderstood providing unexpected behaviour.

[^1]: <https://www.catchpoint.com/blog/http-transaction-steps/>

```xml
<configuration>  
  <system.net>  
    <connectionManagement>  
      <add address = "http://www.contoso.com" maxconnection = "4" />  
      <add address = "*" maxconnection = "2" />  
    </connectionManagement>  
  </system.net>  
</configuration>  
```
Setting ConnectionManagement is pretty straightforward, more specific entries are prioritized.

For configuration above - every host other than "http://www.contoso.com" has maxconnection set on 2 meaning that there can be only 2 parallel calls to exact same host.

## TCP

If ConnectionManagement property is not set in app.config it will default to 2.

It's recommended setting for calling external services (blacklisting), nevertheless in internal environment sometimes you want to parallelize it more for example if service doesn't support batching and/or there is need for a faster response.
```xml
<add address = "*" maxconnection = "2" />  
```

In .net core this doesn't apply, by default there is no limit.

## Connection: keep-alive

This magic header (usually turned on by default) allows to reuse existing tcp connection and not create a new one(every http call gets it own tcp connection), which is noticeable performance wise.  
It's worth noting it can take up to around 3 minutes to close tcp connection(socket) if there is unsent data from server waiting.

When You think about it turning it off with conjuction of high maxconnectionlimit can deplete ports that client host has available resulting in fatal exception when trying to make another http call.

## NTLM and UnsafeAuthenticatedConnectionSharing

Depletion of ports can also happen when using ntlm.
For security reasons ntlm is forcing every http call to initialize new tcp connection (authentication on tcp level).

Thus keep-alive doesn't matter in this scenario, which can lead again to port depletion.  
Luckily reusing connections can be forced by UnsafeAuthenticatedConnectionSharing on HttpWebRequest.

## Play around

Following code is:
* resolving domain name to ip
* calling sample api with HttpWebRequest, HttpClient, RestSharp (all of them respect maxconnectionlimit)
* checking underlying open connection and printing it

Disabling KeepAlive will result in opening more tcp connections.

{% highlight csharp linenos %}
using RestSharp;
using System;
using System.IO;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Net.NetworkInformation;
using System.Threading;
using System.Threading.Tasks;

namespace ConsoleApp
{
    class TextHolder
    {
        public string Text { get; set; }
    }

    class Program
    {
        private static readonly string _address = "www.example.com";
        private static readonly string _fullAddress = $"http://{_address}";
        private static readonly string _addressIp = Dns.GetHostAddresses(_address)[0].ToString();
        private static readonly HttpClient _httpClient = new HttpClient();
        private static readonly RestClient _restClient = new RestClient(_fullAddress);

        static void Main(string[] args)
        {
            var timer = new Timer(
                PrintOpenTcpConnections,
                new TextHolder(),
                TimeSpan.Zero,
                TimeSpan.FromSeconds(1)
            );
            Run();
            Console.ReadLine();
            timer.Dispose(); //without this timer is garbage collected
        }

        static async void Run()
        {
            Console.WriteLine("\n HttpWebRequest");
            for (var i = 0; i < 10; i++)
            {
                CallWithWebRequest();
            }
            await Task.Delay(2000);
            Console.WriteLine("\n HttpClient");
            for (var i = 0; i < 10; i++)
            {
                CallWithHttpClient();

            }
            await Task.Delay(2000);
            Console.WriteLine("\n RestSharp");
            for (var i = 0; i < 10; i++)
            {
                CallWithRestSharp();
            }
        }

        static async void CallWithWebRequest()
        {
            var req = WebRequest.CreateHttp(_fullAddress);
            req.Method = "GET";
            req.KeepAlive = true;
            req.Timeout = 5000;
            req.Pipelined = false;
            req.UnsafeAuthenticatedConnectionSharing = false;
            var resp = await req.GetResponseAsync();
            resp.Close(); // Used to fulfill the request
            PrintReceived();
        }

        static async void CallWithHttpClient()
        {
            var resp = await _httpClient.GetAsync(_fullAddress);
            PrintReceived();
        }

        static async void CallWithRestSharp()
        {
            var resp = await _restClient.ExecuteTaskAsync(new RestRequest(Method.GET));
            PrintReceived();
        }

        static void PrintOpenTcpConnections(object state)
        {
            var textHolder = (TextHolder)state;

            IPGlobalProperties ipGlobalProperties = IPGlobalProperties.GetIPGlobalProperties();
            TcpConnectionInformation[] tcpConnInfoArray = ipGlobalProperties.GetActiveTcpConnections();

            var text = tcpConnInfoArray
                .Where(tcpi => tcpi.RemoteEndPoint.Address.ToString() == _addressIp)
                .Aggregate("", (acc, item) => $"{acc} \n Local: ${item.LocalEndPoint} Remote: ${item.RemoteEndPoint}");
            if (text != string.Empty && textHolder.Text != text)
            {
                textHolder.Text = text;
                Console.Write("\nTCP Connections:");
                Console.WriteLine(text);
            }
        }

        static void PrintReceived()
        {
            Console.WriteLine($"Response received at {DateTime.Now.ToString("ss:ffff")}");
        }
    }
}
{% endhighlight %}

Cheers!
