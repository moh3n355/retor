# retor

A tiny background CLI tool that rotates your Tor exit IP on an interval, so tools like `ffuf`, `gobuster`, `wfuzz`, etc. can be routed through Tor and dodge per-IP rate limits during fuzzing/recon — without blocking your terminal.

```bash
$ retor 5
retor started in background (PID 12345), rotating every 5s.

$ curl --socks5 127.0.0.1:9050 https://check.torproject.org/api/ip
{"IsTor":true,"IP":"185.220.101.15"}

$ retor -d
retor stopped.
```

## Why

Most WAFs / rate limiters throttle or block based on source IP. If you're fuzzing directories, parameters, or subdomains and keep getting `429`s or soft-blocked, routing traffic through Tor and rotating the exit node periodically spreads your requests across many different IPs.

`retor` just automates the "rotate the exit node every N seconds" part in the background, so you can kick it off once and keep fuzzing in the same terminal.

## Requirements

- `tor` installed and running
- `pkill` (from `procps`, included by default on most Linux distros)
- `proxychains` (or any SOCKS-aware fuzzing tool) if you want to route other tools through Tor

## Installation

### 1. Install Tor
```bash
sudo apt update
sudo apt install tor proxychains4 -y
sudo systemctl enable --now tor
```

By default Tor listens as a SOCKS5 proxy on `127.0.0.1:9050`.

### 2. Install retor
```bash
git clone https://github.com/<your-username>/retor.git
cd retor
chmod +x retor
sudo mv retor /usr/local/bin/retor
```

### 3. Verify Tor is working
```bash
curl --socks5 127.0.0.1:9050 https://check.torproject.org/api/ip
```
You should get back `{"IsTor":true,"IP":"..."}`.

## Usage

```bash
retor <seconds>   # start rotating the exit node every N seconds, in the background
retor -d          # stop rotating
```

- Runs detached (`setsid`) — your shell is free immediately, no `&` or extra terminal needed.
- PID is tracked in `/tmp/.retor.pid`, so `retor -d` always finds and kills the right process.
- Logs go to `/tmp/.retor.log` if you want to check activity.

## Using it with ffuf

`ffuf` doesn't support SOCKS proxies natively via a simple flag in older versions in all build configs, so the most reliable way is to route it through `proxychains`:

### 1. Configure proxychains
Edit `/etc/proxychains4.conf` (or `/etc/proxychains.conf`), make sure the bottom has:
```
socks5 127.0.0.1 9050
```
Also set the chain type near the top to:
```
dynamic_chain
```

### 2. Start retor
```bash
retor 5
```
This rotates the Tor circuit every 5 seconds in the background.

### 3. Run ffuf through proxychains
```bash
proxychains4 -q ffuf -u https://target.com/FUZZ -w wordlist.txt -t 10
```

**Relevant ffuf switches for this workflow:**

| Flag | Purpose |
|---|---|
| `-t 10` | Lower thread count. Tor is slow and each new circuit adds latency — too many threads will just cause timeouts, not more speed. |
| `-p 1.0-3.0` | Adds a random delay (in seconds) between requests. Pairs well with rotation so requests aren't machine-gunned through the same circuit. |
| `-rate 5` | Caps requests/sec globally — useful to stay under the rate limit even while rotating. |
| `-mc all` / `-mc 200,301,403` | Match codes — tune so you're not drowning in noise from Tor-related errors. |
| `-timeout 15` | Increase from default (10s) since Tor circuits can be slow, especially right after a rotation. |
| `-o results.json -of json` | Save output, since long fuzzing runs through Tor can take a while and you don't want to lose progress. |

Example full command:
```bash
retor 10
proxychains4 -q ffuf -u https://target.com/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 8 -p 0.5-2.0 -timeout 15 \
  -mc 200,204,301,302,307,401,403 \
  -o results.json -of json
retor -d
```

### Tuning the rotation interval
- Match `retor`'s interval to roughly how many requests fit before you'd normally get rate-limited. If the limit is "~20 requests per IP per 10s", set `retor 8` so you rotate just before hitting it.
- Too aggressive (e.g. `retor 1`) can actually slow you down — new Tor circuits take a moment to establish and some requests will fail or hang while a fresh circuit builds.

## Note on how rotation actually works

`retor` sends a `HUP` signal to the local Tor process, which reloads Tor's configuration. In practice this often shuffles which circuit gets used for new connections, but it isn't a guaranteed "new circuit every time" — Tor may still reuse an existing circuit briefly since it keeps several alive in parallel.

For **deterministic** rotation (force a brand-new circuit/exit on every call), Tor's `NEWNYM` signal sent over the ControlPort is the correct primitive:

```bash
echo -e 'AUTHENTICATE "your_password"\r\nSIGNAL NEWNYM\r\nQUIT' | nc 127.0.0.1 9051
```

This requires enabling `ControlPort 9051` and setting a `HashedControlPassword` (or cookie auth) in `/etc/tor/torrc`. A `NEWNYM`-based version of retor may be added later for stricter rate-limit-evasion needs.

## Disclaimer

This tool is intended for **authorized** security testing — bug bounty programs, pentest engagements, or CTFs/labs where you have explicit permission to test. Bypassing rate limits on infrastructure you don't have permission to test is against most programs' rules and may be illegal in your jurisdiction. Always check the scope and rules of engagement before fuzzing through Tor or any anonymization layer. The author is not responsible for misuse.

## License

MIT
