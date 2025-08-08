---
layout: post
title: "Shareable Web Links that Open iOS Deep Links"
author: "Ben Roesch"
---
When building [55Prophets][1] (available in the [app store][2]), we built in a number of deep linking options. I wrote in-depth about adding deep links to your iOS app [here][3].   
  
Now that we have these deep links, we wanted to be able to share them with people around the web. However, sharing links that look like `ffprophets://leagues/22/join` is confusing to people and isn\'t necessarily clickable/touchable in many cases. To make it easier to share deep links in the app, I added a quick and simple snippet to our nginx config that redirects deep link url\'s to open inside the app. It looks like this:  
> `location /deep_link/ {  if ($http_user_agent ~ iPhone) {    rewrite
> ^/deep_link/(.*) ffprophets://$1 permanent;  }  rewrite
> ^/deep_link/(.*) /deep_link.html last;  break;}`

Now, we can share urls like [http://55prophets.com/deep\_link/leagues/22/join][4] (download the app, visit this page on your iPhone and try it out!). When people access the url in mobile Safari on their device, it will redirect them to `ffprophets://leagues/22/join`, which will open in 55Prophets. If they access the page from a computer, it will show them the deep\_link.html page, which informs them that the link needs to be opened on a mobile device with the app installed and gives them a link to download it.  
  
  
P.S. If you found this helpful, I\'d love to hear from you on twitter, [@bcroesch][5]. 

[1]: http://55prophets.com
[2]: https://itunes.apple.com/us/app/55prophets/id622424094?ls=1&amp;mt=8
[3]: http://benroesch.com/2013/05/14/creating-view-controllers-for-deep-linked-ios-apps/
[4]: http://55prophets.com/deep_link/leagues/22/join
[5]: http://twitter.com/bcroesch
