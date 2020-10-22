---
layout: post
title: Installing Splunk from the Windows Command Line
---
Windows is pointy clicky, how do I command line?!?!

## Introduction:

Have you ever been in a situation where you had to install a Splunk agent on Windows from a shell?  Well I have, and let me tell you how I did it.  Sounds simple, right? Well it was, but when you’re not used to working on Windows from a shell (like a red teamer or a pen tester might be), it can be frustrating and time consuming.  Also remember that shells vary.  If you’re a pen tester, your way in might have a very limited shell.  If you’re a blue teamer, maybe you don’t ever work on Windows from a shell and/or your EDR only gives you a crappy shell.

## Assumptions:

For the purposes of this little guide, we are presuming you have a basic shell on the Windows host and some sort of Splunk instance.  Keep in mind if you don’t have an elevated user (preferably system) this will limit the logs you can collect: the user Splunk runs as limits what logs you can collect.  In my example, I was doing this for blue team reasons and could install Splunk as system, so I had no issues collecting logs.  You can still follow this for evil red team reasons to exfil data that you have access to.

## Uno: Get Splunk

We’ll need a copy of the forwarder to install.  Splunk keeps older versions of the forwarder available, so make sure to click “older releases” and find the version of the forwarder that mis compatible with your indexer as running a newer forwarder with an older version indexer may cause issues.  [Splunk Universal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html)

## Dos: Install Splunk

This part is the part that inspired me to write this.  It took way too long how to do this from the command line.  Linux?  Yeah, install is pretty easy: untar stuff.  But Windows MSI/EXE is made to click and run, so this took a lot of reading to get the commands just right on a limited shell.

However you can, get the forwarder on the target.  EDR makes it easy to get and put files on the client, but for red teamers, certutil, powershell IEX downloadstring, wget, iwr… you get the picture.

Install the Splunk agent: 
> Msiexec.exe /i splunkagentname.msi AGREETOLICENSE=YES SPLUNKPASSWORD=PASSWORD1 /q

Let’s walk through the above.  “/i” is install, “AGREETOLICENSE” will cause a popup if you don’t specify yes, which your shell may not support, and after Splunk 7.0 or so, they force you to immediately change the admin account password.  If you change the password as part of the install, you won’t get (or miss) the popup.  “/q” means quiet, so you won’t cause any popups for any logged in users.

BONUS: You can also specify the install path if you’re trying to sneaky with “TargetDIR” to install it somewhere else than the default location, which is “Program Files”.

## Tres: Install your apps:

Great, we’ve got Splunk.  But what about apps?  What about a Windows TA?  A Splunk Cloud credentials app?  A custom app?

Using the credentials you made in the previous app, we’ll authenticate to Splunk and tell it to install our app.  From the Splunk bin directory:

> Splunk.exe install app /path_to_app_to_install.whatever

Splunk will install your app in splunk/etc/apps, but you still need to restart Splunk in order for Splunk to use the app.  From /bin:

> splunk.exe restart

Sometimes Splunk doesn’t actually restart.  Check it with “splunk status” and if Splunk won’t restart, you may have to force the service to start:

> Net start splunkforwarder service

## Quatro: Configure Your Apps

You’ve got Splunk installed, you’ve got apps, awesome!  But with a deployment server, you still haven’t don’t anything custom.  For prebuilt apps, go to etc/apps/yourapp/default and grab a copy of inputs.conf.  Make the changes you need (like log location, Splunk index you want it in, etc – see [Inputs.conf](https://docs.splunk.com/Documentation/Splunk/8.0.6/Admin/Inputsconf) for details.  This post isn’t to teach you how Splunk works, but the main thing you need is "[monitor://path/to/file.log]”.  Now take that copy of inputs.conf you need and put it in the local folder of your app.  

Restart Splunk like you did above.

## Cinco: Profit

Hopefully following along with this, it took you less than the 2 hours it took me.   
