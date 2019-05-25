---
title: "Hacker News in your terminal!"
date: 2013-10-25T00:00:00+01:00
---
If you're anything like me, you visit Hacker news at least 4 times a day and you spend a lot of time in the terminal.

I usually open the Hacker News site in my browser and decide if any of the headlines catch my attention, I thought it would be cool if I could see a summary of the new featured posts in my terminal and only open them in a browser if I wanted to read them, so I made this simple tool.

![Preview](http://s3-eu-west-1.amazonaws.com/matt-github/hn.png)

##Installation

Before installation you will need node/npm installed.

<pre><code class="language-markup">sudo npm install -g hacker-news-cli</code></pre>

<br>

##Usage
The global installation will symlink an executable script and place it in your PATH. To use simply type:

<pre><code class="language-markup">hn</code></pre>

You will then be prompted to open a post. Type the number of the post to open a post or type 0 to quit and return to your terminal session.
