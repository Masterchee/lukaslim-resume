# 🌐 Personal Portfolio & Cloud-Connected View Counter

An enterprise-ready, lightweight, and responsive personal portfolio built with single-file HTML5/CSS3/JS, integrated with AWS serverless architecture to track real-time page views.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![AWS](https://img.shields.io/badge/AWS-Lambda%20%7C%20Cloud-orange)
![Stack](https://img.shields.io/badge/Stack-HTML5%20%7C%20CSS3%20%7C%20JS-green)

---

## 📌 Overview

This project serves as a lightweight, framework-free web portfolio tailored for IT, Cloud, and Data Center professionals. It features an integrated serverless backend call to an **AWS Lambda Function URL** to update and display page visits dynamically without heavy client-side frameworks.

### Key Features
- **Serverless Analytics**: Asynchronous `POST` request to an AWS Lambda API endpoint for live view counting.
- **Pure Web Performance**: Zero external JavaScript frameworks, dependencies, or bloated libraries.
- **Responsive & Modern UI**: Dark-mode primary design, dynamic hover states, responsive grids, and clean layout scaling across devices.
- **Interactive UX**: Built-in smooth scrolling navigation and scroll-triggered fade-in animations via `IntersectionObserver`.

---

## 🏗️ Architecture
                       ┌─────────────────────────┐
                       │  GitHub / CodeCommit    │
                       └────────────┬────────────┘
                                    │ (Auto-Build / CI-CD)
                                    ▼
                       ┌─────────────────────────┐
                       │    AWS Amplify          │
                       │ (Hosting & Edge Delivery│
                       └────────────┬────────────┘
                                    │ Serves Site
                                    ▼  
    ┌───────────────────────────────────────────────────────────────────────┐
    │                       PRESENTATION TIER                               │
    │                         User Browser                                  │
    └───────────────┬───────────────────────────────────────┬───────────────┘
                    │ Fetches Images / Assets               │ POST (View Count)
                    ▼                                       ▼
    ┌──────────────────────────────┐        ┌──────────────────────────────┐
    │          DATA TIER           │        │          LOGIC TIER          │
    │       Amazon S3 Bucket       │        │   AWS Lambda (Function URL)  │
    │  (Media & Image Storage)     │        │     (Python Runtime)         │
    └──────────────────────────────┘        └───────────────┬──────────────┘
                                                        │
                                                        │ Atomic Update
                                                        ▼
                                        ┌──────────────────────────────┐
                                        │          DATA TIER           │
                                        │       Amazon DynamoDB        │
                                        │    (Counter Persistence)     │
                                        └──────────────────────────────┘


---

## 🛠️ Tech Stack

- **Frontend**: HTML5, CSS3 (Custom CSS Variables, Flexbox, CSS Grid), Vanilla JavaScript (ES6+).
- **Logic API**: AWS Lambda (Function URL integration).
- **Backend**: S3 Bucket static html, DynamoDB

---

## 🚀 Quick Start

1. **Clone the Repository**
   ```bash
   git clone [https://github.com/Masterchee/lukaslim-resume.git](https://github.com/Masterchee/lukaslim-resume.git)
   cd portfolio-site
