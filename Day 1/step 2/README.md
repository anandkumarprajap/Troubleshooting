## Check Routing and Default Gateway

# 🚨 Scenario 2 — Wrong Default Gateway

## 🎯 Goal

Understand what happens when an EC2 instance has an **incorrect default gateway**.

In this lab, we will:

1. Check the correct gateway.
2. Replace it with a wrong gateway.
3. Test connectivity.
4. Troubleshoot the problem.
5. Restore the correct gateway.
6. Understand what happens if SSH gets disconnected.

> ⚠️ **Warning:** Changing the default route can immediately break network connectivity, including your current SSH session. Use a test EC2 instance and have a recovery method available.

---

# 1. Check the Current Route

First, check the routing table:

```bash
ip route
```

Example:

```text
default via 172.31.32.1 dev ens5
```

Here:

```text
172.31.32.1
```

is the **default gateway**.

Record the correct gateway before making any changes.

```text
Correct Gateway:
172.31.32.1
```

---

# 2. What Is a Default Gateway?

The **default gateway** is the router that Linux uses when it doesn't have a more specific route for the destination.

For example:

```text
EC2
 │
 ▼
Default Gateway
 │
 ▼
Internet
```

If the default gateway is incorrect:

```text
EC2
 │
 ▼
Wrong Gateway ❌
 │
 X
Internet
```

The EC2 instance may lose external network connectivity.

---

# 3. Create the Problem

Replace the existing default route with an intentionally wrong gateway:

```bash
sudo ip route replace default via 172.31.32.254 dev ens5
```

Now check the route:

```bash
ip route
```

You should see:

```text
default via 172.31.32.254 dev ens5
```

The correct gateway was:

```text
172.31.32.1
```

The new gateway is:

```text
172.31.32.254
```

So we have intentionally created a wrong route.

---

# 4. Test the Wrong Gateway

Try to reach the wrong gateway:

```bash
ping -c 5 172.31.32.254
```

You should normally see failures or timeouts if that address is not a usable gateway.

---

# 5. Test the Correct Gateway

Now test the gateway that you recorded earlier:

```bash
ping -c 5 172.31.32.1
```

The correct gateway should normally be reachable.

This helps us understand the problem:

```text
Wrong Gateway
172.31.32.254
      ↓
    ❌

Correct Gateway
172.31.32.1
      ↓
    ✅
```

---

# 6. Test Internet Connectivity

Now test an external IP:

```bash
ping -c 5 8.8.8.8
```

Because the default route is incorrect, external connectivity should fail.

Expected behavior:

```text
EC2
 ↓
Wrong Default Gateway ❌
 ↓
Internet
 ↓
Connection fails
```

---

# 7. Troubleshooting the Incident

Now pretend you **don't know what caused the problem**.

The incident is:

> 🚨 **EC2 server cannot reach the Internet.**

Start troubleshooting.

---

## Step 1 — Check the Routing Table

```bash
ip route
```

Look for:

```text
default via <gateway> dev <interface>
```

Example:

```text
default via 172.31.32.254 dev ens5
```

Ask:

> Is this the correct gateway?

---

## Step 2 — Check the Gateway

Test the configured gateway:

```bash
ping -c 5 172.31.32.254
```

If it fails, investigate the route/gateway.

---

## Step 3 — Test the Known Correct Gateway

If you recorded the original gateway:

```bash
ping -c 5 172.31.32.1
```

If the correct gateway is reachable while the configured gateway is not, the default route is likely incorrect.

---

## Step 4 — Test External Connectivity

```bash
ping -c 5 8.8.8.8
```

If this fails:

```text
Internet unreachable
        ↓
Check routing
        ↓
Check default gateway
```

---

# 8. Fix the Problem

Restore the correct gateway:

```bash
sudo ip route replace default via 172.31.32.1 dev ens5
```

Check the route:

```bash
ip route
```

You should see:

```text
default via 172.31.32.1 dev ens5
```

---

# 9. Verify the Fix

Test the Internet:

```bash
ping -c 10 8.8.8.8
```

Expected:

```text
10 packets transmitted
10 received
0% packet loss
```

Your network should now be working again.

---

# 10. Complete Troubleshooting Flow

```text
Internet unreachable
        ↓
     ip route
        ↓
Default gateway incorrect ❌
        ↓
   Ping gateway
        ↓
Gateway unreachable
        ↓
Restore correct gateway
        ↓
     ip route
        ↓
   Ping 8.8.8.8
        ↓
   Connectivity OK ✅
```

---

# 11. ⚠️ What Happens to SSH?

This is the most important lesson from this lab.

You may be connected to EC2 using:

```bash
ssh -i your-key.pem ubuntu@<EC2-IP>
```

When you change the default route:

```bash
sudo ip route replace default via 172.31.32.254 dev ens5
```

your SSH connection may stop working.

You may see:

```text
client_loop: send disconnect: Connection reset
```

or:

```text
ssh: connect to host <EC2-IP> port 22: Connection timed out
```

This happens because SSH itself needs network connectivity.

```text
Your Computer
      │
      │ SSH
      ▼
     EC2
      │
      ▼
Wrong Default Gateway ❌
      │
      X
Network connectivity
```

---

# 12. 🚑 Recovery If SSH Is Lost

If your SSH session is disconnected, you cannot simply SSH back in to fix the route because the network route is already broken.

You need another recovery method.

Possible options include:

* EC2 Serial Console
* AWS Systems Manager Session Manager, if configured
* Another administrative/recovery path
* Rebooting the instance when the route change is only runtime configuration

---

# 13. 🔄 Reboot Recovery

The route was changed using:

```bash
sudo ip route replace default via 172.31.32.254 dev ens5
```

A manually added/changed runtime route is generally not persistent across a reboot.

Therefore, rebooting the instance may restore the normal network configuration provided by the instance's networking setup.

From the AWS Console:

```text
EC2
 ↓
Instances
 ↓
Select Instance
 ↓
Instance State
 ↓
Reboot Instance
```

Wait for the instance to become healthy.

Then try SSH again:

```bash
ssh -i "server-a.pem" ubuntu@<EC2-PUBLIC-DNS>
```

After connecting:

```bash
ip route
```

Check that the correct route has returned.

Then test:

```bash
ping -c 10 8.8.8.8
```

---

# 14. 🧠 Important Commands

### Show routing table

```bash
ip route
```

### Replace default gateway

```bash
sudo ip route replace default via <gateway-ip> dev <interface>
```

### Test gateway

```bash
ping -c 5 <gateway-ip>
```

### Test Internet

```bash
ping -c 5 8.8.8.8
```

---

# 15. 📌 Key Concepts

## Default Route

A default route is used when no more specific route matches the destination.

Example:

```text
default via 172.31.32.1 dev ens5
```

Meaning:

```text
default
   ↓
Use gateway 172.31.32.1
   ↓
Through interface ens5
```

---

## Gateway

A gateway is the network device/router through which traffic leaves the local network.

```text
EC2
 ↓
Gateway
 ↓
Other Networks
 ↓
Internet
```

---

## `ip route`

The command:

```bash
ip route
```

is used to view the Linux routing table.

It is one of the first commands to check when troubleshooting network connectivity.

---

# 16. 🎯 DevOps Troubleshooting Mindset

When an EC2 instance cannot reach the Internet, don't immediately assume the problem is AWS.

Follow a structured process:

```text
Symptom
  ↓
Collect information
  ↓
Check interface
  ↓
Check route
  ↓
Check gateway
  ↓
Check external connectivity
  ↓
Identify root cause
  ↓
Fix
  ↓
Verify
```

For this scenario:

```text
Problem:
Internet unreachable
       ↓
Check:
ip route
       ↓
Found:
Wrong default gateway ❌
       ↓
Fix:
Restore correct gateway
       ↓
Verify:
ping 8.8.8.8
       ↓
Result:
Network working ✅
```

---

# 17. ⚠️ Important Lab Lesson

**Do not intentionally change the default route on your only SSH connection without a recovery plan.**

Before performing this lab, make sure you have at least one recovery option:

```text
Option 1 → EC2 Serial Console
Option 2 → Systems Manager Session Manager
Option 3 → Another recovery/admin connection
Option 4 → Reboot if the change is runtime-only
```

---

# 18. 📝 Quick Revision

```text
Correct:
default via 172.31.32.1 dev ens5
```

Change to:

```text
Wrong:
default via 172.31.32.254 dev ens5
```

Test:

```bash
ip route
```

```bash
ping -c 5 172.31.32.254
```

```bash
ping -c 5 8.8.8.8
```

Fix:

```bash
sudo ip route replace default via 172.31.32.1 dev ens5
```

Verify:

```bash
ip route
```

```bash
ping -c 10 8.8.8.8
```

---

# 🏆 Final Takeaway

> **A wrong default gateway can make an EC2 instance lose external network connectivity.**

The basic troubleshooting approach is:

```text
ip route
   ↓
Check default gateway
   ↓
Ping gateway
   ↓
Ping external IP
   ↓
Fix route
   ↓
Verify connectivity
```

### Remember

> **Network problem → Check the route first.**

And most importantly:

> **Never break the default route on your only SSH connection unless you have a recovery method.**
