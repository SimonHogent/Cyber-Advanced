# Lab 3: SSH

## 1. SSH client config

* **Set up an SSH client config to easily reach all machines. Use a bastion if needed. What files are transferred to what machines? Is this the most secure method?**
  * `companyrouter` acts as the bastion for the company LAN: the firewall only allows SSH into it from the fake internet (`eth1 tcp dport 22`), everything behind it (dns, web, database, employee) is reached through it via `ProxyJump`. `homerouter` plays the same bastion role for the employee home LAN, reaching `remote-employee`. `isprouter` and `homerouter` themselves sit directly on the fake-internet subnet as the laptop, so no jump is needed to reach them.
  * A dedicated SSH keypair (`id_ed25519_ca`, Ed25519, generated on the laptop) was created and its **public key** (`id_ed25519_ca.pub`) copied to `~/.ssh/authorized_keys` on every machine (companyrouter, isprouter, homerouter, dns, web, database, employee, remote-employee) — nothing else is transferred; the private key never leaves the laptop.
  * `~/.ssh/config` on the laptop:
    ```
    Host ca-companyrouter
        HostName 192.168.62.253
        User vagrant
        IdentityFile ~/.ssh/id_ed25519_ca

    Host ca-isprouter
        HostName 192.168.62.254
        User vagrant
        IdentityFile ~/.ssh/id_ed25519_ca

    Host ca-homerouter
        HostName 192.168.62.42
        User vagrant
        IdentityFile ~/.ssh/id_ed25519_ca

    Host ca-dns
        HostName 172.30.0.4
        User vagrant
        ProxyJump ca-companyrouter
        IdentityFile ~/.ssh/id_ed25519_ca

    Host ca-web
        HostName 172.30.20.10
        User vagrant
        ProxyJump ca-companyrouter
        IdentityFile ~/.ssh/id_ed25519_ca

    Host ca-database
        HostName 172.30.0.15
        User vagrant
        ProxyJump ca-companyrouter
        IdentityFile ~/.ssh/id_ed25519_ca

    Host ca-employee
        HostName 172.30.0.123
        User vagrant
        ProxyJump ca-companyrouter
        IdentityFile ~/.ssh/id_ed25519_ca

    Host ca-remote-employee
        HostName 172.10.10.123
        User vagrant
        ProxyJump ca-homerouter
        IdentityFile ~/.ssh/id_ed25519_ca
    ```
  * Result: `ssh ca-<name>` logs in directly, passwordless, on **every** machine in the environment — automatically hopping through the correct bastion where needed, no manual `-J` or IP addresses required.
  * **Is this the most secure method?** Not yet fully: all machines still accept the `vagrant` account with its known default password as a fallback, and the keys themselves have no passphrase. A more secure setup disables password authentication entirely (`PasswordAuthentication no`) and uses a dedicated, non-default user per machine with a passphrase-protected key — planned as a follow-up hardening step (see below).

## 2. SSH port forwarding

* **Open a port on the router that connects back to the database, so the fake internet can reach it although the firewall blocks a direct connection.**
  ```bash
  ssh -L 3307:172.30.0.15:3306 ca-companyrouter
  ```
  Confirmed working from the laptop by reading the raw MariaDB handshake banner on the forwarded port:
  ```
  5.5.5-10.11.11-MariaDB ... mysql_native_password
  ```
  This proves the tunnel reaches the database service, even though the firewall drops a direct connection to `172.30.0.15:3306`.

* **Why is this interesting from a security point of view?**
  * It bypasses the firewall not through a weakness, but by reusing an already-trusted, encrypted channel (SSH) — useful for legitimate emergency access, but also a risk: anyone with SSH access to the router can reach otherwise-blocked internal services.

* **When would you use local port forwarding (`-L`)?**
  * When the client (you) needs to reach a remote, internal service as if it were local — e.g. accessing a database or webserver behind a firewall from your own laptop.

* **When would you use remote port forwarding (`-R`)?**
  * When a remote system needs to reach something that only runs locally on your side, or to expose a machine behind NAT/firewall to a server you can reach externally.

* **Which of the two is more common in security?**
  * **Local forwarding (`-L`)** — pentesters/admins usually want to reach something internal from their own machine, not the other way around.

* **Why is SSH port forwarding called a "poor man's VPN"?**
  * Like a VPN, it tunnels encrypted traffic into an internal network over an untrusted network — but without VPN software, using only the SSH client you already have, and limited to specific ports/services instead of full network access.

### Example 1 — view the webpage from the host

```bash
ssh -L 8080:172.30.20.10:80 ca-companyrouter
```
Browsing to `http://localhost:8080` on the laptop shows the "Welcome to the cybersec webserver!" page.

![Webpage via SSH tunnel]<img width="1212" height="944" alt="image" src="https://github.com/user-attachments/assets/ec0f0ec6-dbff-4086-8cc7-0962ac15a0fe" />


### Example 2 — access the database from the host

```bash
ssh -L 3307:172.30.0.15:3306 ca-companyrouter
```
Verified by reading the MariaDB handshake banner on `127.0.0.1:3307` (see above).

### Example 3 — both at the same time, one command

```bash
ssh -L 8080:172.30.20.10:80 -L 3307:172.30.0.15:3306 ca-companyrouter
```
Both the webpage (port 8080) and the database banner (port 3307) are reachable simultaneously through this single SSH session.

## 3. Reaching web without routes on the host (`-J`)

```bash
ssh -J vagrant@192.168.62.253 vagrant@172.30.20.10
```
Logs in on `web` without needing any static route on the laptop for `172.30.0.0/16` — the jump host (companyrouter) handles the routing, the laptop only needs a route to the bastion itself.

## Follow-up (hardening, not yet done)

* Disable `PasswordAuthentication` in `sshd_config` on every machine once key-based login is confirmed stable.
* Create a dedicated non-default user (instead of relying on `vagrant`) for daily SSH access.
