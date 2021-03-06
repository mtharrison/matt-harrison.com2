---
title: "Perfect web audio on iOS devices with the Web Audio API"
date: 2014-01-02T00:00:00+01:00
---
TL;DR A lot of the restrictions imposed on the HTML5 audio element by iOS can be overcome by using the <a href="#webAudio">Web Audio API</a>.

---

If you've ever built a web based game that requires sound effects, you've no doubt felt the frustration of getting what is trivial to implement on desktop browsers to work smoothly on iOS devices.

Initally the obvious way to implement audio is to use the HTML5 &lt;audio> element. The official browser support for the element is widespread enough to coax you into a false sense of security (http://caniuse.com/audio).

You could go ahead and put all your sound effects in separate &lt;audio> elements like so:

<pre><code class="language-markup">&lt;audio id="blast" src="blast.mp3">&lt;/audio>
&lt;audio id="smash" src="smash.mp3">&lt;/audio>
&lt;audio id="pow" src="pow.mp3">&lt;/audio>
&lt;audio id="bang" src="bang.mp3">&lt;/audio>
&lt;!-- Background Track -->
&lt;audio id="smooth-jazz" src="smooth-jazz.mp3" autoplay>&lt;/audio>
</code></pre>

Then say you have a gun trigger with a class of `blasterTrigger`:

<pre><code class="language-markup">&lt;button class="blasterTrigger">Shoot!&lt;/button>
</code></pre>

You could then use the Javascript API to control the playing of the blast sound when the trigger is clicked like so:

<pre><code class="language-javascript">var blasterTrigger = document.querySelector(".blasterTrigger");
blasterTrigger.addEventListener("click", function(){
    document.getElementById("blast").play();
});
</code></pre>
<br>
###The hunch

If you test the above code in most desktop browers, you'll find it works perfectly (given you provide a suitable audio format for that browser). If you take out your iPad though and give it a go, you'll find the background track doesn't play at all.

Also, if you try to fire any of the other sounds simultaneously, you'll probably only hear one. Asyncronous sounds (i.e. ones that are fired in code and not in the call stack of a UI interaction like a click or touch event) won't play at all.

The problem is that even though Safari on iOS _does_  support the audio element, it imposes some restrictions of its own on playback. These restrictions are driven by a desire to save the user bandwidth and memory. Some have suggested Apple have done this because they want to keep media rich applications to be native and sold through the App Store.

The iOS restrictions basically sum up like this:

* Only 1 channel of audio can play at a time so no layering or overlapping of sounds. This really restricts you if you want to have background audio in your game

* iOS will ignore the autoplay attribute on your elements

* You can't load your audio tracks asyncronously, they must be loaded via a UI interaction like a click event.

####Possible solutions

There are a few method to attempt to overcome the limitations of the &lt;audio> element on iOS.

**Queueing**

One approach is to have a single audio channel and queue the tracks you wish to play on a single queue to get played sequentially.

<pre><code class="language-markup">&lt;audio id=&quot;vivaldi&quot; src=&quot;vivaldi.mp3&quot;&gt;&lt;/audio&gt;
&lt;audio id=&quot;brahms&quot; src=&quot;brahms.mp3&quot;&gt;&lt;/audio&gt;
&lt;audio id=&quot;mozart&quot; src=&quot;mozart.mp3&quot;&gt;&lt;/audio&gt;
&lt;button data-track=&quot;vivaldi&quot;&gt;Play Vivaldi&lt;/button&gt;
&lt;button data-track=&quot;brahms&quot;&gt;Play Brahms&lt;/button&gt;
&lt;button data-track=&quot;mozart&quot;&gt;Play Mozart&lt;/button&gt;
</code></pre>

<pre><code class="language-javascript">var buttons,
    addToPlayQueue,
    track,
    trackEnded,
    i,
    queue = [],
    isPlaying = false,
    tracks;

buttons = document.getElementsByTagName("button");
tracks = document.getElementsByTagName("audio");

addToPlayQueue = function (event) {
    event.preventDefault();
    var track = this.dataset.track;
    queue.push(track);
};

trackEnded = function (event) {
    console.log("Track just ended");
    isPlaying = false;
};

for (i = 0; i < buttons.length; i++)
    buttons[i].addEventListener("click", addToPlayQueue);

for (i = 0; i < tracks.length; i++)
    tracks[i].addEventListener("ended", trackEnded);

//Run loop
setInterval(function () {
    if (queue.length > 0 && isPlaying === false) {
        document.getElementById(queue.pop()).play();
        isPlaying = true;
    }
}, 500);</code></pre>
<br>
**Audiosprites**

Another approach is to use audiosprites (think CSS sprites but audio files). This is where you combine all your audio files into a single track separated by silence. To play a particular sound, you just seek to that time and start playing.

There are various tools to help with the creation and playing of these. There's an NPM package https://github.com/tonistiigi/audiosprite which is a wrapper around _ffmpeg_ that will create your sprites in many formats and output a handy JSON file with the timing of the various tracks encoded within. I originally used this approach, letting a <a href="http://gruntjs.com/">Grunt</a> task manage the creation of the sprites from my sources files as part of the build process. There was also a library from <a href="http://zynga.com/">Zynga</a> called Jukebox that could take these JSON files and expose a convenient API for playing tracks (that implemented a similar run loop and queueing approach as above) but as of Jan 2014 it seems to have been taken down from Github.

Although this approach certainly is an improvement and overcome several of the issues, it's still not perfect. You can still only play one track and a time and if you queue tracks in quick succession, no matter how good your queue implementation, iOS will sometimes just discard the operation.

Luckily, there is another solution…enter the Web Audio API.

<h3 id="webAudio">Web Audio API</h3>

The web audio API is a new standard by the w3c that has started appearing in browsers. It’s supported in the most recent versions of Firefox, Chrome and safari (including iOS6/7). Check http://caniuse.com/audio-api for up to date support information.

It is designed for sophisticated audio sythesis and manipulation and its architecture is modelled around professional sound engineering techniques.

It can however be used to play a basic audio buffer, including looping and stopping the track. It doesn’t (currently) suffer the same limitations as the HTML audio element and it’s Javascript API making it a viable candidate for web based games with sound effects. Using it on iOS sidesteps the single-channel restriction.

The underlying API is optimised at the C++/Assesmbly level so it’s very fast and has a low consumption of memory.

Here's some sample code that I wrote that uses the Web Audio API and acts as a shim for iOS devices to take place of the library (buzz.js) we're using for desktop browsers.

<pre><code class="language-javascript">try {
    window.AudioContext = window.AudioContext || window.webkitAudioContext;
    window.audioContext = new window.AudioContext();
} catch (e) {
    console.log("No Web Audio API support");
}

/*
 * WebAudioAPISoundManager Constructor
 */
 var WebAudioAPISoundManager = function (context) {
    this.context = context;
    this.bufferList = {};
    this.playingSounds = {};
};

/*
 * WebAudioAPISoundManager Prototype
 */
WebAudioAPISoundManager.prototype = {
     addSound: function (url) {
        // Load buffer asynchronously
        var request = new XMLHttpRequest();
        request.open("GET", url, true);
        request.responseType = "arraybuffer";

        var self = this;

        request.onload = function () {
            // Asynchronously decode the audio file data in request.response
            self.context.decodeAudioData(
                request.response,

                function (buffer) {
                    if (!buffer) {
                        alert('error decoding file data: ' + url);
                        return;
                    }
                    self.bufferList[url] = buffer;
                });
        };

        request.onerror = function () {
            alert('BufferLoader: XHR error');
        };

        request.send();
    },
    stopSoundWithUrl: function(url) {
        if(this.playingSounds.hasOwnProperty(url)){
            for(var i in this.playingSounds[url]){
                if(this.playingSounds[url].hasOwnProperty(i))
                    this.playingSounds[url][i].noteOff(0);
            }
        }
    }
};

/*
 * WebAudioAPISound Constructor
 */
 var WebAudioAPISound = function (url, options) {
    this.settings = {
        loop: false
    };

    for(var i in options){
        if(options.hasOwnProperty(i))
            this.settings[i] = options[i];
    }

    this.url = url + '.mp3';
    window.webAudioAPISoundManager = window.webAudioAPISoundManager || new WebAudioAPISoundManager(window.audioContext);
    this.manager = window.webAudioAPISoundManager;
    this.manager.addSound(this.url);
};

/*
 * WebAudioAPISound Prototype
 */
WebAudioAPISound.prototype = {
    play: function () {
        var buffer = this.manager.bufferList[this.url];
        //Only play if it's loaded yet
        if (typeof buffer !== "undefined") {
            var source = this.makeSource(buffer);
            source.loop = this.settings.loop;
            source.noteOn(0);

            if(!this.manager.playingSounds.hasOwnProperty(this.url))
                this.manager.playingSounds[this.url] = [];
            this.manager.playingSounds[this.url].push(source);
        }
    },
    stop: function () {
        this.manager.stopSoundWithUrl(this.url);
    },
    getVolume: function () {
        return this.translateVolume(this.volume, true);
    },
    //Expect to receive in range 0-100
    setVolume: function (volume) {
        this.volume = this.translateVolume(volume);
    },
    translateVolume: function(volume, inverse){
        return inverse ? volume * 100 : volume / 100;
    },
    makeSource: function (buffer) {
        var source = this.manager.context.createBufferSource();
        var gainNode = this.manager.context.createGainNode();
        gainNode.gain.value = this.volume;
        source.buffer = buffer;
        source.connect(gainNode);
        gainNode.connect(this.manager.context.destination);
        return source;
    }
};</code></pre>

The `WebAudioAPISound` class can then be used like so:

<pre><code class="language-javascript">var blastSound, smashSound, backgroundMusic;

blastSound = new WebAudioAPISound("blast.mp3");
smashSound = new WebAudioAPISound("smash.mp3");
backgroundMusic = new WebAudioAPISound("smooth-jazz.mp3", {loop: true});

backgroundMusic.play();
blastSound.play();
smashSound.play();

//Play background music for 30 seconds
setTimeout(function(){
    backgroundMusic.stop();
}, 30 * 1000);</code></pre>

You should find that your audio (including background track) all now works seemlessly on iOS devices.

I've found the following sources useful in writing this:

1. http://www.html5rocks.com/en/tutorials/webaudio/games/
2. http://www.html5rocks.com/en/tutorials/webaudio/intro/

Please feel free to leave comments if you like this or think something could be improved.
