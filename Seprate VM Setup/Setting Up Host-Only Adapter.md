### Prerequisites
- Download the Rocky Linux 10 iso image (7GB approx) or 1GB iso image(it will not have any gui only cli) and a virtualization tool (eg Oracle VirtualBox)
- Create and complete the installation of the Rocky Linux VM.
---

This setup will let your **host computer** (Windows/macOS/Linux) talk to your **Rocky Linux VM** directly using a fixed IP like `192.168.56.101`.
## 🧠 What we’ll do

1. Add a Host-Only network in VirtualBox.
    
2. Attach it to your Rocky Linux 10 VM.
    
3. Find the correct network adapter name inside Rocky Linux.
    
4. Give it a **static IP** (so it doesn’t change).
    
5. Check everything works.
---

## ⚙️ Step 1: Create a Host-Only Network in VirtualBox

1. **Open VirtualBox → File → Tools → Network Manager (or Host Network Manager).**
    
2. Click **Create** (the + icon).
    
3. You’ll see something like this appear:
    
    - **Name:** `vboxnet0`
        
    - **IPv4 Address:** `192.168.56.1`
        
    - **IPv4 Network Mask:** `255.255.255.0`
        
4. Make sure **DHCP Server** is **disabled** (we’ll use static IPs manually).
    
5. Click **Apply** or **OK**.

✅ Now your host (your real PC) has an internal network card `vboxnet0` with IP `192.168.56.1`.

---

## 🖥️ Step 2: Attach this Host-Only adapter to your Rocky Linux VM

1. Select your **Rocky Linux 10 VM** in VirtualBox.
    
2. Click **Settings → Network**.
    
3. You’ll see **Adapter 1** (usually NAT). Keep that for internet if you want.
    
4. Click on **Adapter 2** →
    
    - Check **Enable Network Adapter**.
        
    - Choose **Attached to: Host-only Adapter**.
        
    - In **Name**, pick the one you just made — for example, `vboxnet0`.
        
5. Click **OK**.
    

✅ Now your VM has two adapters:

- Adapter 1 (NAT) → for Internet.
    
- Adapter 2 (Host-Only) → private connection to host.
    

---

## 🧾 Step 3: Start your VM and find the interface name

Inside your Rocky Linux terminal, run:

```bash
ip -brief link
```

You’ll see output like:

```
lo               UNKNOWN        127.0.0.1/8
enp0s3           UP             10.0.2.15/24
enp0s8           DOWN
```

👉 Here, `enp0s8` is your second (Host-Only) network adapter.  
Write that name down — we’ll use it in the next step.

---

## 🌐 Step 4: Assign a static IP using NetworkManager (`nmcli`)

### 4.1 Create a new connection for the host-only adapter:

Replace `enp0s8` with your actual adapter name.

```bash
sudo nmcli connection add \
  type ethernet \
  con-name HostOnly \
  ifname enp0s8 \
  ipv4.addresses 192.168.56.101/24 \
  ipv4.method manual \
  connection.autoconnect yes
```

What this means:

- `type ethernet` → it’s a wired connection.
    
- `con-name HostOnly` → we name it “HostOnly.”
    
- `ifname enp0s8` → use that adapter.
    
- `ipv4.addresses` → static IP `192.168.56.101` with mask `/24` (255.255.255.0).
    
- `manual` → no DHCP, we set IP manually.
    
- `autoconnect` → enable automatically on boot.
    

### 4.2 Set DNS (optional)

```bash
sudo nmcli connection modify HostOnly ipv4.dns "8.8.8.8"
```

We’re using Google’s public DNS here.

### 4.3 Activate it:

```bash
sudo nmcli connection up HostOnly
```

Check the IP:

```bash
ip addr show enp0s8
```

✅ You should see something like:

```
inet 192.168.56.101/24 brd 192.168.56.255 scope global HostOnly
```

---

**Note:** *These steps might depend on your system, run it to avoid any issue at first place.*
## 🔥 Step 5: Allow communication through the firewall 

By default, Rocky Linux may block pings or SSH from the host.  
To fix that, we’ll mark this interface as **trusted**:

```bash
sudo firewall-cmd --permanent --zone=trusted --change-interface=enp0s8
sudo firewall-cmd --reload
```

Now traffic from your host can reach the VM safely.

---

## 🧪 Step 6: Test the connection

### From your Rocky Linux VM:

```bash
ping -c 4 192.168.56.1
```

You should get replies — this means your VM can see the host.

### From your Host (your real computer):

Open **Command Prompt / Terminal** and run:

```bash
ping 192.168.56.101
```

If you get replies, the network is working! 🎉

If not, double-check:

