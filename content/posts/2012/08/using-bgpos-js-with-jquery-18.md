+++
title = 'Using jquery.bgpos.js with jQuery 1.8'
date = 2012-08-29T15:00:00-05:00
draft = false
tags = ['programming', 'javascript', 'jquery']
+++
Today, I spent way too much time working on a front-end problem to a code base that I wasn't very familiar with.  This project's animated navigation menu stopped working when upgrading from jQuery 1.7 to jQuery 1.8, and we needed jQuery 1.8 because of another dependency.  After digging my way through the code, I finally found that the problem was occurring because [Alexander Farkas' jquery.bgpos.js](http://code.google.com/p/exteenscript/downloads/detail?name=jquery.bgpos.js&can=2) jQuery plugin was not compatible with jQuery 1.8.

Since the internals of jQuery's animation code has changed from 1.7 to 1.8, the plugin needed to be updated as well.  I'm pasting the entirety of the change here so that, hopefully, maybe, someone else doesn't go through the same frustration that I went through today. I can't guarantee this code would work with jQuery versions less than 1.8 ;)

    /**
     * @author Alexander Farkas
     * v. 1.02
     *
     * Edited by Nelson Wells for jQuery 1.8 compatibility
     */
    (function($) {
        $.extend($.fx.step,{
            backgroundPosition: function(fx) {
                if (fx.pos === 0 && typeof fx.end == 'string') {
                    var start = $.css(fx.elem,'backgroundPosition');
                    start = toArray(start);
                    fx.start = [start[0],start[2]];
                    var end = toArray(fx.end);
                    fx.end = [end[0],end[2]];
                    fx.unit = [end[1],end[3]];
                }
                var nowPosX = [];
                nowPosX[0] = ((fx.end[0] - fx.start[0]) * fx.pos) + fx.start[0] + fx.unit[0];
                nowPosX[1] = ((fx.end[1] - fx.start[1]) * fx.pos) + fx.start[1] + fx.unit[1];
                fx.elem.style.backgroundPosition = nowPosX[0]+' '+nowPosX[1];
     
                function toArray(strg){
                    strg = strg.replace(/left|top/g,'0px');
                    strg = strg.replace(/right|bottom/g,'100%');
                    strg = strg.replace(/([0-9\.]+)(\s|\)|$)/g,"$1px$2");
                    var res = strg.match(/(-?[0-9\.]+)(px|\%|em|pt)\s(-?[0-9\.]+)(px|\%|em|pt)/);
                    return [parseFloat(res[1],10),res[2],parseFloat(res[3],10),res[4]];
                }
           }
        });
    })(jQuery);
