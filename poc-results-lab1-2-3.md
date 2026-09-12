# POC Results : Labs 1, 2, and 3



---

## Lab 1: Private Image Model (ViT) with Transit Gateway

### What we wanted to prove
A machine learning model should never be open to the public internet. Only trusted internal services should be able to use it.

### What we built
- Two separate networks (VPCs) in AWS.
- One network has the model server (a ViT image model running behind FastAPI).
- The other network has a "client" — a test computer that sends images to the model.
- These two networks are connected only through a **Transit Gateway**, which acts like a private bridge.
- Neither network has an internet gateway. This means there is no door open to the public internet at all.
- The model file is downloaded from S3 using a private connection (S3 Endpoint), so even downloading the model does not need the internet.

### What we tested
1. **Test from the client (inside the private network):** We sent an image to the model and asked for a prediction. It worked. The model replied with a label like "cat" or "dog."
2. **Test from the public internet (outside):** We tried to reach the model from a normal computer on the internet. This did not work. The request just hung and timed out, because there is no path leading to the model from outside.

### Result
 The model works fine for the internal client.
 The model is completely unreachable from the public internet.
 This proves the network is private "by design," not just blocked by a setting that could be turned off by mistake.

---

## Lab 2: Two Regions with Automatic Backup (BGP Failover) + Monitoring

### What we wanted to prove
If the model server in one region goes down, traffic should automatically move to a second region, without a person needing to do anything.

### What we built
- The same voice-to-text model (Whisper) running in **two different AWS regions**: one main (Region A) and one backup (Region B).
- A routing system using **BGP** (a method routers use to decide the best path for traffic). Normally, all traffic goes to Region A.
- A health-check script that watches Region A. If Region A stops responding, the script tells the network "stop sending traffic here," and traffic automatically flows to Region B instead.
- **Prometheus** to collect data about how the model servers are performing (speed, number of requests, whether they are up or down).
- **Grafana** to show that data on dashboards you can watch in real time.

### What we tested
1. Sent a steady stream of requests and watched them all go to Region A (the main region).
2. Manually stopped the model server in Region A, to simulate a crash.
3. Watched the system detect the failure within about 15 seconds and switch all traffic to Region B, with no manual action.
4. Watched the Grafana dashboard show this switch happening — the Region A line drops to zero, and the Region B line goes up, both at the same moment.
5. Restarted Region A and watched traffic move back automatically.

### Result
 When the main region fails, the backup region takes over on its own, in seconds.
 The dashboards clearly show the exact moment the switch happens.
 No customer requests were lost — they were just served by the other region instead.

---

## Lab 3: Secure Voice Model Using an Encrypted Tunnel (IPsec)

### What we wanted to prove
Voice data (which can be private, like patient notes or financial calls) should never travel across the internet in a way that someone could read if they intercepted it.

### What we built
- A model server (Whisper, for turning speech into text) placed in a private network with no public access.
- A simulated "outside" location (like an office or hospital) that sends audio files to the model.
- A secure, encrypted tunnel between the outside location and AWS, called an **IPsec VPN**. This scrambles all the data so it cannot be read while traveling over the internet.

### What we tested
1. Sent an audio file from the outside location, through the tunnel, to the model, and got back the written text.
2. While sending the audio, we recorded the network traffic using a tool called `tcpdump`.
3. Looked at the recorded traffic to check what it actually looked like on the wire.

### Result
 The audio was successfully transcribed to text.
 The recorded traffic showed only scrambled, encrypted data (called "ESP" packets) — never the real audio or any readable text.
 This proves that even if someone captured the network traffic in the middle, they could not read the voice data or the transcript.

---

## Overall Summary

| Lab | Main Idea | Proven Result |
|---|---|---|
| 1 | Keep a model completely private | Model unreachable from internet, but works for the trusted client |
| 2 | Auto-recover if one region fails | Traffic switches to backup region in seconds, shown live on a dashboard |
| 3 | Keep voice data safe in transit | All data travels encrypted; nothing readable is exposed on the network |

All three POCs worked as expected. Together, they show a model setup that is **private**, **reliable during failures**, and **safe for sensitive data**.
