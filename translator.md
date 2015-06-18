---
layout: default
title: Sample Applications
permalink: /win10/samples/Translator.htm
---

<div class="container" markdown="1">
##Telephone system with Translation service
Please read before continue:
This demo is built on MBM. The image we use has some private fixes in it. So you probably will not be able to download the MBM image and build this demo yet.
But all the private fixes should be coming in soon on latest MBM image. Just watch out for that and try it out later.

Also same situation holds for RPi2. In next few release of RPi2, you will be able to build this demo on it. Keep an eye on our announcements.

1. Technologies Involved

	- Speech Recognition
	- Message Socket Transmission between devices
	- Bing Translator API Call
	- Speech Synthesis
	- GPIO Events/Interrupt
2. Project Overview

	There are two MBM devices, and each device is attached to a toggle switch through GPIO pins. The toggle switch is used to select language.
	We can speak into device 1, the input message will be sent to device 2, on device 2 the message is sent to Bing Translator for translation.
	Then the translated message will be returned back and played out on device 2.
	So It is basically like telephone system. You speak on this side, the people will hear you on the other side in a different language, depending on the switch selection.
	
	The switch controls both Input and Output Language. For example, if the switch is on Chinese, you can speak Chinese into this device, and also the message you received from the other device
	will be played out in Chinese as well, vise versa.
	
