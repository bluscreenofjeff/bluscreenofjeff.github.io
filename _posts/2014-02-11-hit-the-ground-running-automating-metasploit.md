---
layout: post
title: Hit the Ground Running- Automating Metasploit
category: Metasploit
year: 2014
month: 2
day: 14
published: true
summary: Use resource scripts to run common Metasploit tasks quickly
image: post_one.jpg
---
     
<div class="row">  
     <div class="span9 columns">
     <p>There are a number of commands that tend to get run on every session on a target I get in Metasploit. Using resource files, these commands can be automated to dump as much information as possible, as quickly as possible. This can be combined with an MSFConsole autostart script to automate the starting of handlers and pre-fill options for post modules that don't need to be run on every session.</p>
     <br><p>First we make a new file for the autorunscript to be run on each new session:</p>
     <pre><code>nano /infogather </code></pre>
     <p>and paste the following in the file:</p>
     <script src="https://gist.github.com/bluscreenofjeff/9106100.js">
     </script>
     <p>Each command will be run sequentially and then background the session. Any command that can be run through a meterpreter shell can be used, with options added (for example, screenshot using the -v false option). Running the above commands will save screenshots to the default directory, <code>/root/*RANDOMNAME*.jpeg</code> and will save the output from the other gather post modules to the <code>/root/.msf4/loot</code> directory.
     It should be noted that running a large number of commands could cause sessions to drop. If you find that being the case, try removing some of the commands you don’t need output from on every session. Alternatively, this could be run up as a resource file by calling it with:</p>
     
     <pre><code>resource /infogather</code></pre>

     <br><p>Now we're all set to catch some shells. Let's make that easier by having a handler autostart, get logging all set up, and get the <code>post/windows/manage/multi_meterpreter_inject module</code> ready to spread our session to some other machines for posterity.</p>
     
     <p>Let's create the <code>/root/.msf4/msfconsole.rc</code> file that MSFConsole will automatically run with </p>

     <pre><code>nano /root/.msf4/msfconsole.rc</code></pre>
     <p>and paste the following in the file:</p>
     <script src="https://gist.github.com/bluscreenofjeff/9106047.js"></script>
     <p>Similar to the autosploit resource file, any command normally entered into the msf prompt can be used-- enabling you to set up any post module you want. Don't forget to replace the Xs above with the proper IP data. If you're using the <code>multi_meterpreter_inject</code>, you can add multiple addresses in a semicolon-separated list to spread the meterpreter session to numerous boxes for penetration.</p>
     <br><p>Finally, we’ll create a resource file to kill and restart a listener-- good for when you’re on a social engineering call and the just isn’t quite coming in. </p>
     <pre><code>nano /bounce</code></pre>
     <p>then paste the following into the file:</p>
     <script src="https://gist.github.com/bluscreenofjeff/9106071.js">
     </script>
     <p>Now we can call the resource file with the following command from the msf prompt:</p>
     <pre><code>resource /bounce</code></pre>
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