+++
title = 'Project Rhythm'
date = 2023-10-22T23:08:43-07:00
draft = false
+++


## Processing grief

Last year, my beloved puppy, Kensi, who was only four years old, passed away due to a medical condition known as Gastric Dilatation-Volvulus (GDV). Feeding Kensi was always a bit of a challenge; she had a voracious appetite and would eagerly consume her meals in mere seconds. Despite our attempts to break her feedings into smaller portions, our busy family often found it difficult to keep track of what had been fed and what was left. This sometimes resulted in accidental overfeeding, as Kensi never seemed to resist the temptation of food. It's been said that some dogs lack the ability to recognize when they've had enough.

![Kensi](/images/IMG_4761_Original.jpg)

## Redirecting grief - seeking a solution

In my desire to address this issue for future dog parents, I began searching for a smartphone app that could serve as a reminder system. However, a noisy alarm that we'd simply dismiss wasn't the ideal solution. I envisioned creating a "visual alarm" that would be unobtrusive yet noticeable to everyone in the household. My goal was to avoid the complexities of developing an iPhone or Android app and spare my family from the constant presence of a smartphone.

## The idea in simple terms
 - A hardware gadget with bright LEDs
 - Usable without a paired phone or special accounts
 - Programmable to keep the feeding schedule
 - Use LED colors to indicate delays, missed feedings

## Choice of hardware
### Why Arduino?

I was looking for a low-power single-board computer and this Arduino from Adafruit appeared to be the perfect choice for my needs.

It has several built-in features, including Bluetooth (BLE), 10 LEDs (any color), and connectors for a battery power supply. Everything except a real-time clock is already builtin.

To make things even more convenient, there's an OEM case readily available that offers easy access to the device's buttons. In fact, if I didn't need to attach an RTC board (more about that later), I wouldn't have found it necessary to construct my custom case.

![Circuit Bluefruit](/images/4333-11.jpg?width=20px&height=60px "Circuit Bluefruit")
![Circuit Bluefruit](/images/3915-00.jpg?width=20px&height=60px "Bluefruit Enclosure")


## Choice of software

I came across TinyGo, a variant of Go, a language that appears better suited than using C. Go has a few high level language features that would help me avoid the usual struggle with C. Although the Circuit Bluefruit can run Python, I had reservations about the code being exposed to anyone who connected it as a USB drive. Opting for Go also presented an opportunity for me to delve into a new programming language.

TinyGo has many features of the parent language Go, but there are a few differences.

## Initial challenge

### Lack of builtin Clock