## Parts needed
- [Two MBM Devices](http://www.minnowboard.org/meet-minnowboard-max/)
- [Two toggle switch](http://www.digikey.com/product-detail/en/ATE1D-2M3-10-Z/563-1157-ND/1792018)
- [Two headSet](https://www.microsoft.com/hardware/en-us/b/lifechat-lx-4000-for-business/7YF-00001)
- Two HDMI Monitor(optional)
- 1 breadboard and a couple of wires

## Parts Connection
1. Toggle Switch to MBM
	- Toggle Switch Middle Pin to GND on MBM
	- Toggle Switch Lower Pin to Pin 2 on MBM
	- Toggle Switch Upper Pin to Pin 9 on MBM
	
	Same as the other toggle switch and MBM board
	
2. Other Connection
	- Plug each headset into each device
	- Connect HDMI monitor into each Device
	(Note: The HDMI monitor is optional; and It will be used to display the debugging message, like it show the speech content you speak into device, and also the message your device received
	from other device, and the device conneciton status etc on; The project will run fine without the monitor)
	
## 	Go through the code
For the complete code, please look for [here](https://github.com/MagicBunny/SRTranslator/tree/master/SampleCode/ShenzhenMBM);
Below we go through all the main pieces of the project;

a. Main Enterance
{% highlight c# %}
public MainPage()
{
	this.InitializeComponent();
	listener = new StreamSocketListener();
	startListener();
	InitGpio();
}
{% endhighlight %}

b. Initialze GPIO 

Note:  switchPin1.ValueChanged is switch event/interrupt. It gets called everytime you switch the switch. Therefore, a new process will be invoked very time you change the switch status.

{% highlight c# %}
        /// <summary>
        /// GPIO Section
        /// </summary>
        // MBM Pin Parameter
        const int switchPinNum1 = 2;
        const int switchPinNum2 = 9;
		
        public async void InitGpio()
        {
            gpioController = GpioController.GetDefault();
            try
            {
                switchPin1 = gpioController.OpenPin(switchPinNum1);
                switchPin2 = gpioController.OpenPin(switchPinNum2);

                switchPin1.SetDriveMode(GpioPinDriveMode.Input);
                switchPin2.SetDriveMode(GpioPinDriveMode.Input);

                await startSRProcess();

                switchPin1.DebounceTimeout = new TimeSpan(0, 0, 0, 1, 0);

                // value change
                switchPin1.ValueChanged += async (s, e) =>
                {
                    if (isListening)
                    {
                        try
                        {
                        await speechRecognizer.ContinuousRecognitionSession.StopAsync();
                        }
                        catch
                        {

                        }

                    }
                   // await selectLang();
                    await startSRProcess();
                    
                };

            }
            catch (Exception ex)
            {
            }
        }
{% endhighlight %}

c. Open Listener on this device: When the application starts, it opens the port on it to be ready to receive the message sent from other device.
OnConnection will be called whenever there are messages sent otver. The translation service will be called here and the message will be converted to SpeechSynthesisStream to be played out.
{% highlight c# %}
        private async void startListener()
        {
            if (String.IsNullOrEmpty(ConstantParam.SelfPort))
            {
                StatusText.Text = "Please provide a service port.";
                return;
            }
            listener.ConnectionReceived += OnConnection;

            try
            {
                await listener.BindServiceNameAsync(ConstantParam.SelfPort);
                StatusText.Text = "Listening on port: " + ConstantParam.SelfPort;
                ReceivedText.Text = "";
            }
            catch (Exception exception)
            {
                if (SocketError.GetStatus(exception.HResult) == SocketErrorStatus.Unknown)
                {
                    throw;
                }
                StatusText.Text = "Start listening failed with error: " + exception.Message;
            }
        }
        private async void OnConnection(StreamSocketListener sender, StreamSocketListenerConnectionReceivedEventArgs args)
        {

            DataReader reader = new DataReader(args.Socket.InputStream);
            try
            {
                while (true)
                {
                    // Read first 4 bytes (length of the subsequent string).
                    uint sizeFieldCount = await reader.LoadAsync(sizeof(uint));
                    if (sizeFieldCount != sizeof(uint))
                    {
                        // The underlying socket was closed before we were able to read the whole data.
                        return;
                    }

                    // Read the string.
                    uint stringLength = reader.ReadUInt32();
                    uint actualStringLength = await reader.LoadAsync(stringLength);
                    if (stringLength != actualStringLength)
                    {
                        // The underlying socket was closed before we were able to read the whole data.
                        return;
                    }

                    // Display the string.
                    string text = reader.ReadString(actualStringLength);

                    // Get the language and the content received
                    char delimiter = ':';
                    string[] res;
                    res = text.Split(delimiter);
                    string InLang = "en";
                    if (res[0].Contains("zh"))
                    {
                        InLang = "zh-CHS";
                    }
                    string outLang = "en";
                    if (selectedLang.LanguageTag.Contains("zh"))
                    {
                        outLang = "zh-CHS";
                    }
                    string content = res[1];

                    Translator Trans = new Translator(content, InLang, outLang);
                    string translatedS = Trans.GetTranslatedString();

                    SpeechSynthesisStream stream = await synthesizer.SynthesizeTextToStreamAsync(translatedS);
                    var ignored = Dispatcher.RunAsync(Windows.UI.Core.CoreDispatcherPriority.Normal, () =>
                    {
                        media.SetSource(stream, stream.ContentType);
                        media.Play();
                        ReceivedText.Text = text;
                    });
                }
            }
            catch (Exception exception)
            {
                // If this is an unknown status it means that the error is fatal and retry will likely fail.
                if (SocketError.GetStatus(exception.HResult) == SocketErrorStatus.Unknown)
                {
                    throw;
                }
                var ignored = Dispatcher.RunAsync(Windows.UI.Core.CoreDispatcherPriority.Normal, () =>
                {
                    StatusText.Text = "Read stream failed with error: " + exception.Message;
                    StatusText.Text += "Reconnecting Later!";              
                    connected = false;
                    CoreApplication.Properties.Remove("clientSocket");
                    CoreApplication.Properties.Remove("clientDataWriter");
                });
            }
        }
{% endhighlight %}

d. Send Data to another device;
First It makes connection to the other device, and once the connection is established, and then it sends the speech recognition message over 
{% highlight c# %}
        private async void SendDataToHost(string dataToBeSent)
        {
            //CoreApplication.Properties.Remove("clientSocket");
            if (!connected)
            {
                if (!CoreApplication.Properties.ContainsKey("clientSocket"))
                {
                    HostName hostName;
                    try
                    {
                        hostName = new HostName(ConstantParam.ServerHostname);
                    }
                    catch (ArgumentException)
                    {
                        return;
                    }

                    StreamSocket locsocket = new StreamSocket();
                    // save the soket so subsequence can use it
                    CoreApplication.Properties.Add("clientSocket", locsocket);
                    try
                    {
                        await locsocket.ConnectAsync(hostName, ConstantParam.ClientPort);
                        await Dispatcher.RunAsync(Windows.UI.Core.CoreDispatcherPriority.Normal, () =>
                        {
                            StatusText.Text += "Connected!";
                        });
                        connected = true;
                    }
                    catch (Exception exception)
                    {
                        // If this is an unknown status it means that the error is fatal and retry will likely fail.
                        if (SocketError.GetStatus(exception.HResult) == SocketErrorStatus.Unknown)
                        {
                            await Dispatcher.RunAsync(Windows.UI.Core.CoreDispatcherPriority.Normal, () =>
                            {
                                StatusText.Text += "Connect to host failed";
                            });
                            //throw;
                            return;
                        }

                    }
                }

                //StatusText.Text = "Must be connected to send";
                //  return;
            }
            // If the connection was not setup still, we return instead of continuing to send data which will break the code.
            if (!connected)
            {
                return;
            }
            object outValue;
            StreamSocket socket;
            if (!CoreApplication.Properties.TryGetValue("clientSocket", out outValue))
            {
                return;
            }
            socket = (StreamSocket)outValue;
            // Create a DataWriter if we did not create one yet. Otherwise use one that is already cached.
            DataWriter writer;
            if (!CoreApplication.Properties.TryGetValue("clientDataWriter", out outValue))
            {
                writer = new DataWriter(socket.OutputStream);
                CoreApplication.Properties.Add("clientDataWriter", writer);
            }
            else
            {
                writer = (DataWriter)outValue;
            }

            // Write first the length of the string as UINT32 value followed up by the string. 
            // Writing data to the writer will just store data in memory.
            string stringToSend = dataToBeSent;
            writer.WriteUInt32(writer.MeasureString(stringToSend));
            writer.WriteString(stringToSend);

            // Write the locally buffered data to the network.
            try
            {
                await writer.StoreAsync();
                //  SendOutput.Text = "\"" + stringToSend + "\" sent successfully.";
            }
            catch (Exception exception)
            {
            }

        }
{% endhighlight %}

e. Bing Translator: For how to use the MS Translator, Click [here](https://www.microsoft.com/translator/getstarted.aspx);

MS translator service is hosted in Azure. You need to get a Azure account then register the translation service; Then you will get the clientID and ClientSecret;
Use that ClientID and ClientSecret to use the service.
I have made the namespace Translator as below;
{% highlight c# %}
    class Translator
    {
        private string originalString;
        private string translated;

        public Translator(string s, string from, string to)
        {
            this.originalString = s;
            AdmAuthentication admAuth = new AdmAuthentication(ConstantParam.clientid, ConstantParam.clientsecret);
            Task<AdmAccessToken> admToken = admAuth.GetAccessToken();
            this.translated = TranslateMethod("Bearer" + " " + admToken.Result.access_token, this.originalString, from, to);
        }

        public string GetTranslatedString()
        {
            return this.translated;
        }

        public static string TranslateMethod(string authToken, string originalS, string from, string to)
        {
            string text = originalS; 
            //string from = "en";
            //string to = "zh-CHS";
            //string from = Constants.from;// "zh-CHS";
            //string to = Constants.to; // "en";

            string transuri = ConstantParam.ApiUri + System.Net.WebUtility.UrlEncode(text) + "&from=" + from + "&to=" + to;

            HttpWebRequest httpWebRequest = (HttpWebRequest)WebRequest.Create(transuri);
            //  httpWebRequest.ContentType = "application/x-www-form-urlencoded";
            httpWebRequest.Headers["Authorization"] = authToken;
            //httpWebRequest.Method = "GET";
            string trans;

            Task<WebResponse> response = httpWebRequest.GetResponseAsync();

            using (Stream stream = response.Result.GetResponseStream())
            {
                System.Runtime.Serialization.DataContractSerializer dcs = new System.Runtime.Serialization.DataContractSerializer(Type.GetType("System.String"));
                //DataContractJsonSerializer dcs = new DataContractJsonSerializer(Type.GetType("System.String"));
                trans = (string)dcs.ReadObject(stream);
                return trans;

            }
        }
    }
{% endhighlight %}
 
In order to use the translator, Just need to do below
{% highlight c# %}
	Translator Trans = new Translator(content, InLang, outLang);
	string translatedS = Trans.GetTranslatedString();
{% endhighlight %}

f. Constant Parameter Configuration
Note: For the two devices, this configuration file will need to be changed a little. As each device is using the other device's name as the serverHostName to send data to.
Pay attention to the parameters SelfPort/ClientPort/ServerHostname and Commented out lines to make sure you understand before deploying to device

{% highlight c# %}
    class ConstantParam
    {
        public static readonly string DatamarketAccessUri = "https://datamarket.accesscontrol.windows.net/v2/OAuth2-13";
        public const int RefreshTokenDuration = 9;

        public static string clientid = "iotmaker";
        public static string clientsecret = "nA/M181Yi4PF08c0n8IjEz1b9SWSkzj0+wukNQBwcf4=";
        public static string ApiUri = "http://api.microsofttranslator.com/v2/Http.svc/Translate?text=";

        /// <summary>
        ///  Uncomment if you are deploying to Device 1
        /// </summary>
        /// 

        // Stream Socket parameters
        public static string SelfPort = "8083";
        public static string ClientPort = "8082";
        // public static string ServerHostname = "shenzhen_mbm2";
        //public static string ServerHostname = "Shenzhenrpi1";
        public static string ServerHostname = "mbmdemo4";


        /// <summary>
        ///  Uncomment if you are deploying to Device 2
        /// </summary>
        /// 
        //public static string SelfPort = "8082";
        //public static string ClientPort = "8083";
        ////public static string ServerHostname = "shenzhen_mbm1";
        ////public static string ServerHostname = "Shenzhen_Rpi2";
        //public static string ServerHostname = "mbmdemo3";
    }
{% endhighlight %}


</div>








