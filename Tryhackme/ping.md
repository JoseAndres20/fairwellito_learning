## Basic Usage

On Linux and macOS, use the `-c` flag to specify the number of packets to send.

```bash
ping -c 5 MACHINE_IP
```

You can also ping a hostname, in which case DNS resolution happens first.

```bash
ping -c 5 tryhackme.com
```

On Windows, the equivalent flag is `-n`.

```cmd
ping -n 5 MACHINE_IP
```

If you omit the count on Linux, ping runs indefinitely. Press **Ctrl+C** to stop it.

You can force a specific IP version using the `-4` and `-6` flags. This is useful in dual-stack environments where a hostname resolves to both IPv4 and IPv6 addresses. On some systems, `ping6` is available as a standalone command for IPv6.

```bash
ping -4 -c 5 MACHINE_IP
ping -6 -c 5 MACHINE_IPV6
```

## Interpreting the Output: Successful Ping

The following example shows a target that is alive and allows ICMP.

```bash
user@AttackBox$ ping -c 5 MACHINE_IP
PING MACHINE_IP (MACHINE_IP) 56(84) bytes of data.
64 bytes from MACHINE_IP: icmp_seq=1 ttl=64 time=0.512 ms
64 bytes from MACHINE_IP: icmp_seq=2 ttl=64 time=0.478 ms
64 bytes from MACHINE_IP: icmp_seq=3 ttl=64 time=0.491 ms
64 bytes from MACHINE_IP: icmp_seq=4 ttl=64 time=0.503 ms
64 bytes from MACHINE_IP: icmp_seq=5 ttl=64 time=0.485 ms

--- MACHINE_IP ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4098ms
rtt min/avg/max/mdev = 0.478/0.494/0.512/0.012 ms
```