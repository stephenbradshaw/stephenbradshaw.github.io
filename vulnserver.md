---
layout: page
title: Vulnserver
sidebar_link: true
redirect_from: /p/vulnserver.html
---

Originally introduced [here](/2010/12/15/introducing-vulnserver.html), Vulnserver is a Windows based threaded TCP server application that is designed to be exploited.  The program is intended to be used as a learning tool to teach about the process of software exploitation, as well as a good victim program for testing new exploitation techniques and shellcode.

# Whats included?

The github repository includes the usual explanatory text files, source code for the application as well as pre compiled binaries for vulnserver.exe and its companion dll file.

# Running vulnserver

To run vulnserver, make sure the companion dll file essfunc.dll is somewhere within the systems dll path (keeping it in the same directory as vulnserver.exe is usually sufficient), and simply open the vulnserver.exe executable.  The program will start listening by default on port 9999 - if you want to use another port just supply the port number as a command line option to the program - e.g. to listen on port 6666 run vulnserver.exe like so:

    vulnserver.exe 6666

The program supports no other command line options.

# What should I be aware of?

The program will spit out its version number when you start it up, as well as the version number of its companion dll, so it's obvious what version you are running just in case I need to update it in future. Exploitation can be a finicky business, and changes to/recompilation of a program can change the buffer structure required to gain control of code execution, so I have made tried to make it as easy as possible to determine what version of the program and its associated dll you are running, so if you are following along with any guide you can ensure you have the same version of the application as used in the guide.  This is also something to be aware of if you want to compile the program yourself - different c compilers (even different versions of the same compiler) can produce binaries that exploit in different ways from the same code, and you will get a warning about this when you start the program too.

Vulnserver doesn't actually do anything other than allow exploitation - there is no useful functionality.  Integrating code to perform some other function didn’t seem to be a good use of binary space or my time considering the purpose of the program.  There has been some effort put in to make it appear as though the program is taking user input and providing responses, so it seems like a regular (albeit very basic) server application, but there's really no reason to run this unless you're actively exploiting it at the time.

You SHOULD NOT run this program on any critical system, and you should not allow this program's listening port to be accessible from any untrusted network, such as the Internet.  There is no malicious code included within this program (check the source to confirm), so this is NOT a virus or malware, but this program can be put to malicious use if an untrusted individual can access it.  That's kind of unavoidable considering what the program was designed for.  So run this only on a well protected (possibly isolated) test system, and only when you are actively using it to test exploitation methods - don’t just leave it running all the time.

Remember that if vulnserver is running on your system, and you're not exploiting it, someone else might be.


# Exploiting Vulnserver

I have written a number of articles about how to go about exploiting the vulnerabilities in Vulnserver, which you can find by checking this blog for posts tagged [vulnserver](/tags.html#vulnserver).

# Download

Download from the github repository [here](https://github.com/stephenbradshaw/vulnserver)