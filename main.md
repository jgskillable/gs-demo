# Lab: Setting Up a Custom DNS Server in Windows


 +++Demo+++ 

## Overview

In this lab, you will configure a custom DNS server on a Windows machine using the **DNS Server** role available in Windows Server. By the end of this lab, you will be able to:

- Install the DNS Server role on Windows Server
- Create forward and reverse lookup zones
- Add DNS records (A, CNAME, PTR)
- Test DNS resolution using `nslookup` and `Resolve-DnsName`

---

## Prerequisites

- A machine running **Windows Server 2019** or **Windows Server 2022** (physical, virtual, or cloud-based)
- Local Administrator privileges
- Basic familiarity with Windows Server administration
- (Optional) A second machine or VM to use as a DNS client for testing

---

## Lab Environment

| Role       | Hostname       | IP Address     |
|------------|----------------|----------------|
| DNS Server | `WIN-DNS01`    | `192.168.1.10` |
| DNS Client | `WIN-CLIENT01` | `192.168.1.20` |

> **Note:** Adjust the hostnames and IP addresses to match your own lab environment.

---

## Part 1 – Install the DNS Server Role

### Step 1.1 – Open Server Manager

1. Log in to the Windows Server machine that will act as the DNS server.
2. Open **Server Manager** (it typically opens automatically at login, or search for it in the Start menu).

### Step 1.2 – Add the DNS Server Role

1. In Server Manager, click **Manage** → **Add Roles and Features**.
2. On the *Before You Begin* page, click **Next**.
3. Select **Role-based or feature-based installation**, then click **Next**.
4. Select the local server from the server pool, then click **Next**.
5. In the *Server Roles* list, check **DNS Server**.
6. When prompted to add required features, click **Add Features**.
7. Click **Next** through the remaining pages, then click **Install**.
8. Wait for the installation to complete, then click **Close**.

### Step 1.3 – Verify the Installation

Open **PowerShell** as Administrator and run:

```powershell
Get-WindowsFeature -Name DNS
```

You should see output similar to:

```
Display Name                           Name                    Install State
------------                           ----                    -------------
[X] DNS Server                         DNS                     Installed
```

---

## Part 2 – Configure a Forward Lookup Zone

A **forward lookup zone** maps hostnames to IP addresses.

### Step 2.1 – Open the DNS Manager

1. In Server Manager, click **Tools** → **DNS**.
2. Expand the server node in the DNS Manager tree.

### Step 2.2 – Create a New Forward Lookup Zone

1. Right-click **Forward Lookup Zones** and select **New Zone…**
2. Click **Next** on the wizard welcome page.
3. Select **Primary zone**, then click **Next**.
4. Enter a zone name (e.g., `lab.local`), then click **Next**.
5. Accept the default zone file name (`lab.local.dns`), then click **Next**.
6. Select **Do not allow dynamic updates**, then click **Next**.
7. Click **Finish**.

### Step 2.3 – Add an A Record

1. In DNS Manager, expand **Forward Lookup Zones** and click the `lab.local` zone.
2. Right-click in the right pane and select **New Host (A or AAAA)…**
3. Enter the following:
   - **Name:** `webserver`
   - **IP address:** `192.168.1.50`
4. Click **Add Host**, then **OK**, then **Done**.

### Step 2.4 – Add a CNAME Record

1. Right-click the `lab.local` zone and select **New Alias (CNAME)…**
2. Enter the following:
   - **Alias name:** `www`
   - **Fully qualified domain name (FQDN) for target host:** `webserver.lab.local.`
3. Click **OK**.

---

## Part 3 – Configure a Reverse Lookup Zone

A **reverse lookup zone** maps IP addresses back to hostnames (PTR records).

### Step 3.1 – Create the Reverse Lookup Zone

1. In DNS Manager, right-click **Reverse Lookup Zones** and select **New Zone…**
2. Click **Next** on the welcome page.
3. Select **Primary zone**, then click **Next**.
4. Select **IPv4 Reverse Lookup Zone**, then click **Next**.
5. Enter the network ID: `192.168.1`, then click **Next**.
6. Accept the default zone file name, then click **Next**.
7. Select **Do not allow dynamic updates**, then click **Next**.
8. Click **Finish**.

