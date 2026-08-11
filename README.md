# Evaluating-UFW-for-a-Docker-Based-OpenMediaVault-Server
Investigated the interaction between UFW, Docker, and Tailscale on an OpenMediaVault server, documenting why UFW was intentionally left installed but inactive after evaluating the underlying Linux firewall architecture and official documentation.


Network Access Policy v1 and implementation
Service	Port	Authorized Users/Devices	Allowed Networks		
SSH	22	Me	LAN+Tailscale		
OMV	80	Me	LAN+Tailscale		
Nextcloud	8080	Me and wife	LAN+Tailscale		
SMB	445/139	Trusted Personal Devices	LAN (Tailscale Optional Later		
Everything Else	Any	None	Deny		
					
					
			
Implement Host Firewall Policy v1
Objective
Implement a firewall policy that:
•	Allows only required services. 
•	Preserves remote administration. 
•	Does not interrupt existing services. 
•	Can be easily expanded later. 
________________________________________
Step 0 — Pre-Change Checklist
Before touching anything, let's answer these.
Current SSH Access
✅ Working?
Yes.
________________________________________
Current OMV Web Interface
✅ Working?
Yes.
________________________________________
Current Nextcloud
✅ Working?
Yes.
________________________________________
SMB
✅ Working?
(yes.)
________________________________________
Recovery Plan
If we somehow block ourselves...
How do we recover? 
Connect a keyboard and monitor directly to the server, log in locally, and correct the firewall configuration.
________________________________________
Implementation: 
Connected to server via
Ssh jordan@192.168.0.2
Refreshed package Metadata
Verified I have latest packages list via sudo apt update
Installed UFW via sudo apt install ufw
Verified installation by checking the status via sudo ufw status verbose
Status came back inactive. 
Investigated default behavior: 
Ran sudo ufw status numbered 
Status came back inactive.
Ran sudo ufw show raw
Which did show the underlying firewall configuration, but not in the way I expected. I had predicted sudo ufw show raw would show the underlying configuration even though UFW is inactive.
However, Tailscale and Debian are already interacting with the host firewall base configuration with accept policies. 
After looking at my firewall output, I think it's actually more accurate to draw the flow like this:
                Internet
                     │
        ┌────────────┴────────────┐
        │                         │
   Tailscale                 Local LAN
        │                         │
        └────────────┬────────────┘
                     │
        Linux Kernel Firewall
     (iptables / nftables)
          ▲          ▲
          │          │
     Docker      UFW (not active yet)
          │
          ▼
     OpenMediaVault
          │
          ▼
    Nextcloud, SMB, SSH

Since I can already see Docker and Tailscale are managing firewall rules, before enabling another firewall manager, I need to understand how these interact.

I went and checked the official documentation from Docker and Tailscale, not blog posts.
Docker:
Docker's documentation explicitly states:
Docker and UFW are incompatible by default.
The reason is that Docker publishes container ports by creating its own firewall/NAT rules before traffic reaches the chains that UFW normally manages. That means a published Docker port can be reachable even if UFW appears to deny it. 
Tailscale:
Tailscale, on the other hand, does document using UFW together. In fact, they recommend a pattern very similar to the security policy I designed:
•	Allow traffic on the tailscale0 interface. 
•	Set the default incoming policy to deny. 
•	Remove ordinary SSH access if you want SSH only over Tailscale. 
So Tailscale isn't my concern.
Docker is.

At first I said:
"Let's install UFW."
That was still the right decision because I learned how package management works and how to investigate a Linux service.
But after reading Docker's documentation, I do not think enabling UFW is the right next step on this server.
My goal is not:
Learn UFW.
My goal is:
Build a secure self-hosted infrastructure.

I decided to leave UFW installed but inactive Rather than forcing a familiar tool into the environment, I chose the solution that best fit the existing architecture after reviewing the official documentation.
Because right now my host runs:
•	OMV 
•	Docker 
•	Nextcloud 
•	Tailscale 
•	SMB 
I'd rather design firewalling around Docker's networking model and Tailscale's access controls than force UFW into the mix and potentially create confusing behavior. That's also in line with Docker's own guidance that disabling Docker's firewall management is likely to break container networking.

## Tradeoffs Considered

### Automatic Security Updates

Benefits
- Reduces the window of exposure to known vulnerabilities.
- Decreases administrative overhead.
- Improves baseline system security.

Risks
- Updates may introduce compatibility issues.
- Service interruptions are possible if packages change behavior.
- Major version upgrades should be reviewed before deployment.

Decision

Enable automatic installation of security updates only.
Continue reviewing feature and major version updates manually.

Revised Infrastructure Roadmap:
1.	Understand Linux firewall architecture (iptables/nftables). 
2.	Understand how Docker publishes ports. 
3.	Use Docker networking intentionally (bind only what you need). 
4.	Use Tailscale as your remote access boundary. 
5.	If I later deploy a plain Debian or Ubuntu VM without Docker, that's where I'll build a full UFW project from scratch.
original roadmap:
1.	✅ SSH Hardening 
2.	✅ Windows SSH Agent 
3.	UFW 
4.	Automatic Security Updates 
5.	Fail2Ban 
6.	Backups 
7.	Docker Services 
8.	Monitoring


Updated Roadmap
Step	Status	Why
SSH Hardening	✅ Complete	Strong authentication
Windows SSH Agent	✅ Complete	Quality of life without sacrificing security
Linux Firewall Architecture (Docker/Tailscale)	✅ Investigated	Learned why UFW isn't the right choice for this host today
Automatic Security Updates	Next	Universally applicable and excellent Linux administration practice
Fail2Ban	After updates	Good security experience, even if SSH isn't internet-exposed today
OMV Configuration Backups	After Fail2Ban	Disaster recovery
Docker Services	Continue	Jellyfin, Immich, etc.
Monitoring	Last	Uptime Kuma, Grafana, Prometheus

