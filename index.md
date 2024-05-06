---
layout: page
title: About me
permalink: /index.html
---

Hi! I'm Szymon. I am a Computer Scientist currently working on [Xsuite](https://xsuite.readthedocs.io/en/latest/) at the European Organization for Nuclear Research (CERN). Previously, I wrote my Theoretical Computer Science PhD thesis [on the Billaud Conjecture](https://repository.lboro.ac.uk/articles/thesis/On_the_Billaud_Conjecture_and_related_problems/21390837) at Loughborough University, UK. During my undergraduate studies I completed a Technical Studentship at CERN working on [Inspire](https://inspirehep.net).

My interests include theoretical computer science and all things related to good software engineering. Have a look at my [portfolio](/portfolio.html) or [GitHub](https://github.com/szymonlopaciuk) to see some of my past projects, and if you would like to know more about me, [this is my LinkedIn profile](https://www.linkedin.com/in/szymonlopaciuk/).

If you would like to get in touch, drop me an email at <span id="magic">szymon&#8203;@&#8203;<span style="display:none">qwerty</span>lopaciuk&#8203;.&#8203;eu</span> or a message on Matrix at [@sz_lop:matrix.org](https://matrix.to/#/@sz_lop:matrix.org).

<script type="text/javascript">
  function getmagic()
  {
    var magic = "mai";
    magic += "lto:Szymon%20Łopaciuk%20";
    magic += encodeURIComponent(String.fromCharCode(60,115,122,121,109,111,110,64,108,111,112,97,99,105,117,107,46,101,117,62));
    return magic;
  }
  var e = document.getElementById('magic');
  if (e != undefined)
  {
    var node = document.createElement('a');
    node.href = 'javascript:window.location=getmagic();';
    node.innerHTML = e.innerHTML;
    e.replaceWith(node);
  }
</script>
