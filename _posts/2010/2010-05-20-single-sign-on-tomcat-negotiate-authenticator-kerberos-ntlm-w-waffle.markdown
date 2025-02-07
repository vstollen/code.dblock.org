---
layout: post
title: "Single Sign-On: Tomcat Negotiate Authenticator (Kerberos + NTLM) w/ Waffle"
redirect_from: "/single-sign-on-tomcat-negotiate-authenticator-kerberos-ntlm-w-waffle/"
date: 2010-05-20 00:32:22
tags: [tomcat, waffle, java, active directory]
comments: true
dblog_post_id: 103
---
![]({{ site.url }}/images/posts/2010/2010-05-20-single-sign-on-tomcat-negotiate-authenticator-kerberos-ntlm-w-waffle/image_12.jpg)

I’ve added a Tomcat Negotiate (Kerberos + NTLM) authenticator to [Waffle](https://github.com/dblock/waffle) 1.3 for Tomcat 6. Here’s how to use it.

#### Download

Download [Waffle 1.3](https://github.com/dblock/waffle/). The zip contains _Waffle.chm_ that has the latest version of this tutorial.

#### Configure Tomcat

_Copy Files_

I started with a default installation of Tomcat 6. Checked that I could start the server and navigate to https://localhost:8080. Copy the following files into tomcat’s _lib_ directory.

- _jna.jar_: Java Native Access
- _platform.jar_: JNA platform-specific API
- _waffle-jna.jar_: Tomcat Negotiate Authenticator

_Authenticator Valve_

Add a valve and a realm to the application context in your context.xml (for an application) or in server.xml (for the entire Tomcat installation).

{% highlight xml %}
<Context>
  <Valve className="waffle.apache.NegotiateAuthenticator" principalFormat="fqn" roleFormat="both" />
  <Realm className="waffle.apache.WindowsRealm" />
</Context>
{% endhighlight %}

_Security Roles_

Configure security roles in your application’s _web.xml_. The Waffle authenticator adds all user's security groups (including nested and domain groups) as roles during authentication.

{% highlight xml %}
<security-role>
  <role-name>Everyone</role-name>
</security-role>
{% endhighlight %}

_Restrict Access_

Restrict access to website resources. For example, to restrict the entire website to locally authenticated users add the following in _web.xml_.

{% highlight xml %}
<security-constraint>
  <display-name>Waffle Security Constraint</display-name>
  <web-resource-collection>
    <web-resource-name>Protected Area</web-resource-name>
    <url-pattern>/*</url-pattern>
  </web-resource-collection>
  <auth-constraint>
    <role-name>Everyone</role-name>
  </auth-constraint>
</security-constraint>
{% endhighlight %}

#### Test

Restart Tomcat and navigate to https://localhost:8080.

You should be prompted for a logon with a popup. This is because by default localhost is not in the _Intranet Zone _and the server returned a 401 Unauthorized. Internet servers with a fully qualified named are detected automatically.

_Internet Explorer_

Ensure that Integrated Windows Authentication is enabled.

1. Choose the_ Tools, Internet Options_ menu.
2. Click the _Advanced_ tab.
3. Scroll down to _Security_
4. Check _Enable Integrated Windows Authentication_.
5. Restart the browser.

The target website must be in the Intranet Zone.

1. Navigate to the website.
2. Choose the _Tools, Internet Options_ menu.
3. Click the _Local Intranet_ icon.
4. Click the _Sites_ button.
5. Check _Autmatically detect intranet network_.
6. For localhost, click _Advanced_.
7. Add _https://localhost_ to the list.

_Firefox_

1. Type _about:config_ in the address bar and hit enter.
2. Type _network.negotiate-auth.trusted-uris_ in the Filter box.
3. Put your server name as the value. If you have more than one server, you can enter them all as a comma separated list.
4. Close the tab.

Navigate to _https://localhost:8080_ after adding it to the Intranet Zone.

![]({{ site.url }}/images/posts/2010/2010-05-20-single-sign-on-tomcat-negotiate-authenticator-kerberos-ntlm-w-waffle/image_2.jpg)

You should no longer be prompted and automatically authenticated.

![]({{ site.url }}/images/posts/2010/2010-05-20-single-sign-on-tomcat-negotiate-authenticator-kerberos-ntlm-w-waffle/image_7.jpg)

#### Logs

In the logs you will see the following output for a successful logon.

```
logged in user: dblock-green\dblock (S-1-5-21-3442045183-1395134217-4167419351-1000)
 group: dblock-green\None
 group: Everyone
 group: dblock-green\HelpLibraryUpdaters
 group: dblock-green\HomeUsers
 group: BUILTIN\Administrators
 group: BUILTIN\Users
 group: NT AUTHORITY\INTERACTIVE
 group: CONSOLE LOGON
 group: NT AUTHORITY\Authenticated Users
 group: NT AUTHORITY\This Organization
 group: S-1-5-5-0-442419
 group: LOCAL
 group: NT AUTHORITY\NTLM Authentication
 group: Mandatory Label\Medium Mandatory Level
successfully logged in user: dblock-green\dblock
```

My laptop is not a member of an Active Directory domain, but you would see domain groups, including nested ones here. There’s nothing special to do for Active Directory. The authenticator also automatically handles all aspects of the Negotiate protocol, chooses Kerberos vs. NTLM and supports NTLM POST. It basically has the same effect in Tomcat as choosing Integrated Windows authentication options in IIS.

#### Related Projects

- [Tomcat SPNEGO by Dominique Guerrin](https://web.archive.org/web/20120114182927/https://tomcatspnego.codeplex.com/): this is a very good prototype of a filter. It uses JNI and not JNA, doesn’t support NTLM POST and the code is pretty thick.
- [SPNEGO Sourceforge](https://spnego.sourceforge.net/): it’s a nightmare to configure, doesn’t work without an Active Directory domain and requires an SPN
- [JCIFS NTLM](https://web.archive.org/web/20130117024232/https://jcifs.samba.org/src/docs/ntlmhttpauth.html): no longer supported and they recommend using Jespa
- [Jespa](https://www.ioplex.com/jespa.html): a commercial implementation that claims to do the same thing as Waffle, but uses the Netlogon service instead of the native Windows API
