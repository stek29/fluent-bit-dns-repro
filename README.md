# Fluent-bit IPv6 DNS Resolution Test Lab

This is a minimal lab setup to reproduce and test IPv6 DNS resolution issues in fluent-bit when DNS responses contain many IPv6 addresses and exceed the 512-byte UDP limit.

## TL;DR: Reproduce the bug

1.  **Start the environment:**
    ```bash
    docker compose up -d --build
    ```

2.  **Check the logs of the default fluent-bit instance (uses UDP for DNS):**
    ```bash
    docker compose logs fluent-bit-default
    ```
    - ✅ You will see **successful** logs for `3-records.test.example.com`.
    - ❌ You will see **DNS resolution errors** for `20-records.test.example.com`, because the DNS response is too large for UDP and fluent-bit doesn't fall back to TCP.

3.  **Check the logs of the TCP-only fluent-bit instance:**
    ```bash
    docker compose logs fluent-bit-dns-tcp
    ```
    - ✅ You will see **successful** logs for both hostnames, as this instance is configured to use DNS over TCP.

4.  **Clean up:**
    ```bash
    docker compose down
    ```

## Problem Context

The original issue was encountered with fluent-bit's `kinesis_streams` plugin when DNS responses for endpoints contained too many IPv6 addresses, causing the response to exceed the 512-byte UDP DNS limit. This lab provides a controlled environment to reproduce and test this issue.

## Components

- **DNS Server**: dnsmasq with custom hosts file containing multiple IPv6 addresses
- **Test Server**: nginx server responding to HTTP requests  
- **Fluent-bit**: Client generating test logs and sending to test server via HTTP output

## Current Status (Reproduced Bug)

✅ **Lab is fully functional and reproduces the bug**:
- **3-records.test.example.com** (3 IPv6 addresses): ✅ Fluent-bit works correctly (DNS response: ~139 bytes, HTTP 200 responses)
- **20-records.test.example.com** (20 IPv6 addresses): ❌ DNS response is ~616 bytes (over 512-byte UDP limit), fluent-bit fails to resolve hostname
- **Error**: `getaddrinfo(host='20-records.test.example.com', err=11): Could not contact DNS servers`
- **DNS queries**: Both UDP and TCP work from host, but fluent-bit does not fall back to TCP
- **Tested versions**: ✅ Confirmed on Fluent Bit v4.0.4 (latest stable release)

## Network Configuration

- **DNS Server**: `2001:db8::2` (port 5453 on host)
- **Test Server**: `2001:db8::10` (port 8080 on host)
- **Fluent-bit**: `2001:db8::100`
- **Network**: `2001:db8::/64` with IPv6 enabled

## Quick Start

1. **Start the lab**:
   ```bash
   docker compose up -d
   ```

2. **Test DNS resolution from host**:
   ```bash
   # Test 3-records server (3 IPv6 addresses)
   dig @localhost -p 5453 3-records.test.example.com 
   
   # Test 20-records server (20 IPv6 addresses)  
   dig @localhost -p 5453 20-records.test.example.com AAAA
   
   # TCP query for 20-records server
   dig @localhost -p 5453 +tcp 20-records.test.example.com AAAA
   ```

3. **Check fluent-bit logs**:
   ```bash
   # Check default fluent-bit instance (UDP DNS)
   docker compose logs fluent-bit-default

   # Check fluent-bit instance with DNS over TCP
   docker compose logs fluent-bit-dns-tcp
   ```

4. **Check server logs**:
   ```bash
   docker compose logs server
   ```

## Current Test Results

### DNS Response Analysis
- **3-records.test.example.com** (3 IPv6 addresses): Response size ~139 bytes, fluent-bit works ✅
- **20-records.test.example.com** (20 IPv6 addresses): Response size ~616 bytes, fluent-bit fails ❌
- **Host DNS**: Both UDP and TCP queries work from host
- **Fluent-bit**: Fails to resolve when response >512 bytes, does not fall back to TCP

### Latest Testing (July 2025)
- **Fluent Bit v4.0.4**: ✅ Issue confirmed and reproducible
- **3-records.test.example.com**: ✅ Works correctly (DNS response: ~139 bytes, HTTP 200 responses)
- **20-records.test.example.com**: ❌ Fails (DNS response: ~616 bytes, exceeds 512-byte UDP limit)
- **Standard DNS Behavior**: `dig` automatically falls back to TCP when UDP response is truncated
- **Fluent Bit Behavior**: Fails with errors `DNS server returned answer with no data` and `Could not contact DNS servers`
- **TCP Fallback**: Fluent Bit does not implement TCP fallback for truncated DNS responses

### Fluent-bit Behavior
- **Working**: `3-records.test.example.com` with 3 IPv6 addresses, HTTP 200 responses received
- **Failing**: `20-records.test.example.com` with 20 IPv6 addresses, DNS resolution fails in the default UDP mode, but works when DNS mode is set to TCP.
  - `fluent-bit-default` (UDP mode): Fails with `[ warn] [net] getaddrinfo(host='20-records.test.example.com', err=11): Could not contact DNS servers`
  - `fluent-bit-dns-tcp` (TCP mode): Works as expected.

