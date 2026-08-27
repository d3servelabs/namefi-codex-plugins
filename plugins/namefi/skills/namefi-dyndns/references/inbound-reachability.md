# Inbound reachability: proving it, and what silently blocks it

Shared reference for all four `namefi-dyndns-*` skills. Publishing an A record
is the easy half. This file is the hard half: whether a packet from the internet
can actually arrive, and how to prove it rather than assume it.

Every claim here was established empirically on 2026-08-21 against two Egyptian
consumer connections; the commands are the ones that produced the verdicts.

---

## 1. Carrier NAT: `100.64/10` is NOT the only form

`namefi dyndns check` flags carrier NAT only when the gateway's external address
falls in RFC 6598 (`100.64.0.0/10`). **Real ISPs also carrier-NAT using RFC1918
`10.x`**, which that test misses completely. Two connections examined the same
day gave opposite verdicts and neither was detected correctly by the range check:

| | Vodafone EG mobile | Fixed line |
|---|---|---|
| Public IPv4 (`ifconfig.co`) | 196.156.202.10 | 197.58.250.173 |
| traceroute hops 2-6 | `10.7.x`, `10.5.x` | `10.45.x`, `10.39.x` |
| **Router WAN IP (UPnP)** | private | **197.58.250.173** |
| Verdict | **carrier NAT — inbound impossible** | **single NAT — forwarding works** |

**Private traceroute hops do not prove carrier NAT.** ISPs routinely number their
core links with RFC1918 while still routing a real public address to your CPE.
Both connections above looked identical in traceroute and were not.

### The decisive test: ask the router its WAN address

UPnP `GetExternalIPAddress` is unauthenticated and takes seconds. If the router's
WAN IP **equals** the address the internet sees, it is single-layer NAT and port
forwarding will work. If it is private, no router setting can ever help.

```bash
python3 - <<'EOF'
import socket, re, urllib.request, urllib.parse
msg = ('M-SEARCH * HTTP/1.1\r\nHOST:239.255.255.250:1900\r\n'
       'ST:urn:schemas-upnp-org:device:InternetGatewayDevice:1\r\n'
       'MX:2\r\nMAN:"ssdp:discover"\r\n\r\n').encode()
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM); s.settimeout(4)
s.sendto(msg, ('239.255.255.250', 1900))
locs = set()
try:
    while True:
        d, _ = s.recvfrom(65507)
        m = re.search(rb'LOCATION:\s*(\S+)', d, re.I)
        if m: locs.add(m.group(1).decode())
except socket.timeout: pass
for loc in locs:
    xml = urllib.request.urlopen(loc, timeout=5).read().decode(errors='replace')
    for svc in re.findall(r'<service>(.*?)</service>', xml, re.S):
        st = re.search(r'<serviceType>(.*?)</serviceType>', svc)
        cu = re.search(r'<controlURL>(.*?)</controlURL>', svc)
        if not st or not cu: continue
        if 'WANIPConnection' not in st.group(1) and 'WANPPPConnection' not in st.group(1): continue
        body = ('<?xml version="1.0"?><s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" '
                's:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"><s:Body>'
                f'<u:GetExternalIPAddress xmlns:u="{st.group(1)}"/></s:Body></s:Envelope>')
        req = urllib.request.Request(urllib.parse.urljoin(loc, cu.group(1)), data=body.encode(),
              headers={'Content-Type':'text/xml; charset="utf-8"',
                       'SOAPAction': f'"{st.group(1)}#GetExternalIPAddress"'})
        r = urllib.request.urlopen(req, timeout=6).read().decode(errors='replace')
        ip = re.search(r'<NewExternalIPAddress>(.*?)</NewExternalIPAddress>', r)
        print("ROUTER WAN IP =", ip.group(1) if ip else r[:200])
EOF
curl -4 -fsS https://ifconfig.co; echo   # what the internet sees
```

If the router speaks no UPnP at all, fall back to its admin UI status page, and
treat "private WAN address" as the only thing you are looking for.

---

## 2. Routers refuse privileged ports over UPnP — retry high before concluding

A router that answers `AddPortMapping` with **`HTTP 500 / UPnP error 501 (Action
Failed)`** for external port **80 or 443** will very often grant `18080`, `8443`,
or `32400` on the first ask. Ports 80/443 are commonly reserved for the router's
own admin UI or blocked by ISP policy.

**A 501 on port 80 does not mean "UPnP is broken."** Always retry a high port
before reporting that mapping is unavailable:

```bash
for P in 18080 8443 32400; do
  namefi dyndns check --hostname <host> --map ${P}:8080/tcp 2>&1 | grep -i 'port mapping'
done
```

Distinguish the two failure shapes — they mean different things:

| Router response | Meaning |
|---|---|
| `connection refused` on SSDP/1900 | no UPnP IGD service at all — enable it, or forward manually |
| `HTTP 500 / UPnP error 501 (Action Failed)` | UPnP works; **this particular port** was refused — retry high |
| mapping granted on a *different* external port | the CLI treats this as failure (request-or-nothing) |

**Port-convention caveat:** `8443` conventionally implies TLS. Serving plain HTTP
on it will confuse scanners, corporate proxies, and any browser handed
`https://…:8443`. `18080` / `32400` carry no such expectation — prefer them when
the service is not actually TLS.

---

## 3. The host firewall blocks inbound while loopback still works

The signature is unmistakable and easy to misread as a NAT problem:

- `curl http://127.0.0.1:PORT/` → **200**
- `curl http://<LAN-IP>:PORT/` → **times out**

