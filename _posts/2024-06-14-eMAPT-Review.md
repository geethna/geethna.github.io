---
title: Achieving eMAPT Certification - My Experience and Insights
date: 2024-06-14 00:00:00 +0530
categories: [Android, RE, Certification]
tags: [android, emapt, certification]
---


Hello all! So this month I got some time to attempt the Mobile application pentesting certification by eLearnSecurity also known as [eMAPT](https://security.ine.com/certifications/emapt-certification/). 

I passed the exam on my first attempt a few days ago and wanted to share my experience with the course and the exam. 

### Background 

Around October last year, I found myself getting interested in android vulnerabilities and exploitation techniques. I started out by trying out a few HTB labs and learning through other Android RE courses. But at first, I had to deepen my understanding of Android architecture, build processes and fundamentals. So I went on looking for certifications or courses to aid my learning. 

That's when I came across INE's Mobile application pentesting course and the reviews were great. People were particularly appreciating the theoretical detailing and the example labs and apks. 

So I decided to prepare for the certification which will help me deepen my understanding of Mobile Pentesting. 

### Course and Practice Materials

As preparation for the certification, they majorly used INE's course materials. I really liked how elaborate the theory part was and each section had relevant labs covering practical real-world apks. 

The course also had a section for setting up the test environment and details regarding device rooting techniques. I had already set up these environments on emulators and physical devices, so I didn't have to look through this that much. The sections - Reversing APKs, Android Application Fundamentals, Device and Data Security and Static Code Analysis were incredibly in-depth and technical. These were the ones to look at particularly throughout the whole course. The labs, especially in the last two sections of the course were pretty neat and I would say helped me a lot during the exam too. Having said this, I have to admit that the course is slightly outdated and I wish it covered more recent techniques and tools. 

Apart from the INE's course, I also explored other materials online. First one was [sievePWN](https://github.com/tanprathan/sievePWN). This one was a password manager apk with particularly 4 vulnerabilities to exploit and was very helpful with respect to static analysis. This also gave more clarity in terms of hooking processes dynamically through frida. 

Even Though, slightly old, I also practiced exploiting the [DIVA](https://github.com/payatu/diva-android) app by Payatu. This will get you covered on most of the basics of exploitations and adb commands. I also prepared a walkthrough for this in [blog](https://geethna.github.io/posts/diva-app-walkthrough/).


I wenth through a lot of reviews of the exam and course and understood that we have to learn android app development too to create an exploit app during the exam. So I also tried to exploit other intentionally vulnerable apps which I could found online. One such was the [InsecureShop](https://github.com/hax0rgb/InsecureShop). I exploited each of the vulnerabilities in this through an exploit apk and I think this helped me to understand things in a much better way.

Now for those you who want to attempt the certification and you don't want to take the INE's course, I came across an incredibly detailed resource - emapt preparation [notes](https://drive.google.com/file/d/1K_xnDKMhlV1aJqXsq4lXiCcliiGvs877/view) by [John A Santos](https://www.linkedin.com/in/joas-antonio-dos-santos/)

I also went through Android Reverse Engineering through MaddieStone's [course](https://www.ragingrock.com/AndroidAppRE/) to get an in-depth idea and creative tricks. 


### The Exam

They provide you with 2 vulnerable applications and you have to create an android application which will exploit the vulnerabilities in both these apps. You get a total of 7 days to create the application, exploit and submit the developed application for review. And no, you do not have to write a Pentesting report for this exam. 

I wouldn't say the exam is very difficult, if you cover the course materials and labs well, it should be good. Finding the vulnerabilities itself took much less time but finding creative ways to exploit and develop the application takes up most of it. 

Overall, the eMAPT certification was a rewarding journey that significantly deepened my understanding of mobile application pentesting. 



