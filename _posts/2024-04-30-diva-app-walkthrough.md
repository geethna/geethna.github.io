---
title: DIVA Android app Walkthrough - Part1
date: 2024-04-30 00:00:00 +0530
categories: [Android]
tags: [android, re]
---

As a part of my journey of learning android reversing, I started exploring different intentionally vulnerable apks for practise. The first one I found was a simply app named [sieve](https://github.com/tanprathan/sievePWN). But I realised a writeup for that would be too short. 

Then I came across the [DIVA](https://github.com/payatu/diva-android) apk by Payatu. This Damn insecure and vulnerable App) is an App intentionally designed to be insecure. It covers common vulnerabilities in Android apps ranging from insecure storage, input validation to access control issues.

It contains a total of 13 vulnerabilities. I will be covering each of these issues section by section. 


### LogActivity


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


### HardCoding - Part1

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

### InsecureDataStorage Part 1

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

### InsecureDataStorage Part 2

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

### InsecureDataStorage Part 3

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

### InsecureDataStorage Part 4

The same functionality but this time it is writing to external storage. I was using an emulated device Pixel 4 - API 23 (Android 6.0) but when I tried to save the credentials, I kept getting an error. When I checked the logs, it was : 

`Diva    : File error: /storage/emulated/0/.uinfo.txt: open failed: EACCES (Permission denied)`

It could be due to the app does not having permission to write to external storage. This is likely because starting from Android 6.0 (API level 23), apps need to request runtime permission from the user to access certain dangerous permissions, including WRITE_EXTERNAL_STORAGE.

So I was unable to get this working, but from what I know this file would be saved in the `/sdcard/` directory. So we can use the adb command to navigate to that and cat the contents of `uinfo.txt`. 


This walkthrough has become longer than I expected, so I decided to make a Part2 and add the rest of the challenges in it. 

Hope you found this helpful, see you in the next blog! 