## Configuration Files

### DNS Configuration (`dns/hosts`)
```
# Test hosts file with multiple IPv6 addresses
# Using RFC 3849 documentation prefix (2001:db8::/32)
# This simulates a scenario where DNS response might exceed 512 bytes

# 3-records server with 3 IPv6 addresses (DNS response < 512 bytes, fluent-bit works)
2001:db8::10 3-records.test.example.com
2001:db8::1 3-records.test.example.com
2001:db8::2 3-records.test.example.com

# 20-records server with 20 IPv6 addresses (DNS response > 512 bytes, fluent-bit fails)
2001:db8::10 20-records.test.example.com
2001:db8::1 20-records.test.example.com
2001:db8::2 20-records.test.example.com
2001:db8::3 20-records.test.example.com
2001:db8::4 20-records.test.example.com
2001:db8::5 20-records.test.example.com
2001:db8::6 20-records.test.example.com
2001:db8::7 20-records.test.example.com
2001:db8::8 20-records.test.example.com
2001:db8::9 20-records.test.example.com
2001:db8::a 20-records.test.example.com
2001:db8::b 20-records.test.example.com
2001:db8::c 20-records.test.example.com
2001:db8::d 20-records.test.example.com
2001:db8::e 20-records.test.example.com
2001:db8::f 20-records.test.example.com
2001:db8::11 20-records.test.example.com
2001:db8::12 20-records.test.example.com
2001:db8::13 20-records.test.example.com
2001:db8::14 20-records.test.example.com
```

### Fluent-bit Configuration
The configuration is now split into multiple files for clarity and to test different DNS resolution strategies.

- **`fluent-bit/common.conf`**:
  - **Input**: Dummy plugin generating test logs every 3 seconds.
  - **Output 1**: HTTP plugin pointing to `3-records.test.example.com` (3 IPv6 addresses, should always work).
  - **Output 2**: HTTP plugin pointing to `20-records.test.example.com` (20 IPv6 addresses, fails in UDP mode but works in TCP mode).
- **`fluent-bit/fluent-bit-default.conf`**:
  - Main configuration for the `fluent-bit-default` service.
  - Includes `common.conf`.
  - Uses the default DNS resolution (UDP with no fallback).
- **`fluent-bit/fluent-bit-dns-tcp.conf`**:
  - Main configuration for the `fluent-bit-dns-tcp` service.
  - Includes `common.conf`.
  - Sets `dns.mode TCP` to force DNS queries over TCP.

### Server Configuration (`server/nginx.conf`)
- **IPv6 listening**: `listen [::]:80` (explicit IPv6 support)
- **IPv4 listening**: `listen 80` (backward compatibility)

## Testing Different Scenarios

### Test with current setup (always reproducible):
```bash
docker compose up -d
docker compose logs fluent-bit-default
docker compose logs fluent-bit-dns-tcp
```

### Test DNS resolution manually:
```bash
# Check DNS response size for 3-records server
dig @localhost -p 5453 3-records.test.example.com AAAA | grep "MSG SIZE"

# Check DNS response size for 20-records server
dig @localhost -p 5453 20-records.test.example.com AAAA | grep "MSG SIZE"
```

## Key Findings

- **DNS server**: dnsmasq works correctly with IPv6 addresses
- **Hosts file format**: Must be `IP hostname` (not `hostname IP`)
- **Port conflicts**: Using port 5453 to avoid system DNS conflicts
- **IPv6 networking**: Docker IPv6 networking properly configured
- **DNS resolution**: ✅ Working with 3 IPv6 addresses (~139 bytes response)
- **Bug reproduced**: ❌ Fluent-bit fails DNS resolution with 20 IPv6 addresses (~616 bytes)
- **No TCP fallback**: Fluent-bit does not retry with TCP when UDP response is too large
- **Nginx IPv6**: Must explicitly configure `listen [::]:80`
- **Fluent-bit DNS**: Fails when DNS response exceeds 512 bytes
- **Version tested**: ✅ Confirmed on Fluent Bit v4.0.4 (latest stable release)
- **Always reproducible**: ✅ Two hostnames configured - 3-records (works) and 20-records (fails)

## Cleanup

```bash
docker compose down
```

## Files Structure

```
fluent-bit-dns-repro/
├── docker-compose.yml          # Main orchestration
├── dns/
│   ├── dnsmasq.conf           # DNS server configuration
│   └── hosts                  # Two hostnames: 3-records and 20-records
├── server/
│   └── nginx.conf             # Simple test server with IPv6 support
├── fluent-bit/
│   ├── common.conf              # Common input/output for both fluent-bit instances
│   ├── fluent-bit-default.conf  # Config for default (UDP) DNS resolution
│   └── fluent-bit-dns-tcp.conf  # Config for DNS over TCP resolution
└── README.md                  # This file
``` 
