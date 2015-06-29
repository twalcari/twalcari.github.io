---
layout: post
title: "URL Handler as an alternative to Java Webstart"
comments: true
date: "2015-06-26"
---

Up until now, [jFed](http://jfed.iminds.be) used Java Webstart to integrate
into websites. Now that Google Chrome has disabled the Java-plugin, I started
looking into alternatives. After looking into how some popular applications
solved this problem, it became clear that registering an URL handler is the
most popular choice.

In this post, I will show how you can register an URL handler on Windows, Mac
and Linux; and how you must process it in Java.

##Registering URL handlers

###Microsoft Windows

Registering an URL handler on Windows is done by adding some keys and values in
the Windows Register.

Detailed information can be found on
<https://msdn.microsoft.com/en-us/library/aa767914(v=vs.85).aspx>


These are the registry entries that have to be added to register the
 URI-scheme `alert`:
{% highlight bash %}
HKEY_CLASSES_ROOT
 alert
    (Default) = "URL:Alert Protocol"
    URL Protocol = ""
    DefaultIcon
       (Default) = "alert.exe,1"
    shell
       open
          command
             (Default) = "C:\Program Files\Alert\alert.exe" "%1"
{% endhighlight %}

###Mac OS X

Registering an URL handler on Mac is done by adding some keys and values into
the `Info.plist`-file.

Detailed information can be found in the [Mac Developer Library](https://developer.apple.com/library/mac/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-102207).

This are the entries which have to be added to register the URI-scheme `local`:
{% highlight xml %}
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>Local File</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>local</string>
        </array>
    </dict>
</array>
{% endhighlight %}

###Linux

Most modern disto's support `.desktop`-files, as standardized by
<http://www.freedesktop.org>. By adding a line
`MimeType=x-scheme-handler/<your-app-uri>` to the `.desktop`-file of your
application, you can link `<your-app-uri>` to your location.
To register this link in the OS, you must run the script `update-desktop-database`
after creating and/or updating the file.

For jFed we added the following post-install script to our packages:

{% highlight bash %}
#!/bin/sh

echo "#!/usr/bin/env xdg-open
[Desktop Entry]
Type=Application
Name=jFed Experimenter Toolkit
Exec=/bin/sh "/opt/jFed/jFed-Experimenter %u"
Icon=/opt/jFed/.install4j/jFed-Experimenter.png
Categories=Development
MimeType=x-scheme-handler/jfed" >> /usr/share/applications/jFed-Experimenter.desktop

update-desktop-database > /dev/null 2>&1
{% endhighlight %}

##Catching URI's in Java

Now that we've registered our URI handler on the OS, we have to add the necessary
logic to catch these URI's in your Java application.

When an URI is launched on Windows and Linux, a new instance of your application
is started with the URI as only argument.

On Mac OS X however, the URI is sent via an event. You can catch these events
with the `com.apple.eawt.OpenURIHandler`-class of the
[Apple Java Extensions](http://developer.apple.com/mac/library/samplecode/AppleJavaExtensions/).
The Apple Java Extensions-classes are only available in the JDK of Mac OSX.
To prevent your application build to fail on other platform,
you can use a library that contains stubs for all these Apple-specific classes.
Apple itself provides a library, but only an ancient version (1.4) is available on
Maven Central. Luckily other developers have already
[solved this for us](http://ymasory.github.io/OrangeExtensions/), and
provided their own stubs. For jFed the following library is used:

{% highlight xml %}
<dependency>
    <groupId>com.yuvimasory</groupId>
    <artifactId>orange-extensions</artifactId>
    <version>1.3.0</version>
</dependency>
{% endhighlight %}

You can then catch these URI-events as follows:

{% highlight java %}
import com.apple.eawt.Application;

public class AppleEvents implements com.apple.eawt.OpenURIHandler {

  public AppleEvents(){  
    //only register if we are on a Mac system. Otherwise, we will encounter NullPointerExceptions!
    if(System.getProperty("os.name").indexOf("mac") > 0){
      com.apple.eawt.Application app = com.apple.eawt.Application.getApplication();
      app.setOpenURIHandler(this);
    }
  }

  public void openURI(OpenURIEvent e) {
    System.out.println("Received URI: " + e.getURI());
  }
}

{% endhighlight %}

Typically, you receive the event in a few microseconds after registering the
handler. Subsequent URI-clicks are delivered to the same instance of your
application. This introduces some additional complexity if you want to achieve
the same behavior on all platforms. As starting a new JVM from an existing JVM
is a non-trivial task (certainly if your application is typically started with
  non-default VM arguments), the easiest way to tackle this is trying to handle
all the URI-events in one instance. You can use [JUnique](http://www.sauronsoftware.it/projects/junique/)
to check if an instance is running, and to pass the URI to the instance of your
application that is already active.