The listener is bound (`lsof -nP -iTCP:PORT -sTCP:LISTEN` shows `*:PORT`), but
the OS firewall drops inbound. Forwarded NAT traffic hits the same wall, so fix
this **before** blaming the router.

### macOS — the Application Firewall, and the Apple-`container` trap

```bash
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
/usr/libexec/ApplicationFirewall/socketfilterfw --listapps | grep -i <binary>
```

macOS allowlists **per binary**, so allowlist the *actual listening process*, not
the CLI you typed. For Apple's `container` runtime, `-p 8080:80` is served by a
helper, and that helper is what needs the entry:

```
/usr/local/libexec/container/plugins/container-runtime-linux/bin/container-runtime-linux
```

Find it generically with `lsof -p "$(lsof -ti tcp:PORT | head -1)" -a -d txt`.

```bash
B=/usr/local/libexec/container/plugins/container-runtime-linux/bin/container-runtime-linux
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add "$B"
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp "$B"
```

**Agent contexts have no TTY, so `sudo` cannot prompt** — including under Claude
Code's `!` prefix. Use the GUI authorization dialog instead:

```bash
osascript -e "do shell script \"…\" with administrator privileges"
```

Prefer a narrow per-binary allowlist over `--setglobalstate off`; it is
reversible with `--remove` and leaves the firewall on.

### Linux

```bash
sudo ufw status                      # or:
sudo firewall-cmd --list-all
sudo nft list ruleset | head -40
```

---

## 4. Hairpin NAT: you cannot test your own public IP from inside

Once the record is live, the hostname resolves to the router's **WAN** address
for everyone, including hosts on the LAN. Many consumer routers do not loop that
traffic back inside, so **an inside test proves nothing in either direction** —
it can fail while the world is fine, or succeed for reasons the world never sees.

The timing is the tell:

| Symptom from inside the LAN | Reading |
|---|---|
| fails in **milliseconds** (`Couldn't connect`) | router actively rejects LAN→WAN-IP — classic no-hairpin. Says nothing about outside |
| **times out** (seconds) | more likely a genuinely closed port or host firewall |

### Report both URLs — this is the default, not an extra

For any **NAT topology** (not a VPS), never hand the user a single URL. The
external one does not work from the machine they are sitting at, and saying only
that will read as "it's broken." Always print both:

```
local:    http://127.0.0.1:8080/hello.html
external: http://home.example.com:32400/hello.html
```

Then the note, every time:

> To reach the **external** URL from inside this LAN, add the hostname to
> `/etc/hosts` (`192.168.1.11  home.example.com`) — the router does not hairpin,
> so the public address is not reachable from behind it. That covers this machine
> only; other LAN devices each need the same entry, or a DNS override on the
> router. Note `/etc/hosts` maps a **name to an address, never a port** — if the
> external URL carries `:32400` while the service listens on `:8080`, you must
> also publish it on the matching port, or the hosts entry alone will not help.

On a **VPS** (no NAT) there is nothing to disambiguate — one URL is correct
everywhere, and adding a `local:` line is noise.

To make one URL work on both sides, both halves must match:

1. `/etc/hosts` → `<LAN-IP> host.example.com` (that machine only; other LAN
   devices need their own entry or a router DNS override), **and**
2. publish the container/service on the *same* port that appears in the URL —
   a hosts entry alone does not fix a port mismatch. Apple's `container` cannot
   add a port to a running container; recreate it with both `-p` flags.

---

## 5. Verify from outside — and control-test the verifier

Only an off-network client counts. In order of preference:

1. **A phone on cellular**, or any host genuinely off the LAN.
2. **`check-host.net`** — distributed probes with a JSON API, no auth:

```bash
# TCP reachability
R=$(curl -sS -H 'Accept: application/json' \
  'https://check-host.net/check-tcp?host=<IP>:<PORT>&max_nodes=4')
RID=$(echo "$R" | python3 -c "import json,sys;print(json.load(sys.stdin)['request_id'])")
sleep 20
curl -sS -H 'Accept: application/json' "https://check-host.net/check-result/$RID"

# Full HTTP through the real hostname (URL-encode it)
curl -sS -H 'Accept: application/json' \
  'https://check-host.net/check-http?host=http%3A%2F%2Fhost.example.com%3A32400%2F&max_nodes=6'
```

A node result of `{"error": "Connection timed out"}` is closed; a TCP entry with
a `time` and no `error`, or an HTTP entry whose first field is `1` with status
`200`, is open. Expect the occasional flaky node — **re-run with more nodes
before declaring a partial failure real**; a 5/6 result is almost always the node.

### Always control-test the checker

Generic fetch proxies (`api.allorigins.win`, `api.codetabs.com`) were themselves
unreachable on the test day and returned `522` for *every* URL. Reading that as
"my port is closed" would have been wrong. **Before believing a negative, point
the same tool at a known-good URL** (`http://example.com/`); if the control also
fails, the verifier is broken and the test is void.

Also confirm the record itself independently:

```bash
dig +short host.example.com A
dig +short @1.1.1.1 host.example.com A
```

---

## Order of operations

Working outward, cheapest and most-decisive first:

1. **Router WAN IP via UPnP** — carrier NAT kills every other step; settle it first.
2. **Host firewall** — loopback-works/LAN-fails is the tell.
3. **Port mapping**, retrying a high external port on a `501`.
4. **External verification**, with a control test.
5. Only then reason about hairpin, and never let an inside test stand as proof.
