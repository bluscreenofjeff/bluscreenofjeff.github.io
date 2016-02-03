---
layout: post
title: Semi-Persistence
category:
author: bluscreenofjeff
year: 2014
month: 6
day: 26
published: true
summary: How to get some semi-persistence using VBScript
image: 
---
     
<div class="row">  
     <div class="span9 columns">
<p>Persistence is a great thing to have on a pentest, especially if testing from the outside. Persistence also seems to be a word that makes clients' hairs on the back of their necks stand up. Backdoors are a scary thing if you're in charge of keeping a network secure.

<p>When it comes to installing persistence, there are quite a few options available such as Metasploit's persistence module or <code>scheduleme.rb</code>. These modules leave a permanent, like one would think, payload and schedule for phoning home to the tester's box. But what about when this is out of scope? 

<p>Here's a method to keep a payload phoning home but leaves no permanent effects. 

<p>Using WinRAR, we can pack our payload with a vbscript that sleeps for 60 minutes and then restarts the payload. 
With WinRAR installed, right click your favorite payload and select WinRar > Add to archive…
The window below will pop up. Select Create SFX archive and keep the Archive format set to RAR. You should also give your payload wrapper an inconspicuous name.

<br>
<center><img src="{{site.url}}/assets/semipersistence1.png" style="height:400px"></center><br><br>

<p>Now duplicate the payload that you just right-clicked and call it something different, it's renamed to <code>persistence_payload.exe</code> here. This will be the secondary payload that is run every 60-ish minutes. Why not just use the one payload? Just in case you can't migrate of off the original process, a second one is used for the persistence for safety.


<p>Now create a text file named <code>persistence_script.vbs</code> and paste the following within it:

<script src="https://gist.github.com/bluscreenofjeff/ca1dee608b253b780ea5.js">
</script>

<br><br>


<p>Change the <code>persistence_payload.exe</code> line to match your persistent payload's name. After saving this, right click the payload wrapper and select WinRAR > Open with WinRAR. Then drag the persistent payload and <code>persistence_script.vbs</code> into WinRAR. You should end up with this:

<br><center><img src="{{site.url}}/assets/semipersistence2.png" style="width:90%"></center><br><br>

<p>Almost there, but we need this executable to do something when the victim clicks on it. Click the SFX button in the ribbon. Then under the SFX tab, select Advanced SFX Options. 
Now we have a number of options we can set, such as having this executable extract to a specified folder (<code>C:/Temp</code> maybe?), set the Overwrite mode to Overwrite all files (Update tab), and tell WinRAR what to run upon extraction.
After setting the first two options, open the Setup tab and in the Run after extraction box, enter the name of your normal payload (<code>payload.exe</code> here) and the VBScript (<code>persistence_script.vbs</code>). 
<br><center><img src="{{site.url}}/assets/semipersistence3.png" style="width:325px"></center><br><br>



<p>Now when the payload is run, it will run as expected (by calling <code>payload.exe</code>) and in Task Manager, a <code>wscript.exe</code> process will run waiting to re-rerun <code>persistence_payload.exe</code> every 60-ish minutes. Before re-running the payload, it will kill the process if it's still running so if you can't migrate you will lose that session -- best to keep it as a backup in those instances. 

<p>Restarts and logoffs will kill <code>wscript.exe</code> resulting in the persistence being lost-- no worries about a permanent backdoor. This method can also be used to launch the initial payload multiple times when first run, if you experience flakiness issues with payloads.




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
