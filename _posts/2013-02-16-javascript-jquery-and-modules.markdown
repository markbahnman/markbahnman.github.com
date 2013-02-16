---
layout: post
title: Javascript, jQuery, and Modules
subtitle: Refactoring the Hubski Enhancement Suite userscript
---

Having not done much with javascript other than some simple hacks, I was inspired by [this post](https://shanetomlinson.com/2013/testing-javascript-frontend-part-1-anti-patterns-and-fixes/) to do a major refactor of the <span class="hint hint--top" data-hint="Hubski Enhancement Suite">HES</span> userscript. You can look at the full refactored userscript [here](https://gist.github.com/markbahnman/4968550). It has the exact same functionality as the before but I believe it will be much nicer to read, maintain and extend with these changes.

### Making javascript human readable with jQuery

First things first, we want to the to be code easy to understand and thus easier to maintain. For instance, can you guess what this piece of code does?

{% prism javascript %}
window.location = document.getElementsByClassName('gridfeed')[0].childNodes[feedSelectionIndex].childNodes[1].childNodes[2].childNodes[1].childNodes[1].href;
{% endprism %}

It's not easy to tell without following the through each set of children. I think it can be agreed that this is easier on the eyes.

{% prism javascript %}
feed.currentNode.find('.savesplit > a:contains("hide")').click();
{% endprism %}

I'll admit, something as simple as <span class="hint hint--bottom" data-hint="The .savesplit, a, and :contains">selectors</span> blew my mind when I first saw them. One disadvantage to using jQuery in a userscript is that in Google Chrome the file containing jQuery is out of the scope of the userscript. In order to access the file you need to inject your script into the page.

{% prism javascript %}
function addScript(callback) {
    window.onload = function() {
        var script = document.createElement('script');
        script.textContent = '(' + callback.toString() + ')();';
        document.body.appendChild(script);
    }
}

function main() { /* Userscript goes here */ }

addScript(main);
{% endprism %}

This uses a simplified version of the addJquery function by [Erik Void](http://erikvold.com/blog/index.cfm/2010/6/14/using-jquery-with-a-user-script). We don't need to load jQuery ourselves because Hubski does that for us; we merely need it in our scope. Also, loading our own jQuery in addition to hubski's introduces its own set of problems.

### Abstracting functionality with modules
Each piece of major functionality can be wrapped into its own module. Each module has the general form

{% prism javascript %}
modules['moduleKey'] = (function() {
    var Module = {
        init: function() { /*initialize stuff*/},
        isLoaded: function() {/*determine if module should be run*/}
    };
    privateMember;
    privateFunction() {}
    return Module;
    }());
{% endprism %}

Every module needs at least two function: ```init()``` to initialize the module (attach event handlers, insert spans, etc) and ```isLoaded()``` to determine if the module should be run on the current page. This makes loading all applicable modules as easy as

{% prism javascript %}
for(mod in modules) {
    if(modules[mod].isLoaded()) {
        modules[mod].init();
    }
}
{% endprism %}

The eventual goal of using this pattern is to be able to do unit tests which will require that we be able to initialize and cleanup the modules for each test. 

### Getting rid of massive if/else blocks
The shortcuts module contains all of the keyboard shortcuts for every applicable page on hubski. One of the major changes made to the shortcut functions is instead of using massive if/else blocks for determining which combination of keys was we use objects to map the functions to the key code. We have seperate objects for each applicable page (A single post, the user feed, notifications page, etc) and store keycode/function pairs in each object as such.

{% prism javascript %}
var postShortKeys = {
        '65': // 'a'
            function() { $('.longplusminus > a').click(); },
        '82': // 'r'
            function() { $('[name="text"]').focus(); },
        '83': // 's'
            function() { $('.titlelinks > a:contains("save")').click(); }
    };
{% endprism %}

Once we have all these objects defined we can determine which ones should be available to the user and combine them using the [extend](http://api.jquery.com/jQuery.extend/) function.

{% prism javascript %}
var keyMap = {};
function buildKeyMap() {
    $.extend(keyMap,generalShortKeys);
    if(isPost) {$.extend(keyMap,postShortKeys)};
    // ...
}        
{% endprism %}

This simplifies the function call in the keyup event handler to something like 
{% prism javascript %}
    keyMap[event.code]();
{% endprism %}

There are a few drawbacks to using this method as opposed to a if/else or switch/case block, however. For instance, it doesn't look as nice when you are using several different key codes to do the same thing. If you were using a switch statement you could let the those cases just fall through. Also if you want to use key combinations such as ```Shift + <key>``` you can't simply rely on the keycode, as ```Shift + o``` and ```o``` will be detected with the same code.