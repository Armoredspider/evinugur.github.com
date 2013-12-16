---
layout: post
title: "Reverse Engineering the FRC NetConsole"
date: 2013-12-08 21:00:00
categories: FIRST FRC NetConsole
root: "../"
description: "A tutorial on how to reimplement National Instruments' FRC NetConsole, and how to extend it wiht your own features"
---

##Getting Started

WPI and National Instruments both provide a wide palette of tools for debugging a robot in the FIRST Robotics competition, and as sophisticated and useful as these tools can be, sending data to standard out is still a useful way of obtaining information during runtime. 

Because the typical robot doesn't have a computer screen, methods that interface with standard out, be it `printf()` or `System.out.println()`, will send data out through a UDP stream at a level of abstraction lower than where the FRC programmer works. In order to view the text output, National Instruments created a Windows program called NetConsole which generates a robot IP address based on team number, prints the data from the robot's standard out onto a screen, and also takes text input to interface with the robot's operating system: VxWorks.

<figure style = "padding-top:20px">
	<img src = "{{ post.root }}/images/netconsole.png" alt = "National Instruments NetConsole">
	<figcaption>An example of National Instrument's NetConsole</figcaption>
</figure>

Seeing as the protocol from fetching data to print, it should be easy to implement our own NetConsole, and add some features that we want to it. This proves to be advantageous for a number of reasons:

* A homebrew NetConsole can be run cross-platform depending on how it is written - for this post I'll be writing one that should run on multiple operating systems.
* A homebrew NetConsole can be integrated into a preexisting application, one year my team added the data stream into a custom dashboard we used because we valued our screen real estate. 
* <span id ="above">NetConsole can be stylized however you'd like it to be. *This is especially useful if you're not fond of the harsh green on black color scheme*
  * Additionally, you could format output depending on data. For example, `printf("Hello world\n");` could be printed on the screen differently than `printf("Hello world!!!\n");`
* You can whip up a quick and dirty networking protocol because all the infrastructure is already there. This coupled with the fact that you have the flexibility to work in higher level languages when reimplementing NetConsole allows you to quickly wire up NetConsole output to code from another model, gem, etc. *That said, this is data over a UDP protocol, so nothing mission critical.* 
* Even with debuggers, they're some FIRST specific benefits to using standard out. At least with the WindRiver environment, it takes time to link the debugger to the target codebase, **and** a non-compiled version of WPILib. When VxWorks encounters an error at runtime, it'll print diagnostic information to standard out. An improperly access pointer, or a null pointer exception can be identified through print data from standard out before actually running robot code. 


##My Implementation

To hint some of the points I raised above, specifically being cross platform and extensibility, I'm going to reimplement NetConsole in JavaScript using Node.js, and step through the steps of the process above. I have the finished project hosted on a GitHub repository <a href="https://github.com/FRCTeam1073-TheForceTeam/netconsole.js">here</a>, and it can be installed through <a href="http://npmjs.org">npm</a> with:

{% highlight bash %}
npm install -g frcnetconsole
{% endhighlight %}

In order to follow along with this tutorial, you'll need to install Node.js. After that, make a file called `netconsole.js` or whatever you deem fit. If you're not familiar with Node, it's easy to run a JavaScript program by entering `node netconsole.js` into the command line.

###Importing Network APIs
I did say this was easy right? All we have to do to get a minimal NetConsole created is to print data from a socket, and we can leverage the cross-platform networking APIs included with a stock installation of Node.js in order to do so. For listening to data, all we have to know is that data is broadcasted outward on UDP port `6666`.

{% highlight javascript %}
// initialize networking objects
var dgram = require("dgram");
var listener = dgram.createSocket("udp4");

// constants
var PORT_IN = 6666;
{% endhighlight %}

###Printing Data From the Robot

From there, all we have to do is implement a callback function to print data whenever the Node.js UDP4 object says it has a new message, and bind that very object to the defined UDP port.

{% highlight javascript %}

listener.on("message", function(msg, rinfo) {
	// prints a human readable version of the message, whenever one is received
	process.stdout.write(msg.toString());
});
listener.bind(PORT_IN); //tells the object where to look for UDP data.
{% endhighlight %}

Ensure that your robot is on the same network as your computer, and then fire up the JavaScript program. If you turn on your robot, you'll start to see any initialization logic for VxWorks alongside any text your robot's executable is programmed to print. Success! 

###Sending Data to the Robot

####Initializing More Constants and Objects 

Sending down UDP data to VxWorks, can also be done with a dozen or so lines of code. We'll need to initialize a new UDP object that's responsible for sending down formatted data to a specific port off of your robot's IP address. We'll also need to provide a way to handle user input, this too is also easy to write by asking Node.js for a new module: `readline`. 

