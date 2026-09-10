## Check Routing and Default Gateway

# Scenario 2 — Wrong Default Gateway

## 🎯 Goal

Create a **wrong default gateway** on an EC2 instance, observe the network problem, troubleshoot it, and fix it.

> ⚠️ **Warning:** Changing the default gateway can disconnect your SSH session. Use EC2 Serial Console, Session Manager, or another recovery method before starting this lab.

---

## 1. Check Current Network Route

First check the current routing table:

```bash
ip route
```

Expected:

```text
default via 172.31.32.1 dev ens5
```

Here:

* `default` → route used for destinations outside the local network
* `172.31.32.1` → default gateway
* `ens5` → network interface

---

# 2. Create the Problem

Replace the correct gateway with an incorrect gateway:

```bash
sudo ip route replace default via 172.31.32.254 dev ens5
```

Check the route:

```bash
ip route
```

You should now see:

```text
default via 172.31.32.254 dev ens5
```

The default gateway is now incorrect.

---

# 3. Observe the Problem

Test the wrong gateway:

```bash
ping -c 5 172.31.32.254
```

Then test Internet connectivity:

```bash
ping -c 5 8.8.8.8
```

You may see:

```text
Destination Host Unreachable
```

or:

```text
100% packet loss
```

If you are connected through SSH, your SSH session may also disconnect:

```text
client_loop: send disconnect: Connection reset
```

After that, a new SSH connection may fail:

```text
ssh: connect to host <EC2-IP> port 22: Connection timed out
```

---

# 4. Troubleshoot the Problem

Start with the routing table:

```bash
ip route
```

Look for:

```text
default via 172.31.32.254 dev ens5
```

The gateway is wrong.

Test the expected gateway:

```bash
ping -c 5 172.31.32.1
```

Then test Internet connectivity:

```bash
ping -c 5 8.8.8.8
```

Check the interface:

```bash
ip -br addr
```

---

# 5. Fix the Problem

Restore the correct default gateway:

```bash
sudo ip route replace default via 172.31.32.1 dev ens5
```

Check the routing table:

```bash
ip route
```

Expected:

```text
default via 172.31.32.1 dev ens5
```

---

# 6. Verify the Fix

Test Internet connectivity:

```bash
ping -c 10 8.8.8.8
```

Expected:

```text
10 packets transmitted, 10 received, 0% packet loss
```

Then test SSH again from your local machine:

```bash
ssh -i "server-a.pem" ubuntu@<EC2-PUBLIC-IP>
```

---

# 🔧 Troubleshooting Flow

```text
Network Problem
      │
      ▼
   ip route
      │
      ▼
Check default gateway
      │
      ▼
Gateway incorrect?
      │
     Yes
      │
      ▼
Restore correct gateway
      │
      ▼
   ip route
      │
      ▼
ping -c 10 8.8.8.8
      │
      ▼
Network restored
```

---

# 📌 Commands Used

### Check routes

```bash
ip route
```

### Check IP/interface

```bash
ip -br addr
```

### Change default gateway

```bash
sudo ip route replace default via 172.31.32.254 dev ens5
```

### Restore default gateway

```bash
sudo ip route replace default via 172.31.32.1 dev ens5
```

### Test connectivity

```bash
ping -c 5 172.31.32.1
ping -c 10 8.8.8.8
```

---

# 🧠 Key Lesson

A **default gateway** is used when the destination is outside the local network.

If the default gateway is wrong:

```text
EC2
 │
 │ Wrong Gateway
 ▼
172.31.32.254
 │
 ✖ Network connectivity problem
```

Correct configuration:

```text
EC2
 │
 │ Default Gateway
 ▼
172.31.32.1
 │
 ▼
Internet
```

### DevOps Troubleshooting Pattern

```text
Check → Identify → Fix → Verify
```

For network issues:

```bash
ip route
ping <gateway>
ping 8.8.8.8
```

Then restore the correct route.

---

## ⚠️ Important

This lab intentionally changes the network route.

If you are connected through your **only SSH session**, changing the default gateway can disconnect you immediately.

Before creating this fault, make sure you have a recovery method such as:

* EC2 Serial Console
* AWS Systems Manager Session Manager
* Another administrative access path

The `ip route replace` change is generally runtime-only, so a reboot may restore the route if it is normally configured by the instance's networking system. Do not rely on reboot as your only recovery method.