### Step 3.2 – Add a PTR Record

1. In DNS Manager, expand **Reverse Lookup Zones** and click the `1.168.192.in-addr.arpa` zone.
2. Right-click in the right pane and select **New Pointer (PTR)…**
3. Enter the following:
   - **Host IP address:** `50` (last octet of `192.168.1.50`)
   - **Host name:** `webserver.lab.local.`
4. Click **OK**.

---

## Part 4 – Configure the DNS Client

On the **DNS Client** machine (`WIN-CLIENT01`):

### Step 4.1 – Set the DNS Server Address

1. Open **Network and Sharing Center** → **Change adapter settings**.
2. Right-click the active network adapter and select **Properties**.
3. Select **Internet Protocol Version 4 (TCP/IPv4)** and click **Properties**.
4. Select **Use the following DNS server addresses** and enter:
   - **Preferred DNS server:** `192.168.1.10`
5. Click **OK**, then **Close**.

Alternatively, use PowerShell:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "192.168.1.10"
```

---

## Part 5 – Test DNS Resolution

### Step 5.1 – Test with `nslookup`

On the DNS client, open a Command Prompt and run:

```cmd
nslookup webserver.lab.local
nslookup www.lab.local
nslookup 192.168.1.50
```

Expected output for the first command:

```
Server:  WIN-DNS01.lab.local
Address:  192.168.1.10

Name:    webserver.lab.local
Address:  192.168.1.50
```

### Step 5.2 – Test with `Resolve-DnsName`

Open PowerShell and run:

```powershell
Resolve-DnsName -Name "webserver.lab.local" -Server "192.168.1.10"
Resolve-DnsName -Name "www.lab.local" -Server "192.168.1.10"
Resolve-DnsName -Name "192.168.1.50" -Server "192.168.1.10"
```

### Step 5.3 – Flush the DNS Cache

If you encounter stale results, flush the DNS resolver cache on the client:

```cmd
ipconfig /flushdns
```

---

## Part 6 – Managing DNS with PowerShell

You can also manage the DNS server entirely from PowerShell using the **DnsServer** module.

### View all DNS zones

```powershell
Get-DnsServerZone
```

### Add a new A record

```powershell
Add-DnsServerResourceRecordA -ZoneName "lab.local" -Name "fileserver" -IPv4Address "192.168.1.60"
```

### Add a new CNAME record

```powershell
Add-DnsServerResourceRecordCName -ZoneName "lab.local" -Name "files" -HostNameAlias "fileserver.lab.local"
```

### Remove a DNS record

```powershell
Remove-DnsServerResourceRecord -ZoneName "lab.local" -RRType "A" -Name "fileserver" -Force
```

### View all records in a zone

```powershell
Get-DnsServerResourceRecord -ZoneName "lab.local"
```

---

## Troubleshooting

| Issue | Possible Cause | Resolution |
|---|---|---|
| `nslookup` times out | Client cannot reach DNS server | Check firewall rules; ensure UDP/TCP port 53 is open |
| Record not found | Record not created or typo in name | Verify the record exists in DNS Manager |
| Wrong IP returned | Stale cache | Run `ipconfig /flushdns` on the client |
| Reverse lookup fails | PTR record missing | Add PTR record in the reverse lookup zone |
| DNS service not starting | Configuration error | Check Event Viewer → Windows Logs → System |

---

## Summary

In this lab, you:

1. Installed the **DNS Server** role on Windows Server.
2. Created a **forward lookup zone** (`lab.local`) and added **A** and **CNAME** records.
3. Created a **reverse lookup zone** and added a **PTR** record.
4. Configured a DNS client to use the custom DNS server.
5. Verified DNS resolution using `nslookup` and `Resolve-DnsName`.
6. Managed DNS records using PowerShell cmdlets.

You now have a fully functional custom DNS server running in your Windows lab environment.
