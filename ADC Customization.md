# ADC Customization

In this blog post, I'll show you some customizations for nFactor using the RfWebUi  theme on the ADC. Some basic, some more advanced.

Let it be known that I am not impressed with Citrix documentation on this subject. I have some bones to pick, such as:
- The nFactor tooling itself is confusing
- The nFactor documentation is even worse
- Citrix Support articles written outside of docs.citrix.com are always of extremely poor quality
- The community has had to carry Citrix because of this
- The community cannot grow because Citrix hides ADC behind a paywall (trial is time-limited and VPX10 developer license requires a paid ADC license)
- A lot of RfWebUi customizations are variations of StoreFront customizations, and it's unclear how these map to RfWebUi on the ADC

Because of some of these points, community solutions are often out of date or misleading.


## Design
Customizing Gateway is a frustrating process that will test your patience. Citrix has for the sake of modularity split up the functions of each file.

You could make all of your customizations in the tmindex.html file, or copy tmindex.html, make your changes there and use a responder/rewrite policy to use that, or use the script.js file for all of your customizations. However, there's a "supported" way of performing these customizations, which is as follows:

- Login Schema for the forms and buttons on the page
- strings.(locale).json for any strings
- plugins.xml for RfWebUi defaults
- script.js for modularity and custom code
- theme.css for styling

In this blog post, we assume this scenario:
-   ADC 13
-   Content Switching vServer
-   Citrix Gateway vServer
-   AAA vServer
-   nFactor Flow bound to AAA vServer
-   RfWebUi applied to Gateway and AAA
Our theme will be located in /var/netscaler/logon/themes/RfWebUi/.