- The firewall rules (`sudo systemctl stop firewalld` temporarily for testing).
    
- The adapter is UP (`ip link set enp0s8 up`).
    
- The IP address (`ip a`).
    

---

## 🧩 Step 7: (Optional) Enable SSH Access

To connect easily from your host to the VM terminal:

Inside Rocky Linux:

```bash
sudo dnf install -y openssh-server
sudo systemctl enable --now sshd
```

Then from your host:

```bash
ssh your_username@192.168.56.101
```

Now you can log in from your host directly into the VM!

---

## ✅ Final check summary

|Item|Setting|
|---|---|
|Host IP|192.168.56.1|
|VM IP|192.168.56.101|
|Network type|Host-Only Adapter (`vboxnet0`)|
|Interface name (VM)|enp0s8|
|Firewall|Trusted for enp0s8|
|Connectivity|Ping host ↔ VM works|

---

Now we’ll make sure your **static IP stays permanent** even after **reboots or network restarts** in your **Rocky Linux 10** VM.

By default, when you use `nmcli`, NetworkManager _should_ remember your configuration — but sometimes (especially after network restarts or OS updates), it might reset or fail to auto-connect.

So we’ll confirm and lock it down **manually**.

---

## 🧭 Step 1: Find your connection configuration file

All NetworkManager connections are saved under:

```bash
/etc/NetworkManager/system-connections/
```

List them:

```bash
sudo ls -l /etc/NetworkManager/system-connections/
```

You’ll see files like:

```
HostOnly.nmconnection
Wired connection 1.nmconnection
System enp0s3.nmconnection
```

Find the one that corresponds to your host-only adapter — in our case, it’s likely:

```
HostOnly.nmconnection
```

---

## 🧾 Step 2: Open the file in an editor

We’ll use `nano` (simple and beginner-friendly):

```bash
sudo nano /etc/NetworkManager/system-connections/HostOnly.nmconnection
```

You’ll see something like this:

```
[connection]
id=HostOnly
uuid=xxxxxx-xxxx-xxxx-xxxx
type=ethernet
autoconnect=true
interface-name=enp0s8

[ipv4]
address1=192.168.56.101/24
method=manual
dns=8.8.8.8;

[ipv6]
method=ignore
```

---

## ✏️ Step 3: Verify (or edit) the important fields

Make sure the following lines exist and look exactly like this 👇

```
[connection]
id=HostOnly
type=ethernet
autoconnect=true
interface-name=enp0s8

[ipv4]
method=manual
address1=192.168.56.101/24
dns=8.8.8.8;
may-fail=false

[ipv6]
method=ignore
```

> 🔹 **`address1`** → your static IP and subnet mask  
> 🔹 **`interface-name`** → your adapter (like `enp0s8`)  
> 🔹 **`autoconnect=true`** → ensures it starts automatically  
> 🔹 **`may-fail=false`** → avoids boot delays even if network not ready  
> 🔹 **`ipv6 method=ignore`** → disables IPv6 (optional but cleaner for private networks)

---

## 🔒 Step 4: Set the correct permissions

NetworkManager requires the file to be owned by root and permissions set to **600** (only root can read/write).

Run:

```bash
sudo chmod 600 /etc/NetworkManager/system-connections/HostOnly.nmconnection
sudo chown root:root /etc/NetworkManager/system-connections/HostOnly.nmconnection
```

---

## 🔁 Step 5: Restart NetworkManager to apply

```bash
sudo systemctl restart NetworkManager
```

Wait a few seconds, then check:

```bash
nmcli connection show
nmcli -f NAME,DEVICE,STATE,IP4 connection show --active
```

✅ You should see your `HostOnly` connection as **activated** with IP `192.168.56.101`.

---

## 🔍 Step 6: Confirm after reboot

Reboot your VM:

```bash
sudo reboot
```

After it starts, log in and run:

```bash
ip addr show enp0s8
```

If you still see:

```
inet 192.168.56.101/24
```

🎉 Congratulations! Your static IP is now **permanent**.

---

## ✅ Quick summary

|Step|Command/Action|Purpose|
|---|---|---|
|1|`sudo ls /etc/NetworkManager/system-connections/`|Locate the config file|
|2|`sudo nano /etc/NetworkManager/system-connections/HostOnly.nmconnection`|Edit the settings|
|3|Set `address1=192.168.56.101/24`, `method=manual`, etc.|Define static IP|
|4|`sudo chmod 600 file`|Secure the config|
|5|`sudo systemctl restart NetworkManager`|Apply changes|
|6|`sudo reboot` then `ip a`|Verify persistence|

---

*You can run multiple VMS following these steps, just set a new IP.* 