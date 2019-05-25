---
title: "Capturing infrared packets with Arduino and plotting them with HTML5 canvas"
date: 2013-10-27T00:00:00+01:00
---
I had an old DVD player lying around that wasn't working anymore so I stripped the infrared sensor from it (Note this is not a standard IR phototransistor/diode, it has a modulated, logic-level output).

![Setup](https://s3-eu-west-1.amazonaws.com/mattharrison/setup1.JPG)

I found the fantastic [Arduino-IRremote](https://github.com/shirriff/Arduino-IRremote) library by Ken Shirriff. This library abstracts away all the processing you need to start getting commands from your remote. It also has decoders for the most popular IR protocols which will strip out any signal headers and return just the data bits.

It turned out my remote conformed to the NEC protocol. The NEC protocol sends:

* a 9ms leading pulse burst (16 times the pulse burst length used for a logical data bit)
* a 4.5ms space
* the 8-bit address for the receiving device
* the 8-bit logical inverse of the address
* the 8-bit command
* the 8-bit logical inverse of the command
a final 562.5µs pulse burst to signify the end of message transmission.

So the total data bits are 32 and the data package may look like this:

<pre><code class='language-c'>∨ device address  ∨ command
10110100 01001011 11100111 00011000
         ∧  ~address       ∧ ~command</code></pre>

I wired the sensor with my arduino as following:


![Breadboard](https://s3-eu-west-1.amazonaws.com/mattharrison/breadboard.png)

![Wiring](https://s3-eu-west-1.amazonaws.com/mattharrison/setup2.JPG)

I then uploaded the following sketch to the Arduino (this is almost exactly the same as the example sketch provided with the library except I write the data as binary rather than hex.

<pre><code class='language-c'>#include <IRremote.h>
int RECV_PIN = 11;
IRrecv irrecv(RECV_PIN);
decode_results results;

void setup()
{
  Serial.begin(9600);
  irrecv.enableIRIn();
}

void loop() {
  if (irrecv.decode(&results)) {
    Serial.println(results.value, BIN);
    irrecv.resume();
  }
}</code></pre>

I can now start collecting commands from the sensor but I need something to do with them. I used the great [node-serialport](https://github.com/voodootikigod/node-serialport) module for Node to collect data from the serial port. I also used the [Express framework](http://expressjs.com/) and [Socket.IO](http://socket.io/) to create a persistent connection to a webpage. The server code for Node is below:

<pre><code class='language-javascript'>var serialport = require("serialport");
var SerialPort = serialport.SerialPort
var serialPort = new SerialPort("/dev/tty.usbmodem1411", {
	parser: serialport.parsers.readline("\n"),
	baudrate: 9600
});

serialPort.open(function () {
	var app = require('express')()
	, server = require('http').createServer(app)
	, io = require('socket.io').listen(server);

	server.listen(8080);

	app.get('/', function (req, res) {
		res.sendfile(__dirname + '/index.html');
	});

	io.sockets.on('connection', function (socket) {
		serialPort.on('data', function(data) {
			io.sockets.emit('data', { data: data});
		});
	});
});</code></pre>

When the websocket client recieves a `data` event, it passes the data to a javascript function which then draws a logic chart on the screen using canvas, showing the bits recieved from the remote. This has the protocol headers stripped and is just the command.

<pre><code class='language-javascript'>//Takes a string like 101010001010
function drawChart(string){

		ctx.strokeStyle = '#ff0000';
		ctx.clearRect(0, 0, 960, 360);
		ctx.beginPath();
		ctx.lineWidth = 3;
		ctx.moveTo(0,330);

		var bits = string.split("");
		var travel = Math.floor(960/string.length);

		var lastBit = "30";
		var x = 0;

		for(var i in bits){
			if(bits[i] === "1"){
				if(lastBit === "1"){
					ctx.lineTo(x+travel, 30);
					x = x + travel;
					ctx.stroke();
				} else {
					ctx.lineTo(x, 30);
					ctx.lineTo(x+travel, 30);
					x = x + travel;
					ctx.stroke();
				}
			} else {
				if(lastBit === "1"){
					ctx.lineTo(x, 330);
					ctx.lineTo(x+travel, 330);
					x = x + travel;
					ctx.stroke();
				} else {
					ctx.lineTo(x+travel, 330);
					x = x + travel;
					ctx.stroke();
				}
			}

			ctx.font = "bold 16px Menlo";
			ctx.fillText(bits[i], x-23, 355);
			ctx.fillText(bits[i], x-23, 15);

			lastBit = bits[i];
		}
		ctx.strokeStyle = '#000000';

		ctx.lineWidth = 0.5;
		for(var i in bits){
			ctx.moveTo(travel * i,360);
			ctx.lineTo(travel * i, 0);
		}

		ctx.stroke();
	};</code></pre>


![Logic plot](https://s3-eu-west-1.amazonaws.com/mattharrison/plot.png)

When I have a IR LED I can start replaying these commands and switching my TV on and off through the web, I'll post updates of this when I get round to it.
