## Check Routing and Default Gateway

# Scenario 2 — Wrong Default Gateway

## 🎯 Objective

In this lab, we will intentionally configure a **wrong default gateway** on an AWS EC2 instance.

We will observe:

```text
Create Network Problem
        ↓
Wrong Default Gateway
        ↓
SSH Connection Disconnects
        ↓
SSH Timeout
        ↓
Recover EC2 from AWS Console
        ↓
Reboot EC2
        ↓
SSH Again
        ↓
Check Route
        ↓
Verify Network
        ↓
Scenario Fixed
```

> ⚠️ **Warning:** This lab intentionally breaks network connectivity. Do this only on a test/lab EC2 instance.

---

# 1. Connect to EC2

From your local machine:

```bash
ssh -i "server-a.pem" ubuntu@<EC2-PUBLIC-IP>
```

Example:

```bash
ssh -i "server-a.pem" ubuntu@ec2-35-91-138-101.us-west-2.compute.amazonaws.com
```

Check that you are connected:

```bash
whoami
```

Expected:

```text
ubuntu
```

---

# 2. Check Current Network Configuration

Check the network interface:

```bash
ip -br addr
```

Example:

```text
lo       UNKNOWN    127.0.0.1/8
ens5     UP         172.31.33.129/20
```

Check the routing table:

```bash
ip route
```

Expected:

```text
default via 172.31.32.1 dev ens5
172.31.32.0/20 dev ens5 proto kernel scope link src 172.31.33.129
```

### Important

The current correct default gateway is:

```text
172.31.32.1
```

The network interface is:

```text
ens5
```

---

# 3. Test Network Before Creating the Problem

First test the gateway:

```bash
ping -c 5 172.31.32.1
```

Then test Internet connectivity:

```bash
ping -c 5 8.8.8.8
```

Expected:

```text
5 packets transmitted, 5 received, 0% packet loss
```

Everything is working.

---

# 4. Create the Network Problem

Now intentionally replace the correct default gateway with a wrong gateway.

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

The problem has been created.

---

# 5. Test the Broken Network

Try:

```bash
ping -c 5 8.8.8.8
```

The connection may fail.

If you are connected through SSH, your SSH session may suddenly disconnect.

You may see:

```text
client_loop: send disconnect: Connection reset
```

Your terminal may return to your local machine.

---

# 6. Try SSH Again

Try connecting again:

```bash
ssh -i "server-a.pem" ubuntu@<EC2-PUBLIC-IP>
```

You may receive:

```text
ssh: connect to host <EC2-PUBLIC-IP> port 22: Connection timed out
```

### What happened?

The EC2 instance may still be **running**.

The problem is the network route.

```text
EC2
 │
 │ Wrong Default Gateway
 ▼
172.31.32.254
 │
 ✖ Network connectivity
 │
 ✖ SSH
```

The EC2 instance itself has not necessarily stopped.

---

# 7. Open AWS EC2 Console

Because SSH is no longer working, go to AWS.

Open:

**AWS Console → EC2 → Instances**

Find your EC2 instance.

Check:

```text
Instance state: Running
Status check: 2/2 checks passed
```

This is an important observation.

The instance can be:

```text
EC2 = Running
SSH  = Not Working
```

So the problem is likely network connectivity rather than the instance being stopped.

---

# 8. Recover the EC2 Instance

There are several recovery methods.

For this lab, we will use:

```text
AWS EC2 Console
      ↓
Reboot Instance
      ↓
Wait for Status Checks
      ↓
Try SSH Again
```

> **Important:** EC2 Serial Console or Systems Manager Session Manager can provide a more direct recovery path if available. Reboot is suitable here because the route was changed using `ip route`, which is normally a runtime change.

---

# 9. Reboot the EC2 Instance

From AWS:

```text
EC2
 ↓
Instances
 ↓
Select your instance
 ↓
Instance state
 ↓
Reboot instance
```

AWS will ask for confirmation.

Click:

```text
Reboot
```

---

# 10. Wait for the EC2 Instance

Wait until the instance returns to:

```text
Instance state: Running
```

Also wait for:

```text
Status checks: 2/2 checks passed
```

Do not immediately try SSH while the instance is still rebooting.

---

# 11. Connect to EC2 Again

From your local terminal:

```bash
ssh -i "server-a.pem" ubuntu@<EC2-PUBLIC-IP>
```

If the route was only changed at runtime, the normal network configuration should have been applied again after reboot.

---

# 12. Check the Route After Reboot

Run:

```bash
ip route
```

Expected:

```text
default via 172.31.32.1 dev ens5
```

The incorrect route:

