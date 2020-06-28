# tape-synth

I got this idea for the cassette synthesizer from [Onde Magnétique](http://www.ondemagnetique.com/) who got the idea from the [Mellotron](http://www.mellotron.com/). The idea is that you can record a single drone onto a tape and then play that back at various speeds to modulate the pitch. 

The neat thing about this synthesizer is that it has a very "analog" quality to the changing of notes - the pitch often *slides* between notes in a neat way. This pitch sliding is called [*portamento*](https://en.wikipedia.org/wiki/Portamento). It also is versatile because you can record *any* sound to the tape and use that as your synthesizer.

Here's what it sounds like with the "angry bees" synth sample applied to the tape. The video shows me playing another synth which is not producing any sound and only controlling the cassette player via MIDI:


<p align="center"><a href="https://www.youtube.com/watch?v=LdBik_Zlwy0"><img src="https://img.youtube.com/vi/LdBik_Zlwy0/0.jpg" alt="Demo of playing" style="max-width:200px;"></a></p>



## Making a cassette synthesizer 


To make one of these things is actually *really* easy. I found a great video from Analog Industries showing exactly how to [hack a cassette player](https://www.youtube.com/watch?v=pF6Yegj7A9o) to add voltage control to the cassette player. I followed that and then wrote a simple MIDI controller in the browser to modulate the voltage to specific notes. 

These instructions will take you through the hardware (Arduino / Cassette player) and the software for the MIDI controller.

### Supplies

- GE 3-5362A walkman ($15, eBay has tons of them)
- [Arduino](https://www.amazon.com/gp/product/B008GRTSV6/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B008GRTSV6&linkCode=as2&tag=scholl-20&linkId=7bcd2ae0b8147ff819937b73da545cfb) ($23)
- [MCP4725 DAC](https://www.amazon.com/gp/product/B00SK8MBXI?ie=UTF8&tag=scholl-20&camp=1789&linkCode=xm2&creativeASIN=B00SK8MBXI) ($11)
- [Audio jack breakout](https://www.amazon.com/gp/product/B07Y8KR21P?ie=UTF8&tag=scholl-20&camp=1789&linkCode=xm2&creativeASIN=B07Y8KR21P) ($8)
- [Jumper wires](https://www.amazon.com/gp/product/B07GD2BWPY?ie=UTF8&tag=scholl-20&camp=1789&linkCode=xm2&creativeASIN=B07GD2BWPY) ($5, if you don't have)
- [Solder iron + supplies](https://www.amazon.com/gp/product/B07Q2B4ZY9?ie=UTF8&tag=scholl-20&camp=1789&linkCode=xm2&creativeASIN=B07Q2B4ZY9) ($25, and it lasts a lifetime)

## Hacking the GE 3-5362A walkman

_Note:_ You don't necessarily have to use the GE 3-5362A. Any walkman with a variable playback will work. You'll just have to figure out how to hook up the voltage :)

First thing is to open up the walkman. There are four screws on the back. Just unscrew them and open it carefully. The power lines are connected on the back plate so just don't rip those out.

![Hacking the GE 3-5362A walkman.](https://schollz.com/img/s1/overview.jpg)

To get this walkman working for us we will solder two new components. First we will solder in the `Vin` which will allow us to control the speed of the cassette player with a voltage. Then we will add a `Line in` which will let us record directly onto tape (in case you don't have a tape deck).

If you don't know how to solder - don't sweat. Its easy. Check out [this video](https://youtu.be/HTy9Z9LpA2U?t=1011) which shows you how to use the existing solder pad and put something onto it.

### Adding the `Vin`

Locate the dial that says "Variable Speed Playback". This is where we will splice in two lines. I like to use red for active and brown/green for ground. Attach the active line to `VS+` and the ground to the pad right below the one labeled `B+`.

![Solder a cable to VS+ and one to the pad next to B+.](https://schollz.com/img/s1/vs.jpg)

A note - I like to use jumper cables that have a female end so I can easily plug stuff into here!

### Add in a `Line in`

Locate the red and black cables plugged into pads labeled "MIC-" and "MIC+". You can solder and remove these cables and attach your own from the audio breakout cable. Just solder red to "MIC+" and black to "MIC-".

![Solder a line-in via the MIC- and MIC+.](https://schollz.com/img/s1/linein.jpg)

### Record the tape

Put in a tape and record via the line in! Record anything you want, usually a single drone on C works well as a starting point. Record a long time - 30 minutes or so (this would be a good place for tape loops if you have them!).

![Recording a drone from my OP-1 to the tape.](https://schollz.com/img/s1/rec.jpg)

## Setup an Arduino

Simply connect the Arduino to the MCP4725. The MCP4725 is a digital-to-analog converter (DAC) that allows modulating specific voltages directly from the Arduino. 

![Connecting the MCP4725 DAC to the Arduino](https://schollz.com/img/s1/arduino.png)

The `OUT` from the MCP4725 should go to the RED wire you connected to the cassette player. Then attach the ground wire on the cassette player to the ground on the Arduino.

The code for the Arduino just communicates via a Serial port to send voltages.

Here is the code:


```c
#include <Wire.h>
#include <Adafruit_MCP4725.h>

Adafruit_MCP4725 dac;
String sdata = ""; // Initialised to nothing.
bool started = false;
void setup(void) {
  Serial.begin(9600);

  // For Adafruit MCP4725A1 the address is 0x62 (default) or 0x63 (ADDR pin tied to VCC)
  // For MCP4725A0 the address is 0x60 or 0x61
  // For MCP4725A2 the address is 0x64 or 0x65
  dac.begin(0x62);
  pinMode(2, OUTPUT);
  pinMode(3, OUTPUT);

  Serial.println("Begin");
}

void loop(void) {
  if (started == false) {
    started = true;
    dac.setVoltage(0, 1);
    digitalWrite(2, LOW);
    digitalWrite(3, LOW);
  }
  byte ch;
  if (Serial.available()) {
    ch = Serial.read();
    sdata += (char)ch;
    if (ch == '\n') {
      sdata.trim();
      if (sdata.indexOf("voltage") > -1) {
        sdata.remove(0, 7);
        float newVal = sdata.toFloat();
        // set voltage
        float newVoltage = round(910.0 * newVal);
        if (newVoltage > 4095) {
          newVoltage = 4095;
        }
        uint16_t newVolts = uint16_t(newVoltage);
        dac.setVoltage(newVolts, 1);
        Serial.print("volts: ");
        Serial.println(newVolts);
      } else {
        Serial.println("?");
      }
      sdata = "";
    }
  }
}
```

To communicate with the Arduino you can use a simple server that hooks the MIDI to the voltage Serial. You can get this code from https://github.com/schollz/cassettesynthesizer. Make sure you have Golang installed (install [here](https://golang.org/dl/)).

```bash
$ git clone https://github.com/schollz/cassettesynthesizer
$ cd cassettesynthesizer
$ go build  
$ ./cassettesynthesizer -com ARDUINOCOM
```

Now you can open up Chrome to `localhost:8080` and you'll be able to connect a MIDI keyboard and send voltages to the Arduino. Make sure to edit the voltage map to tune each note of the cassette synthesizer, in `index.html`:

```javascript
var voltageMap = {
    "C": 0,
    "C#": 0.7,
    "D": 0.9,
    "D#": 1.2,
    "E": 1.4,
    "F": 1.62,
    "F#": 1.85,
    "G": 2.25,
    "G#": 2.6,
    "A": 3.0,
    "A#": 0,
    "B": 0,
}
```

Those are the voltages that I mapped out for my particular cassette player.

## That's it!

To get it going, start the serial server. Plug in a midi keyboard. Open chrome to the `localhost:8080`. Turn on the cassette player and **start jamming!**

If something is unclear (it probably is) don't hesitate to reach out to me. I'm [yakcar @ twitter](https://twitter.com/yakczar) and [infinitedigits @ instagram](https://instagram.com/infinitedigits).