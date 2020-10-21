---
title: Safari is Fast, but So What?
date: 2020-10-21 11:52:13
tags: [Safari, Apple, Web, W3C, JavaScript]
---

中文版本[见此](/2020/10/21/safari-is-fast-but-so-what/)

## A Mysterious Bug

In a day of 2016, we found that our users could not pass the CDN authentication with their iPhones. We then took several days to debug. The situation is that we need to upload three files at the same time. We use the token of the user to generate three random ids. The CDN server would use these ids to authenticate the upload of user files. In this case, we don't have to transfer the files to CDN on our server.

But soon, iOS users found a weird problem. Users could only upload one of the free files. After debugging, we found that after uploading the first file, the next two ids become illegal. Furthermore, we found the three ids fetched by Safari are precisely the same?!

## Reproduction

I soon designed a reproduction of this bug:

```ruby
require 'sinatra'

get '/' do
  <<-EOF
<html>
<script type="text/javascript">
function reqListener () {
  console.log(this.responseText);
}

for (i = 0; i < 3; i++) {
  var oReq = new XMLHttpRequest();
  oReq.addEventListener("load", reqListener);
  oReq.open("GET", "/count");
  oReq.send();
}
</script>
</html>
EOF
end

count = 0
get '/count' do
  count += 1
  count.to_s
end

```

On Firefox, you would get 1 2 3, but on Safari, you would get three 1s.

For the same API request, if the parameters are identical. Safari may return same results of all these requests if they are sent asynchronously.

## Analysis

In general, we may think that GET requests of RESTful API are Idempotent. If we treat x as the status of the server, and f is the GET request, we would have:

$$
f(f(x)) = f(x)
$$

The idempotence ensures that the side effects of multiple calls are identical to a single call. We could infer that all responses of the same GET request should also be exact. But first, this is the requirements of RESTful API instead of the HTTP standard. Second, some of the requests can't meet such needs like random generator or statistics.

If we assume the reliability of idempotence, we could hash the parameters to improve the performance of callbacks in the event engine of a browser. But apparently, this assumption is incorrect, and Safari does make such optimizations, which causes the bug.

## Bug Report

![Screenshot](/static/safari-js-bug.png)

I submitted this bug from Apple's feedback system in 2016. After four years, the feedback system has evolved to Feedback Assistant; Mac OS X has been renamed to macOS; El Capitan has been upgraded to Big Sur. But this bug is still in the latest Safari (16610.2.8.1.1). My ticket is still open, with NO RESPONSE.

## Conclusion

Safari is fast, efficient, and power-saving. But if Safari can't keep essential compatibility with W3C Web API standards, how dare we using this browser? But due to the monopoly of iOS and App Store, iOS developers are not allowed to use third-party Webview, including Chrome and Firefox. We, the web developers, have to compromise with the incorrect implementation of Safari. Even the evil IE, didn't use the monopoly of the operating system to force users to accept the specification of a browser.

Safari is not only the new IE, but it is also more evil than IE. Apple is the destroyer of the free Internet system.

F**k you, Apple.