{% highlight javascript %}
var listener = dgram.createSocket("udp4");
var sender = dgram.createSocket("udp4"); 

// get keyboard input
var readline = require("readline");
var scanner = readline.createInterface(process.stdin, process.stdout);

scanner.setPrompt("->"); // a string to appear whenever user input can be entered
scanner.prompt();	// tell the scanner to start asking for input

// constants
var PORT_IN = 6666;
var IP_ADDR = "10.10.73.2";
var PORT_OUT = 6668;
{% endhighlight %}

#### Finding your Robot's IP Address
Assuming your robot was imaged properly with the official FIRST utilities, it should be easy to determine the IP address. These addresses follow the pattern `10.xx.yy.2` where `x` and `y` are is the team number that the imaging tool was supplied when flashing your robot. If you're team number does not contain 4 digits you'll have to pad it with enough zeroes from the left.

####Writing the Callback Function
Now that everything has been defined, we have to implement yet another callback function. This function is called whenever our readline interface, `scanner`, is told it has read a new line. We used `process.stdin` when defining `scanner`, so the function will be called whenever the enter key is pressed. Before we send our data down, the data has to be suffixed with `"\r\n"` in order for VxWorks to identify that text as something that is a command. And finally, we have to take the formatted text input and convert it into an array of bytes in order for VxWorks to properly read the data. The byte conversion is done with the `Buffer` class from Node.js.

{% highlight javascript %}
// cmd is the text the user inputted
scanner.on("line", function(cmd)) {
	if (cmd !== "") { // let's not bother to send down an empty string
		var buffer = new Buffer(cmd + "\r\n");	// add rollback and carriage return chars
		// send the first character, until the last of buffer to IP_ADDR @ PORT_OUT
		sender.send(buffer, 0, buffer.length, PORT_OUT, ID_ADDR);
	}
}
{% endhighlight %}

If everything is networked properly, boot up your robot and run the JavaScript program. Once VxWorks has finished booting, enter a valid VxWorks command, if you don't know one try `ls` and you should get a full directory listing of your working directory. *For a full list of shell commands for VxWorks, check out this <a href ="http://csg.lbl.gov/pipermail/vxwexplo/2003-March/000762.html">link</a>.*


###Just the Beginning
Now that you  have a fully fledged NetConsole port, you can expand on it however you see fit. If you've done anything interested, shoot me an email and I'll feature it on this page alongside your team number. One such implementation, would be to filter out data that has a specific string in it to show that it's significant, let's use the example I gave <a href = "#above">above</a>: `"!!!"`.

####Robot Side Code
Let's say we have a DriveTrain system that uses classes from <a href = "https://github.com/FRCTeam1073-TheForceTeam/WPILibExtensions">WPILibExtensions</a> and prints very verbosely to the console. It'll print joystick values from a Joystick class <a href = "https://github.com/FRCTeam1073-TheForceTeam/WPILibExtensions/blob/master/SmartJoystick.h">`SmartJoystick`</a> and prints y values before sending them to a CANJaguar. The robot will also talk to the console whenever one of these CANJaguars times out and otherwise is not part of the network *(this is a feature of WPILibExtensions's <a href = "https://github.com/FRCTeam1073-TheForceTeam/WPILibExtensions/blob/master/SmartCANJaguar.h">`SmartCANJaguar`</a>)*.

{% highlight c++ %}
#include "DriveTrain.h"
#include "../RobotMap.h"
DriveTrain::DriveTrain : public Subsystem("DriveTrain") {
	leftMotor = RobotMap::leftDriveMotor;
	rightMotor = RobotMap::rightDriveMotor;	
}

void DriveTrain::Set(float left float right) {
	printf("L:%f\tR:%f\n", left, right);
	if(!(leftMotor->ExistsOnBus()) {
		puts("CRITICAL: left drive motor isn't on bus!!!");
	}
	if(!(rightMotor->ExistsOnBus()) {
		puts("CRITICAL: right drive motor isn't on bus!!!");
	}
	leftMotor->Set(left);
	rightMotor->Set(right);
}
{% endhighlight %}

####Corresponding JavaScript Modifications to our NetConsole

Depending on what we want printed, we can add a Boolean switch to not print data that has `"!!!"` in it. 

Place this global variable somewhere in your JavaScript

{% highlight javascript %}
var OnlyPrintCritical = true;
{% endhighlight %}

Then, depending on this variable, we can just skip over things we don't want to print in our listener object's callback function.

{% highlight javascript %}

listener.on("message", function(msg, rinfo) {
	// prints a human readable version of the message, whenever one is received
	var str = msg.toString();
	if (OnlyPrintCritical && str.indexOf("!!!") !== -1)
		return;
	process.stdout.write(str);
});
{% endhighlight %}