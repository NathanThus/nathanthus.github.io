---
title: Interacting With VISACOM
tags: [Embedded, C++, C, MATLAB, Learning]
style: fill
color: primary
description: Learning about communicating with scientific equipment over VISACOM.
---

# Learning VISACOM

So, a bit of context is required here. After obtaining my sensors, I wanted to see what the exact characteristics were. The joystick doesn't actually have a proper datasheet, detailing it's behaviour under load. While I could fairly assume it's two potentiometers slapped together, it's better to measure.

For this, I planned to use the oscilloscope we have at Fontys, at 3.3V and 5V to see how it behaves. Once I got to the oscilloscope however, I noticed that there's no easy way to actually grab the data off the oscilloscope directly into MATLAB. Originally, I intended to just import the data via USB to MATLAB and plot from there.

After a bit of digging, I found a video on MATLAB detailing connecting an oscilloscope to MATLAB directly, allowing me to easily import data. That sounds great, but it's from 2008 and the relevant API has been deprecated. Not so great.

With that bit of wind taken out of my sail, I decided to peek at the new API. With a fair few installations, including **IVI Shared Component**, **Keysight EDUX 1000X Series Driver**, **Instrument Toolbox for MATLAB**, **Keysight instrument Toolbox for MATLAB**, relevant support packages and VISA drivers (this genuinely took about half a day), I had all the relevant tools for developing a MATLAB script to connect to the oscilloscope!

## MATLAB Shenanigans

After doing some digging, I found that once connected, you can grab the USB Address of your scope with the following command:

``` matlab
availableResources = resources(scope)
```

This in turn allows me to dynamically grab the address for the scope. Once I managed to grab that, there was a slight hiccup as MATLAB was a bit confused as to which driver it had to use. Seeing as I only had **one** driver installed I was a bit suprised, but I pointed it to the AgInfiniiVision driver for the Keysight Scope.

```matlab
scopeResource = availableResources; % Your address goes here.

scope = oscilloscope();
scope.Resource = scopeResource;

scope.Driver = "AgInfiniiVision"; % Driver Name
```

However, here's the kicker. Despite me double checking every address, tool, install and smidge of MATLAB script, it outright refused to connect. I tested the connection via the Keysight IO Suite, and not only did it **find** the scope, I could send commands to it using **VISACOM**. Somewhere within MATLAB, the API didn't work as intended.

Now, for many this would be a sign for me to toss the towel in the ring, grab a trusty USB-drive and capture the data that way. Those people, evidently, have not met me.

## C++, my trusty old friend