```text
default via 172.31.32.254 dev ens5
```

should no longer be present.

---

# 13. Verify the Gateway

Run:

```bash
ping -c 5 172.31.32.1
```

Expected:

```text
5 packets transmitted, 5 received, 0% packet loss
```

---

# 14. Verify Internet Connectivity

Run:

```bash
ping -c 10 8.8.8.8
```

Expected:

```text
10 packets transmitted, 10 received, 0% packet loss
```

The network is working again.

---

# 15. Verify SSH

Exit the EC2 instance:

```bash
exit
```

Then connect again:

```bash
ssh -i "server-a.pem" ubuntu@<EC2-PUBLIC-IP>
```

SSH should now work normally.

---

# 16. Final Scenario Verification

After reconnecting:

### Check interface

```bash
ip -br addr
```

### Check route

```bash
ip route
```

Expected:

```text
default via 172.31.32.1 dev ens5
```

### Check gateway

```bash
ping -c 5 172.31.32.1
```

### Check Internet

```bash
ping -c 10 8.8.8.8
```

### Check DNS

```bash
ping -c 5 google.com
```

All tests should work.

---

# 17. Complete Incident Flow

```text
                 NORMAL STATE
                      │
                      ▼
              ip route
                      │
                      ▼
       default via 172.31.32.1
                      │
                      ▼
             Network Working
                      │
                      ▼
        CREATE THE PROBLEM
                      │
                      ▼
       Replace Gateway with
             172.31.32.254
                      │
                      ▼
             Network Broken
                      │
                      ▼
             SSH Disconnect
                      │
                      ▼
             SSH Timeout
                      │
                      ▼
              AWS Console
                      │
                      ▼
             EC2 Running?
                      │
                     Yes
                      │
                      ▼
             Reboot Instance
                      │
                      ▼
          Wait for 2/2 Checks
                      │
                      ▼
                SSH Again
                      │
                      ▼
                ip route
                      │
                      ▼
       default via 172.31.32.1
                      │
                      ▼
            Ping Gateway
                      │
                      ▼
           Ping 8.8.8.8
                      │
                      ▼
             Network Fixed
```

---

# 18. Commands Used in This Scenario

## Check interface

```bash
ip -br addr
```

## Check routing table

```bash
ip route
```

## Test correct gateway

```bash
ping -c 5 172.31.32.1
```

## Create wrong gateway

```bash
sudo ip route replace default via 172.31.32.254 dev ens5
```

## Test Internet

```bash
ping -c 10 8.8.8.8
```

## Restore manually if you have console access

```bash
sudo ip route replace default via 172.31.32.1 dev ens5
```

---

# 19. Troubleshooting Commands

When SSH/network connectivity fails:

```bash
ip -br addr
ip route
ip -s link
```

Test gateway:

```bash
ping -c 5 172.31.32.1
```

Test external IP:

```bash
ping -c 5 8.8.8.8
```

Test DNS:

```bash
ping -c 5 google.com
```

---

# 20. Problem → Cause → Fix

| Problem                | Cause                                              | Fix                   |
| ---------------------- | -------------------------------------------------- | --------------------- |
| SSH disconnected       | Wrong default route                                | Restore correct route |
| SSH timeout            | Network path broken                                | Check `ip route`      |
| Internet unavailable   | Wrong gateway                                      | Restore gateway       |
| EC2 still running      | OS/network issue, not necessarily instance failure | Check route           |
| After reboot SSH works | Runtime route was reapplied                        | Verify `ip route`     |

---

# 🧠 DevOps Troubleshooting Pattern

When SSH suddenly stops working:

```text
1. Check EC2 state
       ↓
2. Check AWS status checks
       ↓
3. Check network route
       ↓
4. Check default gateway
       ↓
5. Check connectivity
       ↓
6. Recover access
       ↓
7. Fix configuration
       ↓
8. Verify
```

The key command in this scenario is:

```bash
ip route
```

It can quickly show whether the EC2 instance has the expected default route.

---

# ✅ Final Result

Before the problem:

```text
default via 172.31.32.1 dev ens5
```

Problem created:

```text
default via 172.31.32.254 dev ens5
```

Result:

```text
❌ Network connectivity
❌ SSH connection
```

After EC2 recovery:

```text
default via 172.31.32.1 dev ens5
```

Result:

```text
✅ Network connectivity
✅ Internet connectivity
✅ SSH connection
```

## Final Lesson

> **A running EC2 instance does not always mean that SSH will work. Network routing problems can make a healthy instance unreachable.**

```text
Check → Create Problem → Observe → Recover → Fix → Verify
```
