## ViSiGenie4DSystems.Async
### About
Provides Windows IoT apps with an easy to use interface to 4D Systems� graphic display modules. IoT devices like the Raspberry Pi 2, 3, Dragonboard 410c or Minnowboard Max are connected to the 4D Systems display module via a Silicon Labs CP2102 USB to Serial UART Bridge Converter cable. Host communications with the display module is accomplished by using the singleton class named Host, which is located in the namespace ViSiGenie4DSystems.Async.SerialComm. Host Class Methods enable the discovery of multiple displays connected to the IoT device. Once device discovery is complete, the app can connect,  subscribe to reports, and send commands.

<img src="https://github.com/CQDX/visi-genie-4d-systems-async/blob/master/ViSiGenie4DSystems.Async/TouchDisplay.jpg">

* Adheres to the ViSi-Genie Communication Protocols, Objects, Properties, and Genie Magic idiom. 
  See [ViSi-Genie Reference Manual](http://www.4dsystems.com.au/productpages/ViSi-Genie/downloads/Visi-Genie_refmanual_R_1_11.pdf) 

* Intended for headed or headless Windows IoT applications requiring a non-primary display running on a Raspberry Pi 2 and 3, Arrow DragonBoard 410c or MinnowBoard MAX.  

* Connects and discovers one or more displays.
   
* Queues up all incomming ViSi Genie Report messages that originate from the display. 
  Forwards each Report as C# async events to the app's handler function. 
  For instance, the user presses a menu button object on the resistive touch display.  

### Roadmap

Future plans for ViSiGenie4DSystems.Async is to support Reactive Extensions - Main Library. 2.3.0-beta2 support UWP but currently this version of RX is not a stable Nuget package state.   
For example, ViSi Genie Reports could use RX. See [Reactive Extensions (Rx) � Part 1 � Replacing C# Events](http://rehansaeed.com/reactive-extensions-part1-replacing-events/)

### Hardware 

* [4D Programming Capable, USB to Serial UART Converter Cable](http://www.4dsystems.com.au/product/4D_Programming_Cable/)
* [uSD-4GB-Instustrial rated micro-SD card](http://www.4dsystems.com.au/product/uSD_4GB_Industrial/). The Phison brand is what 4D Systems display modules use. Industrial grade is optional. 
* [uLCD-35DT 3.5" TFT LCD Display Module with Resistive Touch] (http://www.4dsystems.com.au/product/uLCD_35DT_PI/) or other size module.
* [Raspberry Pi 3 - Model B - ARMv8 with 1G RAM] (https://www.adafruit.com/product/3055) or equivalent Raspberry P2, Broadcomm or Intel Atom SOC.

### Bring-Up Notes 

* From the 4D Workshop4 IDE, build and deploy your project to the display's micro-SD card. With ESD protection, safely remove micro-SD card from PC and install micron-SD card into display's pannel.

* In Microsoft Visual Studio, use the NuGet Package Manager Console to install the ViSiGenie.4DSystems.Async Library version 1.2.0: 

```
PM> Install-Package ViSiGenie.4DSystems.Async -Version 1.2.0
```
* In Microsoft Visual Studio, edit the project Package.appmanifest file. 
  Add a capability to support *serialcommunication*. 
  If *DeviceCapability* is not configured, then the *Host* will throw an exception when *Connect* gets called.

```XML
	<Capabilities>
		<DeviceCapability Name="serialcommunication">
			<Device Id = "any" >
				<Function Type="name:serialPort" />
			</Device>
		</DeviceCapability>
	</Capabilities>
```		
* Plug the USB programmers cable into the USB port on Pi 3 or equivalent IoT device. 
  Connect the other end of the cable into the backside of the display's 5 pins connector.

* Review and add code clips shown below to your project. 

* Build your ARM project in Visual Studio, cycle power on Pi 3. Finally deploy and debug your app :)

### Class Relationships

The singleton class named *Host* creates and manages the lifetime of serial device instances. For example, a Raspberry Pi 2 could have four different display modules, where each monitor is dedicated to specific subsystem monitoring and related interactive control. 

<img src="https://github.com/CQDX/visi-genie-4d-systems-async/blob/master/ViSiGenie4DSystems.Async/ClassDiagram.png">

### Host.Instance 

To demo Host.Instance, a new project in Microsoft Visual Studio needs to be created. 
For example, File New -> .NET Framework 4.5.2 -> Windows IoT Core -> Background Application (IoT).
The exemplar below shows how-to use the C# *Host* class in a Headless background application:

```C#
using System;
using System.Linq;
using Windows.ApplicationModel.Background;
using System.Diagnostics;
using Windows.System;
using System.Threading.Tasks;
using System.Threading;

using ViSiGenie4DSystems.Async.Enumeration;
using ViSiGenie4DSystems.Async.Message;
using ViSiGenie4DSystems.Async.SerialComm;

namespace DisplayHeadless
{
    public sealed class StartupTask : IBackgroundTask
    {
        BackgroundTaskDeferral _defferal;

        public async void Run(IBackgroundTaskInstance taskInstance)
        {		
			//1. The host application finds the connected device identifier string
			Task<List<string>> discoverDeviceIdsTask = Host.Instance.DiscoverDeviceIds();
	
			await discoverDeviceIdsTask;
			 
			//2. In this case, we demo a single display connected to the host. Therefor, get the first device identifier.
			//   The deviceId can logically thought of as the "token" used for all other subsequent HOST-DISPLAY I/O in methods 3 - 13 below.
			var deviceId = discoverDeviseIdsTask.Result.First();
			
			//3. Host baud rate must match the 4D Workshop project's baud rate, otherwise the Connect method will throw an exception
			var portDef = new PortDef(BaudRate.Bps115200);
			
			//4. Using the device identifier, Connect the host to the display
			Task connectTask = Host.Instance.Connect(deviceId, portDef);
			await connectTask;

			//5. Subscribe to potential ReportEventMessages that originate from the touch display.
			//   Notice the host app must provide a callback delegate 
			await Host.Instance.SubscribeToReportEventMessages(deviceId, ReportEventMessageHandler.Handler);

			//6. The host starts listening for display reports. 
			await StartListening(deviceId);
		
			//7. Create a WriteObjectValueMessage per the ViSi Genie manual. 
			//   The class, WriteObjectValueMessage, provides a helper constructor to create required data structure.
			const int displayFormId = 0;
			var writeObjectMessage = new WriteObjectValueMessage(ObjectType.Form, displayFormId);

			//8. Send the WriteObjectValueMessage to the display form. Notice index 0 applies to a hypothetical 4D Workshop4 project. 
			var cts = new CancellationTokenSource();
			await Host.Instance.Send(deviceId, writeObjectMessage, cts.Token);
		
			//9. Create a WriteStringASCIIMessage per the ViSi Genie manual. 
			//   The class, WriteStringASCIIMessage, provides a helper constructor to create required data structure.
			//   Notice displayStringId applies to a hypothetical 4D Workshop4 project. 
			const int displayStringId = 0;
			var writeStringMessage = new WriteStringASCIIMessage(deviceId, "Hello 4D Systems via Windows IoT!");
			
			//10. Send the WriteStringASCIIMessage to the display form. 
			await Host.Instance.Send(deviceId, writeStringMessage, cts.Token);
		
			//11. The host is no longer interested in receiving report event from the display
			await Host.Instance.UnsubscribeFromReportEventMessages(deviceId, ReportEventMessageHandler.Handler);

			//12. The host stops all monitoring. For example, StopListening is called with the app is shutting down or rebooting.
		    await Host.Instance.StopListening(deviceId);
    
			//13. Disconect from display by giving up the serial device resource to the garbarge collector
			Host.Instance.Disconnect(deviceId);

            _defferal.Complete();
        }        
    }
}
```

### Report Event Message Handler

When designing the Genie display application in Workshop, each Object can be
configured to report its status change without the host having to poll it (see ReadObject
StatusMessage class). If the object�s �Event Handler� is set to �Report Event� in the �Event� tab,
the display will transmit the object�s status upon any change. For example, Slider 3 object
was set from 0 to 50 by the user. The exemplar below shows how-to recieve Report Event Messages 
that occur when user touches an object on the display. 

The switch statement shown below is for demo purposes only! 
Instead, you need to customize the *Handler* method per project Workshop 4 requirements.

```C#
using System.Threading.Tasks;

using ViSiGenie4DSystems.Async.Enumeration;
using ViSiGenie4DSystems.Async.Event;
using ViSiGenie4DSystems.Async.Message;

namespace HeadlessDemoApp 
{
	public class ReportEventMessageHandler 
    {
        public async void Handler (object sender, DeferrableDisplayEventArgs e)
        {
            using (var deferral = e.GetDeferral())
            {
                //Run task message cracker in thread pool thread
               await Task.Run( () =>
               {
					ReportEventMessage hotReportEventMessage = (ReportEventMessage)sender;
					//
					//TODO: Switch on specific  identifiers per specific Workshop 4D project layout
					//      EXAMPLES BELOW SHOWS HANDLING VARIOUS 4D BUTTON HANDLERS...
					//
					switch (hotReportEventMessage.ObjectType)
					{
						case ObjectType.Button4D:
							{
								switch (hotReportEventMessage.ObjectIndex)
								{
									case 0:
										{
											//TODO: User pressed button id 0 on display
											break;
										}
									case 1:
										{
											//TODO: User pressed button id 1 on display
											break;
										}
								}
								break;
							}
						case ObjectType.Form:
							{
								switch (hotReportEventMessage.ObjectIndex)
								{
									case 0:
										{
											//TODO: user activated Form 0 on display
											//WARNING: DON'T BLOCK
											break; 
										}
									case 1:
										{
											//TODO: user activated Form 1 on display
											//WARNING: DON'T BLOCK
											break; 
										}
								}//END OF SWITCH

								break;
							}//END OF FORM ACTIVATE
						case ObjectType.Winbutton:
							{
								//Winbutton event was recevied from display
								switch (hotReportEventMessage.ObjectIndex)
								{
									case 0:
										{
											//EXAMPLE:  shutdown headless app 
											ShutdownManager.BeginShutdown(ShutdownKind.Shutdown, new TimeSpan(0));
											break;
										}

									case 5:
										{
											//EXAMPLE: reboot headless app 
											ShutdownManager.BeginShutdown(ShutdownKind.Restart, new TimeSpan(0));
											break;
										}
								}
								break; 
							}//END OF WIN BUTTON

						default:
							{
							    // TODO: application specific logic..
								break;
							}
					} //end of switch
				}); //end async thread pool execution
			}
        }
    }
}
```

### Report Object Status Message Handler 

The Host Sends a ReadObjectStatusMessage when it wants to determine the current value of a
specific object instance. Upon receipt of this message the display will reply with either a NAK
(in the case of an error) or the ReportObjectStatusMessage message. 
Not all Workshop 4 projects use this features.

```C#
using System.Threading.Tasks;
using ViSiGenie4DSystems.Async.Event;
using ViSiGenie4DSystems.Async.Message;

namespace HeadlessDemoApp //Contrived example
{
    public class ReportObjectStatusMessageHandler
    {
        public async void Handler(object sender, DeferrableDisplayEventArgs e)
        {
            using (var deferral = e.GetDeferral())
            {
                await Task.Run(() =>
                {
                    ReportObjectStatusMessage hotReportObjectMessage = (ReportObjectStatusMessage)sender;

                    //TODO: Switch on the report object status message ...
                });
            }
        }
    }
}
```