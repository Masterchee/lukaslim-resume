# Project Blueprint: www.lukaslim.com — Tech Education Hub

## 1. Project Overview & Goals
- **Domain:** `www.lukaslim.com` (Routed via AWS Route 53)
- **Objective:** Transform a static personal resume site into an authoritative, practitioner-led tech education hub and personal brand platform.
- **Core Philosophy:** Teach fundamental tech concepts from a **technician's perspective**—focusing on mental models, real-world execution, and practical troubleshooting ("What happens when it breaks?").

---

## 2. Site Architecture & Navigation
Keep the existing resume intact as a credibility anchor while expanding into structured learning tracks.

```text
www.lukaslim.com/
├── index.html               # Landing Page: Bio, Resume, & CTA to Learning Tracks
├── learn/                   # Knowledge Hub Overview
│   ├── networking-osi.md    # Track 1: OSI Layers & Troubleshooting
│   ├── operating-systems.md # Track 2: OS Internals & CLI
│   ├── cloud-aws.md         # Track 3: Cloud Infrastructure (Live Case Study)
│   └── databases.md         # Track 4: Data Storage, SQL vs. NoSQL
└── work-with-me.md          # Consulting, Mentorship, & Infrastructure Audits
```

### Technical Stack Recommendation (AWS + Kiro Workflow)
- **Framework:** Lightweight Static Site Generator (Astro or Hugo) or clean Semantic HTML/CSS + Markdown parser.
- **Hosting:** AWS S3 (Static Website Hosting) + AWS CloudFront (CDN & HTTPS via ACM).
- **DNS:** AWS Route 53 (already configured).

---

## 3. Core Curriculum Pillars (The Technician's Angle)

### Pillar 1: OSI Layers (Networking)
- **The Beginner Struggle:** Memorizing the 7 layers without understanding practical application.
- **The Technician's Angle:** Teach through bottom-up troubleshooting.
  - *Layer 1 (Physical):* Cables, link lights, hardware interfaces.
  - *Layer 2 (Data Link):* MAC addresses, local switching, ARP tables.
  - *Layer 3 (Network):* IP addressing, subnetting, routing tables (`ping`, `traceroute`).
  - *Layer 4 (Transport):* TCP vs. UDP, ports, firewalls, connection states (`netstat`, `ss`).
  - *Layer 7 (Application):* DNS resolution (`dig`, `nslookup`), HTTP status codes, SSL/TLS handshakes.

### Pillar 2: Operating Systems
- **The Beginner Struggle:** Treating the OS as an abstract black box.
- **The Technician's Angle:** Demystify resource management using real terminal workflows.
  - *Boot Process:* BIOS/UEFI -> Bootloader -> Kernel -> Init/Systemd.
  - *Processes & Memory:* CPU scheduling, RAM vs. Swap, inspecting live workloads (`top`, `htop`, `ps`).
  - *File Systems & Permissions:* Inodes, mounting disks (`lsblk`, `df`), ownership and access control (`chmod`, `chown`).

### Pillar 3: Cloud Technology (AWS)
- **The Beginner Struggle:** Overwhelmed by 200+ managed services.
- **The Technician's Angle:** Use `www.lukaslim.com` itself as a live architectural case study.
  - *DNS & Routing:* How Route 53 resolves the domain name.
  - *Storage & Delivery:* How S3 stores static assets and CloudFront caches them globally at the edge.
  - *Security & IAM:* Principle of least privilege, bucket policies, and SSL certificates.

### Pillar 4: Databases
- **The Beginner Struggle:** Confusion over when to choose SQL vs. NoSQL.
- **The Technician's Angle:** Focus on how data hits the disk and how applications connect.
  - *Storage Mechanics:* Rows/Tables (Relational) vs. Documents/Key-Value (NoSQL).
  - *Performance Basics:* Why queries slow down and how indexing works under the hood.
  - *Connection Flows:* Connection pooling, network latency between compute and DB, and basic backup strategies.

---

## 4. Standard Lesson Template
Every educational guide published on the site should follow this repeatable 4-part structure:

1. **The 30-Second Mental Model:** A plain-English analogy paired with a clean ASCII or SVG diagram.
2. **How It Works in Practice:** Real terminal commands, packet flows, or AWS architecture configurations.
3. **What Happens When It Breaks (Failure Modes):** Common real-world errors, symptoms, and step-by-step diagnostic commands.
4. **Hands-On Lab:** A safe, copy-paste exercise readers can run locally or in a free-tier cloud environment.

---

## 5. Brand & Business Growth Roadmap
- **Phase 1 (Foundation):** Launch `/learn` with the 4 core pillar guides using Kiro to scaffold the site layout and Markdown rendering.
- **Phase 2 (Distribution):** Publish condensed visual diagrams and troubleshooting takeaways on LinkedIn/Medium linking back to `www.lukaslim.com`.
- **Phase 3 (Lead Capture):** Add an unobtrusive newsletter signup at the bottom of each guide for readers who want updates on new field guides.
- **Phase 4 (Monetization):** Activate the `/work-with-me` page offering 1-on-1 cloud/Linux mentorship, technical training, or small-business infrastructure reviews.