So yeah, I spent about 1.5 days making my own terminal program for interacting with VISACOM, and per extension, the oscilloscope. Starting off, I spent a fair whack of time looking at potential python libraries (make it platform agnostic, as we've got a mix of macOS, Windows and Linux devices at Fontys). USBTMC seemed to be promising, but failed to deliver any actually compliable code. This was due to a change in the USB library it used in the backend, having changed a fair bit. Considering my distinct lack of python knowledge (I can do some basics, but not much beyond), I turned to my old friend C/C++.

After configuring Visual Studio for using external libraries for the first time in 2 years (and it being both painful and painless at the same time), I was off to the races. Setting up some basic conditions for managing the connection, I sent over the first command: `*IDN?` using the following function:

```cpp
int DoQueryString(const char query[]) // query: {'*','I','D','N','?'}
{
	char message[80] = { 0 };
	strcpy_s(message, query);
	strcat_s(message, "\n");
	visaStatus = viPrintf(scope, message);

	QuerySleep(QUERY_DELAY);

	if (visaStatus != VI_SUCCESS)
	{
		std::cout << "Failed in sending phase!" << std::endl;
		return -1;
	}
	visaStatus = viScanf(scope, "%t", queryResultText);
	if (visaStatus != VI_SUCCESS)
	{
		std::cout << "Failed while parsing!" << std::endl;
		return -1;
	}
	return 0;
}
```

This requests a short string identifying the Model number and Creator of the scope, namely Keysight EDUX 1002G.

### Commanding Query on Deck

With this small success, I continued adding basic commands until I could auto scale the scope for increased clarity on the sinewave I was recording:

```cpp
int DoCommand(const char command[]) // command == "autoscale"
{
	char message[80] = { 0 };
	strcpy_s(message, command);
	strcat_s(message, "\n");
	visaStatus = viPrintf(scope, message);

	QuerySleep(QUERY_DELAY);

	if (visaStatus != VI_SUCCESS)
	{
		std::cout << "Failed in sending phase!" << std::endl;
		return -1;
	}
	return 0;
}
```

Actually setting the properties of the scope, did turn out to be a bit more of a hassle. While the rest of the API syntax was quite lax, the properties were incredibly pedantic by comparison. This caught me slightly off guard, but after finding out some of the more nitty-gritty details that I won't bother you with, I managed to prepare the scope, as well as being able to enable which probe you want to use remotely!

### Precious Data Points

Finally, the pièce de résistance. Actually getting the data from the scope.

With VISA actually taking care of the serial bus implementation (Though I've had to do it before in Semester 3 Technology/Embedded), I was left with ingesting the data from the bus and making sure it was parsed into a usable format.

The main difference here was using `viScanF()` instead of `viPrintf()`, requiring some more specific parameters to be handed to the function. We needed to give the array, alongside a variable to store the **actual** amount of transmitted data.

```cpp
int DoQueryIEEEBlock(const char query[])
{
	char message[80] = { 0 };
	strcpy_s(message, query);
	strcat_s(message, "\n");
	visaStatus = viPrintf(scope, message);
	QuerySleep(QUERY_DELAY);

	if (visaStatus != VI_SUCCESS)
	{
		std::cout << "Failed in sending phase!" << std::endl;
		return -1;
	}
	int dataLength = IEEEBLOCK_SPACE; // About 5MB worth of space

	visaStatus = viScanf(scope, "%#b\n", &dataLength, IEEEEBlockData);
	if (visaStatus != VI_SUCCESS)
	{
		std::cout << "Failed while parsing!" << std::endl;
		return -1;
	}

	// This gets changed in the library, and never exceeds the allocated space.
	if (dataLength == IEEEBLOCK_SPACE)
	{
		std::cout << "Not all data may have been saved!" << std::endl;
	}
	return dataLength;
}
```

Now for the slightly more keen eyed among you, you will have spotted the `%#b\n` in the `viScanf()` function. Now while `%b` in standard C ASCII means the use of binary data, in VISA it's used to signify an **array** of data. Once we've verified that there is, in fact, data available, we can get started with the reading.

I needed to quickly grab some data from the scope as well, so I knew how to read the data correctly. This was achieved by querying the system for the Origin Point, the increment scale as well as reference values for the wave form using `:WAVeform:PREamble?`. This command specifically did require exact capitalisation, for reasons I do not understand, unlike the others.

```cpp
double x_origin = queryResultNumbers[WAVEFORM_X_ORIGIN];
double x_increment = queryResultNumbers[WAVEFORM_X_INCREMENT];
double y_increment = queryResultNumbers[WAVEFORM_Y_INCREMENT];
double y_origin = queryResultNumbers[WAVEFORM_Y_ORIGIN];
double y_reference = queryResultNumbers[WAVEFORM_Y_REFERENCE];
```

After having the actual measurement data and the reference data, I could finally print it to a CSV file. With some fairly simple file handling, I wrote the file to the disk in the same directory as the executable (assuming no-one puts this in their System32 folder).

```cpp
	std::cout << "Writing data to disk." << std::endl;

	for (size_t i = 0; i < bytes; i++)
	{
		/* Write time value, voltage value with inverted order in the column. */
		fprintf(writeFilePtr, "%9f, %6f\n",
			x_origin + ((float)i * x_increment),
			((y_reference - (float)IEEEEBlockData[i]) * y_increment) - y_origin); // Inverted order in each column
	}
```

And now, the magical moment. I ran the program, having briefly hardcoded all the values I wanted to use. With a few seconds of silent delay, the oscilloscope lit up the relevant buttons like a Christmas tree, I saw the CSV file pop up in my directory.

With a simple import of the data into MATLAB, I plotted it. Struggling a bit with the roughly 20000 data points it had to churn through, it produced the exact frequency of sinewave I had setup on the scope. It worked!

#### Whoops, wrong way

After my brief moment of joy, I took another close look at the figure. It was inverted. Where the sinewave peaked on the right side of the scope, it was peaking on the left side of my figure. With a quick band-aid fix, I inverted the index. (Yes, this could have been done better, but I doubt the number of people using this will exceed **Me**, **Myself** and **I**).

```cpp
	for (size_t i = 0; i < bytes; i++)
	{
		/* Write time value, voltage value with inverted order in the column. */
		size_t reversedIndex = bytes - 1 - i;
		fprintf(writeFilePtr, "%9f, %6f\n",
			x_origin + ((float)i * x_increment),
			((y_reference - (float)IEEEEBlockData[reversedIndex]) * y_increment) - y_origin); // Inverted order in each column
	}
```

This recreated the figure exactly as it was on the oscilloscope, meaning that I had properly transferred the data over USB!

The data & figure in question:

[The data acquired from the oscillocsope](https://github.com/NathanThus/Minor-Embedded-Systems/blob/develop/Personal/Assets/VISA/data.csv)
![A Figure displaying a sine wave, obtained from a wave generator on an oscilloscope.](https://raw.githubusercontent.com/NathanThus/Minor-Embedded-Systems/refs/heads/develop/Personal/Assets/VISA/figure.png)
(*A sine wave corresponding to the generated wave on the oscilloscope*)

## Advantages over USB stick

There is one quite distinct advantage over the USB stick approach. **You don't need to mess with USB sticks.** Quite simply open the program, select the options you wish to use and **that's it**. A CSV data file is created in the executable directory, immediately available for export into MATLAB. Combine that with renaming the file (as my program does overwrite the last one), and some `openfile` commands in MATLAB, you can easily setup your MATLAB environment to use the shiny new data!

## Conclusion

Honestly, this was an insane amount of fun. I've learned about a new protocol, managed to tinker with some interesting tooling and overall, just had a blast. I'm definitely far more familiar and comfortable with the programming side of embedded systems, that's for sure. Still, I'll want to improve the other aspects of the Embedded Systems field with my main project, the Game Controller, for which I built this tool in the first place.

I might add a few more things, and improve the overall program if Fontys wants to actually use it as tooling, but that's a matter for another day.

## Copyright Notice

I do not consent for the use of this article, or any other webpage, to be used by AI for training purposes.
© 2025 Nathan Thus. All rights reserved.
