---
title: "C2 Communication, Permissions & Persistence: Teaching Android Security Through Real Malware"
date: 2026-06-25
categories: [Android, Malware, Workshop]
tags: [android, malware, c2, reverse-engineering, workshop, security]
---

So it's been a minute. Or six months and honestly, no good reason. Anyway, I just got back from Blackhoodie at Troopers26 where I designed a malware workshop, and setting up functional C2 infrastructure for teaching purposes was way more interesting than expected. Here's what I built.

![Blackhoodie Workshop](/assets/images/Blackhoodie-Workshop.png)

## Why AhMyth RAT and FamilyGuard?

For a workshop, I needed two samples that show different threat models. **AhMyth RAT** is open source. I deliberately chose it because attendees can see the full codebase and understand command-based exploitation from first principles. It's straightforward: C2 sends command, device executes, data returns.

**FamilyGuard** is different. It's a custom-built surveillance app I created specifically to demonstrate a different threat model. Passive, continuous exfiltration rather than command-based. More complex architecture, different feature set. This let me show attendees exactly *why* certain design choices matter and how threat models inform implementation.

Having both meant I could show attendees: "Here's an open-source RAT so you understand command dispatch. Here's custom malware I built to show how a different threat model works." That contrast is valuable. Plus, since FamilyGuard was custom-built, I could explain the design decisions directly. Why I chose this architecture, why certain features matter, what the trade-offs are.

## The C2 Infrastructure

For AhMyth, the C2 setup is command-dispatch based. It needed to work reliably when multiple attendees were connecting their devices simultaneously. Node.js + Socket.IO backend handles device connections and routes commands:

```javascript
['x0000ca', 'x0000fm', 'x0000sm', 'x0000cl', 'x0000cn', 'x0000mc', 'x0000lm'].forEach(event => {
    socket.on(event, (data) => {
        ahmythResults[id].push({ type: event, data, id, time });
    });
});
```

Each command (x0000ca for camera, x0000sm for SMS, etc.) has a handler. AhMyth waits for these commands, executes them, and sends results back.

FamilyGuard works differently. It pushes data continuously without waiting for explicit commands. Different C2 interaction pattern, different feature set. Since I built FamilyGuard, I designed its C2 interaction to show attendees a different approach: instead of command-response, it's publish-subscribe. Data flows continuously. This made the teaching point clear: *malware architecture reflects threat intent*.

## API 24 as a Teaching Point

I deliberately targeted **API 24** for both APKs. Here's why: it's a perfect inflection point to teach how Android's permission model actually works.

Both malware declare dangerous permissions in the manifest (READ_SMS, CAMERA, ACCESS_LOCATION, etc.). On **API 23 and below**, those permissions are granted at install time and just *work*. On **API 24+**, they still work at install time. But on **API 27+**, the permission model shifts harder toward runtime grants, and install-time permissions alone become insufficient.

By using API 24, attendees see permissions *actually functioning as designed*, which makes understanding the exploit mechanism clear. If I'd used API 27, half the commands would fail silently because the runtime permission grants wouldn't happen automatically. That would obscure what's actually happening.

This isn't a limitation; it's a teaching strategy. API 24 is the sweet spot where you can show how both malware abuse permissions without the complexity of modern runtime permission dialogs obscuring the real attack.

## AhMyth: Command-Based Architecture

ConnectionManager is AhMyth's dispatcher. Socket.IO events arrive and map to specific handlers:

```java
socket.on("x0000ca", new Emitter.Listener() {
    public void call(Object... args) {
        cameraManager.startUp(cameraId);
    }
});
```

Once a handler executes, it emits results back:

```java
ioSocket.emit("x0000ca", imageData);
```

Walking through this flow in real-time, from command receipt to execution to data transmission, is where the architecture becomes clear. Attendees see that `x0000ca` is just a string key mapping to a function. It's not magic.

## FamilyGuard: Custom-Built Threat Model

FamilyGuard demonstrates a different approach that I built specifically for this. Rather than waiting for C2 commands, it continuously collects and exfiltrates data. Different feature set, different architecture, different threat profile.

Since I designed and built FamilyGuard, I could explain directly: "I chose this architecture because passive surveillance doesn't need command dispatch. It needs persistent collection and low-latency exfiltration." Attendees could see the *reasoning* behind the design, not just the code.

This is important for attendees to see: **not all mobile malware is RAT-based**. Some malware is purely surveillance. It runs silently in the background, collects everything, and sends it continuously. No commands, no interactive control. Just persistent data theft. By building FamilyGuard with this specific threat model in mind, I could teach attendees why different threats require different architectures.

## Persistence: Both Take Different Approaches

Both use **`START_STICKY`** in their background service. If killed, they restart automatically.

Both use **`BOOT_COMPLETED`** to launch on device boot.

But FamilyGuard's persistence strategy is stricter because it has no C2 fallback. If it stops running, it stops collecting data. AhMyth can wait for commands indefinitely. FamilyGuard has to stay active constantly.

This distinction taught attendees something real: persistence mechanisms vary based on threat model. A passive surveillance malware has different persistence requirements than a command-based RAT. And since I built FamilyGuard, I could explain exactly why I made those design choices.

## The Live Demo Approach

The workflow for AhMyth:

1. Install APK → service starts + BOOT_COMPLETED registered
2. Device connects to C2 → appears in admin UI
3. Trigger command (e.g., x0000ca) → camera executes
4. Check logcat → see the execution in real-time
5. Admin UI displays stolen data

For FamilyGuard, the demo showed:

1. Install APK → service runs continuously
2. Device connects to server → begins exfiltration
3. No commands needed—data flows automatically
4. Logcat shows continuous collection
5. Server receives ongoing data stream

Showing both workflows made clear that **there's no single "malware attack flow."** Different threats operate differently.

## Why This Lab Design Works

The interesting part of setting this up was **intentional constraint and contrast**. I could've built more sophisticated systems, but for teaching, simplicity is better. And using both an open-source RAT I could dissect line-by-line and custom malware I designed meant attendees saw the full spectrum.

Each choice was deliberate:
- **Open source (AhMyth)** — attendees can read full source, understand design decisions from established code
- **Custom-built (FamilyGuard)** — demonstrates different threat model, I can explain the architectural choices directly
- **API 24** — permission model works as intended, teaching is clear
- **Simple C2** — focuses on concepts, not infrastructure complexity

## Defensive Implications

From a defense perspective, this lab teaches:

1. **Different threats require different detection**. RAT activity looks different from passive surveillance. Defenders need to understand both patterns.

2. **Architecture reflects intent**. How malware is built tells you what it's trying to do. A command-dispatch RAT behaves differently than continuous exfiltration.

3. **Permissions are exploited identically**. Whether it's command-based or passive, malware abuses the same permissions. Understanding permission behavior is critical regardless of malware type.

4. **Persistence mechanisms are invisible**. Both keep running after app close. Detection requires understanding service lifecycle, not UI state.

5. **Threat models vary**. Not all mobile malware is a remote access trojan. Some is pure surveillance. Defenders need to detect both.

## What It Came Down To

The interesting part wasn't just understanding existing malware. The interesting part was designing a lab that teaches multiple threat models. Using open-source AhMyth to show command-based architecture, and custom FamilyGuard to show how different threat intents shape different designs. Choosing API 24 deliberately, building one sample to specification, constraining the C2 to essentials, structuring the demos to show different patterns. Those are the setup decisions that made the workshop work.
