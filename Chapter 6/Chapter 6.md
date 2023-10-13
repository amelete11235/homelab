# Chapter 6:
##### *Friday, October 13th, 2023*
* Using a WMI Subscription to Establish Persistence
---
## Intro

In this chapter, I'll be continuing my investigation of the ISS Playlist website. Read Chapter 5 if you want to start from the beginning.

## Using a WMI Subscription to Establish Persistence

``` powershell
PS C:\Windows\system32> net user /add notsus Muchsecure1!       
net user /add notsus Verysecure1!
The command completed successfully.

PS C:\Windows\system32> net localgroup administrators /add notsus
net localgroup administrators /add notsus
The command completed successfully.
```

I created an extra admin account on the compromised web server to act as a primary method of persistence.

![](a/978487a992408557b3eeeac831f3ea19.png)

I can log in later using rpcclient or smbclient, but I'd like to use a method that's a little harder to find than by typing `Get-LocalUser`. I'll try Metasploit's `exploit/windows/local/wmi_persistence`:

![](a/cb86480512681dd818c9784ef1feafef.png)

I had some trouble for a while figuring out how to get the `wmi_persistence` exploit to run. I'm not sure how to background a meterpreter session while still in Powershell...

![](a/c39ae723dff35b430399019e35def03e.png)

![](a/40b7f2b569a119d9a244fca67beb0494.png)

![](a/7c0e707808c53666ed45f978c0ea989a.png)

![](a/f0b65066a7f31d1920f819090aa0af73.png)

First I tried the above, where I started a Powershell process that had SYSTEM privileges, hiding it with -H, then migrating to it. Sadly, it didn't:

![](a/38a89a85ec386e70bde7aa5d3c2d692c.png)

![](a/7195aa2dafa0cc2f67d3ef5c2504a536.png)

I bet I fixed it!

![](a/0e0918b59c8ccdd8e93015f9934c8631.png)

I think I'm going to persist in a different direction now...

![](a/cdd6b870929cad94e3839146d5b9ac2f.gif)

After some research, I found out that some modules may be incompatible with `wmi_persistence`. I'll try using a different module for the reverse shell.

![](a/e386323908d31e5e7e996d7d23255944.png)

![](a/89ee0e945a035977f2e8b5afdc011233.gif)

I switched to the more basic `exploit/windows/meterpreter/reverse_tcp` module and it worked!!! 

The `wmi_persistence` module works by creating a WMI `Windows Management Instrumentation` subscription to a windows event. The default event used by the `wmi_persistence` module is a failed login to an account named "BOB". You can choose any account name that you want, or even use an entirely different event ID. I used the default event, and switched the username to "Guest". Once the target machine detects the security log event ID `4625 Failed Logon` for the account "Guest," it will send a reverse shell to my waiting listener.

![](a/564bc8ffbe03db495d69716956fbea12.png)

![](a/f937d561e3cdbd88e8d0884d34d55e30.gif)

![](a/95e409aef0ebab172155d0ea1b68d7f2.png)

![](a/c4fc421105d99a28d76cf2582236cf19.gif)

# Happy Friday, and see you next time!