## Enhanced Authentication Feedback
File: **/var/netscaler/logon/LogonPoint/receiver/js/ctxs.core.min.js**</br>
References: [https://www.jgspiers.com/netscaler-enhanced-authentication-feedback/](https://www.jgspiers.com/netscaler-enhanced-authentication-feedback/)

By default, Gateway will simply return a "Incorrect user name or password" message on any authentication failure, regardless if it's actually incorrect username or password, or if the password's expired, or account disabled, etc.

We can change this by enabling Enhanced Authentication Feedback, which outputs a more precise message based on Citrix-defined error codes.

Citrix Gateway > Global Settings > Change authentication AAA settings > Enable Enhanced Authentication Feedback

We enabled Enhanced Authentication Feedback for messages such as "Account Locked" or "Password Expired". As the default strings are very telling, "No such user" being an example that an adversary can use to enumerate possible user names, it's good security hygiene to obfuscate some of those strings.

For our scenario, we edited these:

- BEFORE: `"errorMessageLabel4009":"User not found"`
- AFTER:  `"errorMessageLabel4009":"Incorrect user name or password"`

This is also possible on X1:</br>
/var/netscaler/logon/themes/THEMENAME/resources/en.xml

- BEFORE: `<String id="errorMessageLabel4009">User not found</String>`
- AFTER:  `<String id="errorMessageLabel4009">Incorrect user name or password.</String>`




## StoreFront 3.0 vs RfWebUI
A thing that is not very clear on the official docs, is that many StoreFront 3.0 customizations translate quite well to RfWebUi on the ADC, but the file names are different. With this in mind, we're going to be using a lot of StoreFront customization content, because that community and its content is just much more mature.

| Storefront | RfWebUi     |
| ---------- | ----------- |
| Web.config | Plugins.xml |
| Custom.css | Theme.css   |
| Script.js  | Script.js   |

   

I would like to give a shoutout to some very helpful blog posts, articles and websites.

- [Citrix Customization Cookbook (Citrix Blogs)](https://www.citrix.com/blogs/2015/08/25/citrix-customization-cookbook/): The first article anyone tweaking RfWebUi should read
- [Adding Text, Links and Other Elements to the NetScaler Logon Page - Part 2 (mycugc.org)](https://www.mycugc.org/blogs/cugc-blogs/2016/12/15/adding-text-links-and-other-elements-to-the-netsca): It's not perfect - it actually kept me hanging for days because the author does not explain their environment. Me and some others set up a responder policy for it to get 0 hits because the binding is in the incorrect place, because our vServer design is different.
- [nFactor Extensibility (citrix.com)](https://docs.citrix.com/en-us/citrix-adc/current-release/aaa-tm/authentication-methods/multi-factor-nfactor-authentication/nfactor-extensibility.html): Goes into great detail on Login Schemas
- [Citrix Synergy TV - SYN229 - nFactor and Login Schemas... - YouTube](https://www.youtube.com/watch?v=dQRJo1Dm_Aw): Also goes into great detail on Login Schemas
- [NetScaler Enhanced Authentication Feedback – JGSpiers.com](https://www.jgspiers.com/netscaler-enhanced-authentication-feedback/): Great reference on Enhanced Authentication Feedback
- [How To Detect Caps Lock (w3schools.com)](https://www.w3schools.com/howto/howto_js_detect_capslock.asp): Used this as a base to implement a Caps Lock warning

And a small wall of shame for the lousy Citrix articles I've had to deal with.

- [How to change default view on storefront website. (citrix.com)](https://support.citrix.com/article/CTX216893): Terrible.
- [How to select by default Apps instead of Favorites tab on RfWebUI Theme (citrix.com)](https://support.citrix.com/article/CTX262102): Bad and out of date.

Among many other misspelled, badly worded support articles.


## Basic Customization
RfWebUi provides us with 4 HTML divs that we can (somewhat easily) add HTML code to. In our case, we'll use them for basic text and a link.

.customAuthHeader (top of page)
.customAuthFooter (bottom of page)
.customAuthTop (above form)
.customAuthBottom (below form)

In the real-world scenario which this blog builds on, we will use .customAuthTop, .customAuthBottom and .customAuthFooter.

-   .customAuthTop will display our Caps Lock warning
-   .customAuthBottom will display our "forgot password? Click here to reset" link
-   .customAuthFooter will display our helpdesk contact information

When adding HTML code to tmindex.html, either with script.js or editing the HTML file on the ADC, you should add a \<p\>-tag with the class name "_ctxstxt_\<variablename\>", as you can then define the string (as text or HTML) in the strings.en.json file, but without the _ctxstxt_ prefix.

We will be defining the divs in script.js.



## RfWebUi - script.js
File: **/var/netscaler/logon/themes/RfWebUI/script.js**

Any JavaScript is to be placed in this file. As you can access the DOM, you could do all of your customizations in JavaScript, but stick to the Citrix approved solutions for now. This is complicated enough as it is.

-   We define our HTML divs, which have strings defined in strings.en.json.
-   We define our code for WHEN the Caps Lock warning should show on the .customAuthTop div.

**script.js:**
```
$('.customAuthBottom').html('<p class="_ctxstxt_sspr"></p>');
$('.customAuthFooter').html('<p class="_ctxstxt_helpdesk"></p>');
$('.customAuthTop').html('<p class="_ctxstxt_capslock"</p>');
// we select the div with the text, which is hidden by default
var warningtext = document.querySelector(".customAuthTop");
// we listen on each keypress
document.addEventListener("keyup", function(event) {
// is capslock on?
var caps = event.getModifierState && event.getModifierState('CapsLock');
 if (caps === true) {
// show text
warningtext.style.display="block";
 } else {
// hide text
warningtext.style.display="none";
 }
});
```



## RfWebUi – Custom strings (ctxs.strings.json + strings.en.json)
File: **/var/netscaler/logon/LogonPoint/receiver/js/localization/en/ctxs.strings.json**</br>
File: **/var/netscaler/logon/themes/\<themename>/strings.en.json**

**ctxs.strings.json** is the file listing all builtin strings. These can be changed with the strings.\<locale>.json file in the theme folder. We will be using strings.en.json for this example.

We apply any modifications to the strings.en.json file.
Example:
Add to strings.en.json: "waitMessage":"Please press Approve in your Authenticator app .."

In our scenario, we added this:
```
{"waitMessage":"Please press Approve in your Authenticator app ..",
"Epa_Pre_Reqm_Msg":"Checking if the device is a <company> PC.. If it's not a <company> PC, press Skip Check.",
"helpdesk":"Contact <company> IT Support: itsupport@company.com / +1 123456789",
"sspr":"Has your password expired, or have you forgotten it?<br><a class='ssprlink' href='https://aka.ms/sspr'>Reset your password here</a>"
}
```


## RfWebUi – Plugins.xml
File: **/var/netscaler/logon/themes/RfWebUI/Plugins.xml**
plugins.xml == web.config on StoreFront 3.0.

For our scenario, we wanted the Desktops tab to be shown by default, instead of the Favorites tab.
- BEFORE: `defaultView="apps"`
- AFTER:  `defaultView="desktops"`


## RfWebUi – theme.css
File: **/var/netscaler/logon/themes/GEUS-RfWebUI/css/theme.css**

Go nuts with CSS. We didn't bother that hard, so we copy-pasted from the internet.
A.sspr: A is the \<a> tag, sspr is the class name. The possible options are link, visited, hover and active. Link is yellow normally, becomes red when hovering over.

theme.css:
```
.customAuthBottom {
color:white;
height:30px;
font-size:14px;
width:80%;
margin:0 auto;
padding-left:180px;
padding-top:50px;
text-align: center;
}

.customAuthFooter {
position: absolute;
bottom: 5px;
height: 30px;
font-size: 12px;
color: white;
font-weight: bold;
text-align: center;
width: 100%;
background-color: black;
padding-top: 5px;
}

a.ssprlink:link, a.ssprlink:visited {
   color: yellow;
}

a.ssprlink:hover, a.ssprlink:active {
   color: red;
}
```