We take it for granted that we can get the current year, month, day, and time accurately from the laptop anytime. I learnt that computers have what is called a [real time clock](https://en.wikipedia.org/wiki/Real-time_clock). A real time clock can keep track of time while a computer is powered on and running. However once the laptop is powered off and restarted, the initial time needs to be set again.

### Where do I find the time to set it?

If our code needs to give the initial time to the clock, we need to know the time to begin with. But we can't call our real-time-clock, because it doesn't know the time yet! 

It appears time is given out to everyone by [NTP servers](https://www.ntppool.org/en/), keepers of time for everyone on the internet. But where do they get their time? Apparently from many physical sources, some of which include GPS satelites. However that is too complicated to delve into now.

In order for us to connect to NTP servers, we need internet or a wifi connection. We need to add another breakout board like the below.

![Adafruit HUZZAH ESP8266 Breakout](/images/2471-06.jpg?width=20px&height=60px "Circuit Bluefruit")

### Making do without an additional Wifi Chip (Save $10)

We need the current time only when manage a schedule (feeding timings and feeding count for my puppy), so it makes sense to send the current time along with the schedule via Bluetooth

Moreover, wifi requires another setup of SSID and Password via Bluetooth. Adding a wifi chip just to get time information from NTP is an overkill.

### Retaining time during power off

I bought a RTC DS3231 from Amazon. It takes a rechargeable LR2032 as power backup for DS3231. However it seems unreliable in keeping time when powered off. For all practical purposes, I am going to keep this connected to wall power as much as possible.

![DS3231 Connection](/images/IMG_1474.jpg)

## How bluetooth works from TinyGo and Go

### Peripheral (Arduino) adveritizes - uses TinyGo

    err = bleAdapter.Enable()
    if err != nil {
      ...
    }

    adv := bleAdapter.DefaultAdvertisement()
    err = adv.Configure(bluetooth.AdvertisementOptions{
      LocalName: "Rhythm-",
      // Save power by advertizing only once every second
      // default is every 150 ms. 6 times less often
      Interval: bluetooth.NewDuration(time.Second * 1),
    })
    if err != nil {
      ...
    }
    err = adv.Start()
    if err != nil {
      ...
    }

### Central (My Mac) scans - uses Go

The following Go code, running on Mac, will look for any device that advertizes a name *starting with* **Rhythm**. We have to assume there will be multiple 'pet feeder / reminder' devices in the house and need the ability to identity one from the other. For now we assume that they will be named "Rhythm-1", "Rhythm-2" etc. For simplicity, we only connect to the first detected Rhythm device.


    // Start scanning.
    devicePrefix := "Rhythm-"
    err = adapter.Scan(func(adapter *bluetooth.Adapter, result bluetooth.ScanResult) {
      _, loaded := foundDevices.LoadOrStore(result.Address.String(), result.LocalName())
      if !loaded {
        println("found device:", result.Address.String(), result.RSSI, result.LocalName())
        ln := result.LocalName()
        if len(ln) >= len(devicePrefix) && ln[:len(devicePrefix)] == devicePrefix { // allow trailing stuff
          adapter.StopScan()
          ch <- result
        }
      }
    })

### How scanning appears in real life

![Scan Results](/images/scan_results.png)

## How to send time & schedule

We can only send roughly 20 bytes in each BLE message once connected. Although I can send a series of messages one after the other, let us keep it simple and fit everything in a single message.

First two bytes are magic - so just we won't interpret accidental random input as a schedule. We will only accept messages that start with 0xCD followed by another number.

    0: Byte 1   0xCD  // A magic byte, short for (C)omman(d)
    1: Byte 2   0x01  // First command?

Next 4 bytes will be time, represented as a [Unix time](https://en.wikipedia.org/wiki/Unix_time). However we will subtract or add appropriate offset to convert it to local time (from GMT/UTC). This is because I couldn't figure out how to set a timezone for Arduino's RTC clock. But this method of adding offset from Central works for now.

    2: Byte 3   101 // 4 bytes together represents
    3: Byte 4   52  // a time on Oct 22nd, 2023
    4: Byte 5   218
    5: Byte 6   8

Remaining bytes will a sequence of bytes where every 3 bytes repesent 'Hour-Of-Day', 'Minute-Of-day', 'Number-of-Feedings' followed by the next schedule. We can probably fit 3 or 4 timings in a single message (whatever remains in 20 after timestamp and magic bytes)
 
    6:  Byte   7   8   // At 8
    7:  Byte   8   15  // :15 AM
    8:  Byte   9   2   // 2 feedings

    9:  Byte  10   12  // At 12
    10: Byte  11   45  // :45 PM (Noon)
    11: Byte  12   1   // 1 feeding

    12: Byte  13   6   // At 6
    13: Byte  14   45  // :45 PM
    14: Byte  15   2   // 2 feedings


## Correcting time and date

Once Rhythm receives time, it should correct its internal clock (DS3231), becuase the Mac/PC always has the most accurate time.

	for t := range timeChan {     // Time received
		realTime := time.Unix(t, 0)
		dt, err := rtc.ReadTime()   // what we have...
		if err != nil {
          ...
		} else {
			date := time.Date(
                      realTime.Year(),
                      realTime.Month(),
                      realTime.Day(),
                      realTime.Hour(),
                      realTime.Minute(),
                      realTime.Second(), 
                      0, time.UTC)
			err := rtc.SetTime(date)  // correct it..
			if err != nil {
				println("Error configuring RTC")
			}
		}
	}

## Coding a wakeup alarm

	tick := time.NewTicker(time.Second * 15) // check every 15 seconds

	for {
		select {
		case <-tick.C:
			dt, _ := rtc.ReadTime()
			h := dt.Hour()
			m := dt.Minute()

			// check each of the alarms (10 max)
			for _, sched := range currentSchedule {
				if sched.hourOfDay == h && sched.minuteOfHour == m {
						println("Alarm manager: WAKEY WAKEY", h, m)
						// light up the LEDs to appropriate color
						

## Laser-cut enclosure from school, power options

### Cardboard version

![Cardboard version](/images/IMG_1504.JPG)

![Cardboard version](/images/IMG_1505.JPG)

### Wood version of enclosure

Here is the online design of the wooden prototype (built via Onshape).

![Onshape design](/images/Onshape1.png)

![Onshape design](/images/Onshape2.png)


Below is the laser cutter I used for the prototype from the LGHS Robotics Lab. 

![Laser cutter at LGHS](/images/IMG_8606.jpg)

![Laser cutter at LGHS](/images/IMG_8607.jpg)


![Wood version](/images/IMG_1508.JPG)

### Working demo

Sunny, my new pup, waiting for his food.

![Demo Shot](/images/IMG_1611.JPG)

## Challenges faced

### Concurrent programming is new to me

Only one thread/goroutine can access the real-time clock at the same time. Read it from multiple places might result in corrupted time information as shown below

    Date (#257) 2023 10 22 20 46 49
    Date (#257) 2023 10 22 20 46 54
    Date (#298) 1999 1 1 1 0 0
    Date (#257) 2023 10 22 20 46 59

### Special handling required for TinyGo Bluetooth interface handlers

    Alarm manager: Received new schedule [12/16]0x20007a40
    Received 15 bytes
    Received time as 1698008151
    panic: runtime error at 0x00028b99: heap alloc in interrupt
    [tinygo: panic at /usr/local/Cellar/tinygo/0.30.0/src/runtime/slice.go:30:15]

**heap alloc in interrupt** - methods we register with bluetooth to process incoming data cannot allocate memory. 

Solution: We will do minimal process in the handlers and pass it via channel to other goroutines.

# Future - Manufacturing
 - TODO: Research vendors
 - TODO: Cost of small batch production
 - TODO: Explore https://easyeda.com/
 - TODO: Simpler design that reduces cost

