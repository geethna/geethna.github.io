---
title: DIVA Android app Walkthrough
date: 2024-04-30 00:00:00 +0530
categories: [Android]
tags: [android, re]
---

As a part of my journey of learning android reversing, I started exploring different intentionally vulnerable apks for practise. The first one I found was a simply app named [sieve](https://github.com/tanprathan/sievePWN). But I realised a writeup for that would be too short. 

Then I came across the [DIVA](https://github.com/payatu/diva-android) apk by Payatu. This Damn insecure and vulnerable App) is an App intentionally designed to be insecure. It covers common vulnerabilities in Android apps ranging from insecure storage, input validation to access control issues.

It contains a total of 13 vulnerabilities. I will be covering each of these issues in sections. 

### InsecureLogging


I started by analysing the LogActivity. 


```java
    public void checkout(View view) {
        EditText cctxt = (EditText) findViewById(R.id.ccText);
        try {
            processCC(cctxt.getText().toString());
        } catch (RuntimeException e) {
            Log.e("diva-log", "Error while processing transaction with credit card: " + cctxt.getText().toString());
            Toast.makeText(this, "An error occured. Please try again later", 0).show();
        }
    }
```

Here, we can see that when a user inputs the credit card number in `ccText` it takes it and stores. Then it sends it to the processCC method. 

```java
    private void processCC(String ccstr) {
        RuntimeException e = new RuntimeException();
        throw e;
    }
```

This method essentially deliberately crashes the application by throwing an unhandled exception of type RuntimeException. 

Then the code inside the catch is executed. Hence, the message "Error while processing transaction with credit card: " along with the entered credit card number is stored in `diva-log` and a notification is sent as displayed in the UI. 

![alt text](assets/images/log1.png "log1")

The issue here is that the credit card details are stored in plaintext which enables anyone who has access to `diva-logs` to view these details. 

In an attacker's view, we can first get the process id using : `adb shell ps` 

```bash
adb shell ps | grep diva
u0_a55    2411  903   1145804 89392 SyS_epoll_ 791307ca34 S jakhar.aseem.diva
```

Now to get the logs : 

```
adb shell logcat | grep 2411
```

![alt text](assets/images/log2.png "log2")


### HardCoding Issues - Part 1

Here, we can see that when a user inputs the a vendor key in in `hcKey` it takes it stores in a variable of type EditText.

```java
public void access(View view) {
        EditText hckey = (EditText) findViewById(R.id.hcKey);
        if (hckey.getText().toString().equals("vendorsecretkey")) {
            Toast.makeText(this, "Access granted! See you on the other side :)", 0).show();
        } else {
            Toast.makeText(this, "Access denied! See you in hell :D", 0).show();
        }
    }
```

Then it compares it with the hardcoded value "vendorsecretkey", if its correct access is granted (or rather just printed) if not access is denied. 

This was fairly very easy but very important. We can see that the developer has hardcoded the value of the supposedly "secret" vendor key which makes it easy for any user with the apk to get this value and use it for further attacks. 

### InsecureDataStorage - Part 1

When we click on this option, we can see that we are supposed to enter a third party username and password which is stored in the app. A notification is printed when the storage is successful - "3rd party credentials saved successfully!". 

I went and started looking at `InsecureDataStorage1Activity`. 

```java
public void saveCredentials(View view) {
        SharedPreferences spref = PreferenceManager.getDefaultSharedPreferences(this);
        SharedPreferences.Editor spedit = spref.edit();
        EditText usr = (EditText) findViewById(R.id.ids1Usr);
        EditText pwd = (EditText) findViewById(R.id.ids1Pwd);
        spedit.putString("user", usr.getText().toString());
        spedit.putString("password", pwd.getText().toString());
        spedit.commit();
        Toast.makeText(this, "3rd party credentials saved successfully!", 0).show();
    }
```

The SharedPreferences object spref is obtained using `PreferenceManager.getDefaultSharedPreferences(this)`.

Now, the methods like putString(), putInt(), etc., on the spedit, let's us to add or modify key-value pairs in the preference file associated with spref.

The key-value pairs are stored in an XML file in the app's data directory. 

Specifically, the SharedPreferences file of an app is stored in its private data directory (`/data/data/<package_name>/shared_prefs/`) and is only accessible to that specific app.	

By default, one app cannot directly access the SharedPreferences file of another app. Android enforces a strict security model that isolates each app's data and prevents unauthorized access to it. This is done to protect user privacy and ensure the security of app data.

Each app runs within its own sandboxed environment, and its private data is stored in directories that are inaccessible to other apps.

But we can get this data using the command : 

```bash
adb shell cat /data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml            

```

So in conclusion I understood that it is not possible for another app to access this data without other privileges like Content Providers, App-to-App Communication or Permissions. But we can see that the key pairs are stored in plaintext which again is very insecure method to store data. 

### InsecureDataStorage - Part 2

Again, the basic functionality is the same but the data is stored in an SQL databased name - `ids2`. 

```java
public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        try {
            this.mDB = openOrCreateDatabase("ids2", 0, null);
            this.mDB.execSQL("CREATE TABLE IF NOT EXISTS myuser(user VARCHAR, password VARCHAR);");
        } catch (Exception e) {
            Log.d("Diva", "Error occurred while creating database: " + e.getMessage());
        }
        setContentView(R.layout.activity_insecure_data_storage2);
    }

    public void saveCredentials(View view) {
        EditText usr = (EditText) findViewById(R.id.ids2Usr);
        EditText pwd = (EditText) findViewById(R.id.ids2Pwd);
        try {
            this.mDB.execSQL("INSERT INTO myuser VALUES ('" + usr.getText().toString() + "', '" + pwd.getText().toString() + "');");
            this.mDB.close();
        } catch (Exception e) {
            Log.d("Diva", "Error occurred while inserting into database: " + e.getMessage());
        }
        Toast.makeText(this, "3rd party credentials saved successfully!", 0).show();
    }
```

This is stored in the path `/data/data/jakhar.aseem.diva/databases/ids2`, so I first pulled this file using adb and tried to view it using an [sqlite viewer website](https://inloop.github.io/sqlite-viewer/) 

![alt text](assets/images/data2.png "data2")

I could see the credentials I entered stored in plaintext. 

### InsecureDataStorage - Part 3

This time again, the functionality is same but the data is stored in a different way. 

To store the credentials, a temporary file is created named `unifo*` and is given readable and writeable permissions. Using `FileWriter` the username and password is converted to strings and stored.

```java
    public void saveCredentials(View view) {
        EditText usr = (EditText) findViewById(R.id.ids3Usr);
        EditText pwd = (EditText) findViewById(R.id.ids3Pwd);
        File ddir = new File(getApplicationInfo().dataDir);
        try {
            File uinfo = File.createTempFile("uinfo", "tmp", ddir);
            uinfo.setReadable(true);
            uinfo.setWritable(true);
            FileWriter fw = new FileWriter(uinfo);
            fw.write(usr.getText().toString() + ":" + pwd.getText().toString() + "\n");
            fw.close();
            Toast.makeText(this, "3rd party credentials saved successfully!", 0).show();
        } catch (Exception e) {
            Toast.makeText(this, "File error occurred", 0).show();
            Log.d("Diva", "File error: " + e.getMessage());
        }
    }
```

First we can find the filename : 

```bash
adb shell ls /data/data/jakhar.aseem.diva/
```

Everytime a new pair is stored a temporary file is created. 
One such file was `uinfo604414850tmp` at my end and the contents had these credentials stored

### InsecureDataStorage - Part 4

The same functionality but this time it is writing to external storage. I was using an emulated device Pixel 4 - API 23 (Android 6.0) but when I tried to save the credentials, I kept getting an error. When I checked the logs, it was : 

`Diva    : File error: /storage/emulated/0/.uinfo.txt: open failed: EACCES (Permission denied)`

It could be due to the app does not having permission to write to external storage. This is likely because starting from Android 6.0 (API level 23), apps need to request runtime permission from the user to access certain dangerous permissions, including WRITE_EXTERNAL_STORAGE.

So I was unable to get this working, but from what I know this file would be saved in the `/sdcard/` directory. So we can use the adb command to navigate to that and cat the contents of `uinfo.txt`. 


### Input Validation Issues - Part 1

In this challenge the app takes a username as input in the textbox and prints out the details if the user exists. It is mentioned that there is no input validation done when checking for a user.
There are 3 users stored by default and the objective is to find a payload which will printout the details of all these user's. 

This is vulnerable part of the code : 

```java
    public void search(View view) {
        EditText srchtxt = (EditText) findViewById(R.id.ivi1search);
        try {
            Cursor cr = this.mDB.rawQuery("SELECT * FROM sqliuser WHERE user = '" + srchtxt.getText().toString() + "'", null);
            StringBuilder strb = new StringBuilder("");
            if (cr != null && cr.getCount() > 0) {
                cr.moveToFirst();
                do {
                    strb.append("User: (" + cr.getString(0) + ") pass: (" + cr.getString(1) + ") Credit card: (" + cr.getString(2) + ")\n");
                } while (cr.moveToNext());
            } else {
                strb.append("User: (" + srchtxt.getText().toString() + ") not found");
            }
            Toast.makeText(this, strb.toString(), 0).show();
        } catch (Exception e) {
            Log.d("Diva-sqli", "Error occurred while searching in database: " + e.getMessage());
        }
    }
``` 

The `cr` variable will store the output of this command. And it will print out each row of cr one by one, until the count is 0. 

payload : `'' OR '1'='1'`

Gives the output : 

![alt text](assets/images/sql1.png "sql1")


### Input Validation Issues - Part 2

In this challenge, we are supposed to enter a URL and it loads the URL obtained from the EditText into the WebView (wview). It instructs the WebView to navigate to the URL specified by the user input.

```java
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_input_validation2_urischeme);
        WebView wview = (WebView) findViewById(R.id.ivi2wview);
        WebSettings wset = wview.getSettings();
        wset.setJavaScriptEnabled(true);
    }

    public void get(View view) {
        EditText uriText = (EditText) findViewById(R.id.ivi2uri);
        WebView wview = (WebView) findViewById(R.id.ivi2wview);
        wview.loadUrl(uriText.getText().toString());
    }
```

I also noticed that `setJavaScriptEnabled(true)`. This method enables JavaScript execution in the WebView. By default, JavaScript is disabled for security reasons. Enabling JavaScript allows the WebView to interpret and execute JavaScript code embedded in web pages.

I tried to inject javascript, but I did not get anything fruitful. 

The objective of this challenge is to try and access sensitive information. So let's try to access a file using this functionality. 

I tried to display the shared preferences file which we mentioned earlier, at this path - `/data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml`

![alt text](assets/images/url1.png "url1")

Sucessful! 

### Access Control Issues - Part 1

The challenge is pretty simple. When we click the `VIEW API CREDENTIALS` button, it dispalyes the secret credentials. 

We have to find a way to get these credentials or open this intent outside of the app. 

We can try and identify the activity which is triggered when we click on this button using the logs : 

`2068 I ActivityManager: START u0 {act=jakhar.aseem.diva.action.VIEW_CREDS cmp=jakhar.aseem.diva/.APICredsActivity} from uid 10055 on display 0
`

In the source code we can see that this acticity is defined in the manifest file with intent-filters, which means this activity is exported and any external app can start this acticity. 

```java
 <activity
            android:label="@string/apic_label"
            android:name="jakhar.aseem.diva.APICredsActivity">
            <intent-filter>
                <action android:name="jakhar.aseem.diva.action.VIEW_CREDS"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
        </activity>
```

So using the adb command we can trigger this activcity outside the app : 

`adb shell am start -n jakhar.aseem.diva/.APICredsActivity`

### Access Control Issues - Part 2

I used the text search in jadx to find the class where the methods triggering this issue was defined. This can also be found using logcat. 

The challenge here again is to access the API credentials outside of the app. 

We can follow the same steps as before to trigger the activity outside of the app using adb shell since it is exported. But when we do that the activity will open the `Enter PIN` page. 

The reason is due to a check happening here : 

```java
boolean bcheck = i.getBooleanExtra(getString(R.string.chk_pin), true);
        if (!bcheck) {
            apicview.setText("TVEETER API Key: secrettveeterapikey\nAPI User name: diva2\nAPI Password: p@ssword2");
            return;
        }
```

We can only get these credentials if the `chk_pin` value is set to false. 

To find the variable name we can go to /res/values/strings.xml and seach for `chk_pin` 

`<string name="chk_pin">check_pin</string>` - this is what we get

Combining everything, the command is : 

`adb shell am start -n jakhar.aseem.diva/.APICreds2Activity --ez check_pin false`

The `--ez` flag is used to specify a boolean extra. 

### Access Control Issues - Part 3

In this challenge, we can access private notes after creating a PIN once. We are supposed to access these private notes outside of the app without knowing the PIN. 

The main classes to start looking are `AccessControl3Activity` and `AccessControl3NotesActivity`. 

The second activity is launched when we click `VIEW PRIVATE NOTES` button. 

Here, we can see that the `accessNotes` method checks the pin entered. It retrieves the user-entered PIN from an EditText field (pinTxt). It retrieves the saved PIN from SharedPreferences (spref). It compares the user-entered PIN with the saved PIN.

```java
        if (userpin.equals(pin)) {
            ListView lview = (ListView) findViewById(R.id.aci3nlistView);
            Cursor cr = getContentResolver().query(NotesProvider.CONTENT_URI, new String[]{"_id", "title", "note"}, null, null, null);
            String[] columns = {"title", "note"};
            int[] fields = {R.id.title_entry, R.id.note_entry};
            SimpleCursorAdapter adapter = new SimpleCursorAdapter(this, R.layout.notes_entry, cr, columns, fields, 0);
            lview.setAdapter((ListAdapter) adapter);
            pinTxt.setVisibility(4);
            abutton.setVisibility(4);
            return;
        }
```

If the user-entered PIN matches the saved PIN, it:

- Retrieves notes data from a content provider (NotesProvider.CONTENT_URI).
- Sets up a SimpleCursorAdapter to display the notes data in a ListView (lview).
- Hides the PIN EditText field and the access button (pinTxt and abutton).

If the user-entered PIN does not match the saved PIN, it shows a toast message indicating that a valid PIN should be entered.

In this challenge the objective is to get the private notes without the PIN. I tried to access the NotesProvider Class and then searched for the `CONTENT_URI` : 

`static final Uri CONTENT_URI = Uri.parse("content://jakhar.aseem.diva.provider.notesprovider/notes");`

I could see that the `NotesProvider` is exported with `android:exported="true"` in the manifest file, it means that other apps or components outside of the app's package can access it. In this case, we can directly access the NotesProvider.CONTENT_URI from adb commands. 

So the final adb command is : 

`adb shell content query --uri content://jakhar.aseem.diva.provider.notesprovider/notes`

Output

```bash
Row: 0 _id=5, title=Exercise, note=Alternate days running
Row: 1 _id=4, title=Expense, note=Spent too much on home theater
Row: 2 _id=6, title=Weekend, note=b333333333333r
Row: 3 _id=3, title=holiday, note=Either Goa or Amsterdam
Row: 4 _id=2, title=home, note=Buy toys for baby, Order dinner
Row: 5 _id=1, title=office, note=10 Meetings. 5 Calls. Lunch with CEO
```

### HardCoding Issues - Part 2

Started looking at the `Hardcode2Activity`. Nothing much, but we can see that it instantiates an object of type DivaJni. 

Overall, this class is responsible for handling user input, passing it to a native method through the DivaJni object, and displaying a toast message based on the result of the method call. The actual logic of the access() method is implemented in the native code (likely written in C or C++) accessed through the DivaJni object.

For this to give us access, the access method should return 0 by the end of execution. 

Looked at the `DivaJni` class 

```java
package jakhar.aseem.diva;

/* loaded from: classes.dex */
public class DivaJni {
    private static final String soName = "divajni";

    public native int access(String str);

    public native int initiateLaunchSequence(String str);

    static {
        System.loadLibrary(soName);
    }
}
```

We can see that a library is being loaded with the string `divajni`. So the actual name of the library will be something of this sort `libdivajni.so`. I was able to find that in the `/lib` directory. 

Since I was using an M1 mac, I didn't have access to gdb or any other tools to decompile and I was too lazy to open up Ghidra. So I used an online decompiler https://dogbolt.org/ which gave me outputs from all these different tools (Huh, that worked out well!)

I searched for the access function, since - `this.djni.access` was being called before the check was made. 

```cpp
//----- (000000000000059C) ----------------------------------------------------
bool __fastcall Java_jakhar_aseem_diva_DivaJni_access(__int64 a1, __int64 a2, __int64 a3)
{
  const char *v3; // x0

  v3 = (const char *)(*(__int64 (__fastcall **)(__int64, __int64, _QWORD))(*(_QWORD *)a1 + 1352LL))(a1, a3, 0LL);
  return strncmp("olsdfgad;lh", v3, 0xBuLL) == 0;
}

```

As we can see this is where the string comparison happens and if it matches with our input, it returns 0 which will give us "Access Granted". 

I entered the string and it worked!

### Input Validation Issues - Part 3


The objective of the challenge is to crash the application and if possible get code execution :P

The starting parts of the challenge were the same, a function called `initiateLaunchSequence` is called and the input string is passed. 

The function is defined in the .so file as mentioned previously. 

This is the code : 

```c
bool __fastcall Java_jakhar_aseem_diva_DivaJni_initiateLaunchSequence(__int64 a1, __int64 a2, __int64 a3)
{
  const char *v3; // x0
  char v5[24]; // [xsp+18h] [xbp-18h] BYREF

  v3 = (const char *)(*(__int64 (__fastcall **)(__int64, __int64, _QWORD))(*(_QWORD *)a1 + 1352LL))(a1, a3, 0LL);
  strcpy(v5, v3);
  if ( v5[0] == 33 )
    v5[0] = 46;
  return strncmp(".dotdot", v5, 7uLL) == 0;
}
```

To crash this is pretty easy, we can see that `strcpy` is used here, which is vulnerable as there are no length checks, so we can get a buffer overflow and overwrite EIP. 

The input string is copied to v5 which takes up 24 bytes the declaration `const char *v3` typically takes up 4 bytes of memory on a 32-bit system and 8 bytes on a 64-bit system for the pointer itself.

So in total 24+8 = 32 bytes is enough to fill EBP, the next 8 bytes will overwrite EIP. To crash the app, any string over 26 characters will do the work. 

The exploitation of this is a very long process requiring debug setups and frida hooking. I am looking forward to covering that in another blog. 

This walkthrough has become longer than I expected! But, I hope you found this helpful!
