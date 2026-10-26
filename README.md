# 🎓 Certificate Submission & Tracker Portal

> A resilient, cloud-integrated web portal designed for engineering students to submit course completions, workshops, and certifications, paired with an automated **n8n** backend for dynamic form controls, cloud storage, and automated record-keeping in Google Sheets.

Developed for the **Department of Artificial Intelligence & Data Science**, **SNJB's KBJ College of Engineering, Chandwad**.

---

## 📌 Table of Contents

- [Overview](#-overview)
- [Key Features](#-key-features)
- [System Architecture](#-system-architecture)
- [Workflow & Process Automation](#-workflow--process-automation)
  - [Workflow Diagram (Mermaid)](#1-interactive-mermaid-flowchart)
  - [Workflow Image Placeholder](#2-n8n-workflow-visual-screenshot)
- [Technical Stack](#-technical-stack)
- [Deep Dive: Frontend Architecture](#-deep-dive-frontend-architecture)
  - [Dynamic Platform Injection](#1-dynamic-platform-injection)
  - [IndexedDB File Survival Engine](#2-indexeddb-file-survival-engine)
  - [Mobile Form Auto-Recovery (LocalStorage)](#3-mobile-form-auto-recovery)
  - [Client-Side Validation & Safeguards](#4-client-side-validation--safeguards)
- [Backend & Automation Architecture (n8n)](#-backend--automation-architecture-n8n)
  - [Webhook Endpoints](#webhook-endpoints)
  - [Conditional Routing & Storage Schema](#conditional-routing--storage-schema)
- [Local Development & Setup](#-local-development--setup)
- [Deployment Guide](#-deployment-guide)
  - [Frontend Deployment (GitHub Pages / Netlify)](#frontend-deployment)
  - [Backend Deployment (Self-Hosted n8n via Docker & DuckDNS)](#backend-deployment)
- [Google Sheets & Drive Setup](#-google-sheets--drive-setup)
- [Troubleshooting & FAQs](#-troubleshooting--faqs)
- [License & Credits](#-license--credits)

---

## 🚀 Overview

Collecting academic certifications across multiple classes (Second Year, Third Year, Final Year) manually via traditional Google Forms often introduces friction:
- Browsers frequently reload when opening file pickers on mobile devices, discarding student inputs and uploaded files.
- Static dropdowns become stale whenever new learning platforms (Coursera, Infosys Springboard, NPTEL, Udemy, etc.) are introduced.
- Manual verification, renaming files, sorting them into Google Drive folders, and updating departmental spreadsheets requires hundreds of manual hours.

The **Certificate Submission Portal** solves this end-to-end:
1. **Frontend**: A zero-dependency, Google Forms-inspired single-page web app built with vanilla HTML5, CSS3, and modern JavaScript.
2. **Backend**: An event-driven workflow engine powered by **n8n**, routing files directly into organized Google Drive directories and logging metadata into class-specific Google Sheets tabs.

---

## ✨ Key Features

- **Dynamic Platform Population**: Fetches issuing organization options in real-time from an n8n webhook, with automatic retries and fallback handling.
- **Batch Submission Support**: Submit up to **10 certificates** in a single submission session, with dynamic card rendering and individualized metadata.
- **Mobile Resilient File Handling (IndexedDB)**: Files are cached locally as Blobs in IndexedDB the instant they are chosen, preventing data loss when mobile browsers unload memory during file selection.
- **Auto-Save Draft Engine**: Student identification and metadata are continuously saved to `localStorage` and automatically restored on page refresh or browser restart.
- **Defensive Upload Validation**: Enforces aggregate payload limits (< 50 MB), file type verification (`.pdf`, `.jpg`, `.jpeg`, `.png`), and deep client-side form auditing before hitting network endpoints.
- **Accessible & Responsive UX**: Modern card-based UI optimized for desktop, tablets, and smartphones, with high-contrast status modals and visual loading states.

---

## 🏛 System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             CLIENT BROWSER (User)                           │
│  ┌───────────────────────┐  ┌─────────────────────┐  ┌───────────────────┐  │
│  │ UI: Google-Forms Card │  │ IndexedDB Storage   │  │ localStorage      │  │
│  │ Layout & Dynamic Form │  │ (File Blobs Buffer) │  │ (Form Meta Draft) │  │
│  └───────────┬───────────┘  └──────────┬──────────┘  └─────────┬─────────┘  │
└──────────────┼─────────────────────────┼───────────────────────┼────────────┘
               │ (1) Fetch Platforms (GET)│                       │
               │ (2) Submit Form (POST)   │ (Auto-restore)        │ (Auto-save)
               ▼                          ▼                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      n8n WORKFLOW ENGINE (DuckDNS + SSL)                    │
│                                                                             │
│  [ Webhook: Platform Options ] ──► Returns JSON: { options: [...] }         │
│                                                                             │
│  [ Webhook: Multipart Receiver ]                                            │
│        │                                                                    │
│        ▼                                                                    │
│  [ Multipart Binary Parser ] ──► Extract Student Info & Files (1..N)        │
│        │                                                                    │
│        ▼                                                                    │
│  [ Switch Node: Academic Year ]                                             │
│        ├── SY (Second Year) ──► [ SY Drive Folder ] ──► [ Sheet: "SY" Tab ] │
│        ├── TY (Third Year)  ──► [ TY Drive Folder ] ──► [ Sheet: "TY" Tab ] │
│        └── BY (Final Year)  ──► [ BY Drive Folder ] ──► [ Sheet: "BY" Tab ] │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Workflow & Process Automation

Here are **two ways** to visualize the workflow: an interactive Mermaid flowchart (which renders on GitHub natively) and a placeholder to display a screenshot of your n8n workflow canvas.

### 1. Interactive Mermaid Flowchart

```mermaid
flowchart TD
    classDef startEnd fill:#673AB7,stroke:#512DA8,stroke-width:2px,color:#fff;
    classDef client fill:#EDE7F6,stroke:#673AB7,stroke-width:1.5px,color:#202124;
    classDef n8n fill:#FFF8E1,stroke:#FFA000,stroke-width:1.5px,color:#202124;
    classDef cloud fill:#E8F5E9,stroke:#2E7D32,stroke-width:1.5px,color:#202124;

    Start([Student Opens Portal]):::startEnd --> Init[Load Draft & Fetch Platform Options]:::client
    Init -->|GET /webhook/platform-options| N8N_Opt[n8n Platform Provider]:::n8n
    N8N_Opt -->|Return JSON Platforms| BuildUI[Render Form & Dynamic Certificate Cards]:::client

    BuildUI --> UserInput[User Fills Details & Attaches Files]:::client
    UserInput --> CacheFile[(IndexedDB & LocalStorage Cache)]:::client
    UserInput --> Submit{User Clicks Submit}:::client

    Submit -->|Validate Failed| Modal[Show Validation Modal]:::client
    Modal --> UserInput

    Submit -->|Validation Passed| PrepPayload[Pack Multipart FormData < 50MB]:::client
    PrepPayload -->|POST /webhook/form-submit| N8N_Main[n8n Submission Webhook]:::n8n

    subgraph n8n Workflow Automation
        N8N_Main --> BinaryExtract[Extract Form Data & Binary Files]:::n8n
        BinaryExtract --> YearSwitch{Switch by Academic Year}:::n8n
        YearSwitch -->|SY (Second Year)| RouteSY[Route to SY Pipeline]:::n8n
        YearSwitch -->|TY (Third Year)| RouteTY[Route to TY Pipeline]:::n8n
        YearSwitch -->|BY (Final Year)| RouteBY[Route to BY Pipeline]:::n8n
    end

    RouteSY --> DriveUpload[Upload Files to Academic Year Google Drive]:::cloud
    RouteTY --> DriveUpload
    RouteBY --> DriveUpload

    DriveUpload --> SheetLog[Append Rows with File Links to Google Sheet Tab]:::cloud
    SheetLog --> HTTPResponse[Return 200 OK Response]:::n8n

    HTTPResponse --> SuccessOverlay([Display Success Modal & Purge Local Cache]):::startEnd
```

---

### 2. n8n Workflow Visual Screenshot

You can showcase a visual screenshot or export of your n8n node canvas below:

<!-- ==================== WORKFLOW IMAGE PLACEHOLDER START ==================== -->
> ### 🖼️ Workflow Canvas Diagram
> 
> Place your exported n8n workflow image inside your repository (e.g., `docs/workflow.png` or `assets/workflow.png`) and update the link below:
> 
> ```markdown
> ![n8n Certificate Tracker Workflow](docs/workflow.png)
> ```
> 
> _Or embed an interactive canvas link or Loom walk-through here._

<!-- Replace the line below with your image tag once uploaded -->
<p align="center">
  <img src="https://via.placeholder.com/900x420/EDE7F6/512DA8?text=Place+Your+n8n+Workflow+Screenshot+Here+(e.g.+docs/workflow.png)" alt="n8n Workflow Canvas" width="850"/>
</p>

<!-- ===================== WORKFLOW IMAGE PLACEHOLDER END ===================== -->

---

## 🛠 Technical Stack

| Layer | Technology | Description |
|---|---|---|
| **Frontend Language** | HTML5, Vanilla JavaScript (ES6+) | Lightweight, zero-dependency, ultra-fast initial load |
| **Styling** | Vanilla CSS3 (Custom Design System) | Google Material / Forms aesthetic, responsive card layouts |
| **Client Storage** | IndexedDB & LocalStorage | Offline caching for file blobs and draft metadata recovery |
| **Workflow Automation** | [n8n](https://n8n.io/) | Self-hosted visual workflow orchestrator |
| **Domain & Network** | DuckDNS & Reverse Proxy (HTTPS) | Dynamic DNS with automated SSL/TLS termination |
| **Cloud Storage** | Google Drive API | Automated folder management & file preservation |
| **Database / Records** | Google Sheets API | Structured academic record tracking by year & division |

---

## 🔬 Deep Dive: Frontend Architecture

The frontend is housed in [index.html](file:///e:/Antigravity/Certificate_Tracker/index.html). It implements several enterprise-grade patterns to ensure high reliability across mobile and low-bandwidth connections.

### 1. Dynamic Platform Injection
Instead of hardcoding course platforms, the portal makes an asynchronous call upon initialization:
```javascript
const PLATFORM_OPTIONS_WEBHOOK_URL = N8N_BASE_URL + "/webhook/616164fa-f480-4486-97a7-b9a41421032c";
```
- Fetches active platforms (e.g., *Coursera*, *NPTEL*, *Infosys Springboard*, *Udemy*, *AWS Academy*, *HackerRank*).
- A fallback retry mechanism ensures that if n8n is restarting, the UI prompts a polite reconnect message before attempting to generate the certificate block selects.

### 2. IndexedDB File Survival Engine
Browsers restrict `<input type="file">` state from persisting across reloads for security reasons. On mobile devices with strict RAM limits, switching between camera/file manager and the browser often triggers a tab reload, losing attached documents.
- Whenever a file is selected:
  ```javascript
  await filePut('cert_' + i, { blob: f, name: f.name, type: f.type, t: Date.now() });
  ```
- Upon reload, files are fetched from the `certPortalFiles` IndexedDB store, wrapped inside standard `File` objects, and programmatically reinjected into the form input using the `DataTransfer` API:
  ```javascript
  const f = new File([rec.blob], rec.name, { type: rec.type });
  const dt = new DataTransfer();
  dt.items.add(f);
  input.files = dt.files;
  ```

### 3. Mobile Form Auto-Recovery
The form continuously listens for input changes and writes student metadata (`studentName`, `rollNo`, `academicYear`, `certs` array) to `localStorage` under `certPortalDraft_v1`.
- Cached drafts expire automatically after **6 hours** to prevent outdated submissions.
- On successful submission, drafts and local IndexedDB binary caches are cleanly purged.

### 4. Client-Side Validation & Safeguards
- **In-Memory File ArrayBuffer Conversion**: Clones raw array buffers into new `File` objects prior to upload to prevent pointer corruption from cloud-synced files (Google Drive/OneDrive).
- **Aggregate Size Enforcer**: Validates that all files combined do not exceed **50 MB** before making network requests.
- **AbortController Network Timeout**: Guarantees requests do not hang infinitely on bad networks (5-minute hard limit).
- **Validation Modal Dialog**: Aggregates missing fields into a single, clean modal dialog and scrolls smoothly to the first erroneous element.

---

## ⚙️ Backend & Automation Architecture (n8n)

The backend is completely serverless and event-driven, orchestrated by an **n8n** automation instance.

### Webhook Endpoints

| Endpoint | Method | Purpose | Payload |
|---|---|---|---|
| `/webhook/616164fa-f480-4486-97a7-b9a41421032c` | `GET` | Fetches available issuing platforms | None |
| `/webhook/64b07f1a-e0c3-4714-9711-05b466dd07ae` | `POST` | Processes certificate submissions | `multipart/form-data` |

### Conditional Routing & Storage Schema

```
n8n Flow Pipeline:
  [Webhook POST]
       │
       ▼
  [Function / Code Node: Flatten & Parse Binary Fields]
       │
       ▼
  [Switch Node: Academic Year]
       │
       ├─► Case 1: "SY (Second Year)"  ──► Upload to Drive/SY ──► Append to Sheet/SY
       ├─► Case 2: "TY (Third Year)"   ──► Upload to Drive/TY ──► Append to Sheet/TY
       └─► Case 3: "BY (Final Year)"   ──► Upload to Drive/BY ──► Append to Sheet/BY
```

**Uploaded File Naming Convention in Google Drive:**
```
[RollNo]_[StudentName]_[Platform]_[CertName].[ext]
Example: 45_Rohit_Sharma_Coursera_Machine_Learning.pdf
```

---

## 💻 Local Development & Setup

### Prerequisites
- Modern web browser (Chrome, Edge, Firefox, Safari).
- (Optional) Local n8n instance installed via npm or Docker:
  ```bash
  npx n8n
  # or
  docker run -it --rm --name n8n -p 5678:5678 -v ~/.n8n:/home/node/.n8n docker.n8n.io/n8nio/n8n
  ```

### Step-by-Step Setup

1. **Clone the repository:**
   ```bash
   git clone https://github.com/ssarkan07/Certification_Tracker.git
   cd Certification_Tracker
   ```

2. **Configure Webhook Endpoints:**
   Open [index.html](file:///e:/Antigravity/Certificate_Tracker/index.html#L918-L926) and update the configuration variables with your local or live n8n instance URLs:
   ```javascript
   // ---------- CONFIGURE THESE TWO URLS ----------
   const N8N_BASE_URL = "http://localhost:5678"; // or "https://your-n8n-domain.com"
   const PLATFORM_OPTIONS_WEBHOOK_URL = N8N_BASE_URL + "/webhook/<your-platform-options-webhook-id>";
   const FORM_SUBMIT_WEBHOOK_URL = N8N_BASE_URL + "/webhook/<your-form-submit-webhook-id>";
   // -----------------------------------------------
   ```

3. **Launch Local Server:**
   You can serve the directory using Python, Node, or VS Code Live Server:
   ```bash
   # Using Python
   python -m http.server 8000

   # Or using Node http-server
   npx http-server . -p 8000
   ```
   Open `http://localhost:8000` in your web browser.

---

## 🌐 Deployment Guide

### Frontend Deployment

Because the application is static (HTML, CSS, JS), it can be deployed to any static host for free:

#### Option A: GitHub Pages
1. Push your repository to GitHub:
   ```bash
   git push origin main
   ```
2. In your GitHub repository, navigate to **Settings** > **Pages**.
3. Under **Build and deployment** > **Source**, choose `Deploy from a branch`.
4. Select `main` (or `master`) branch and root directory `/`, then click **Save**.
5. Your portal will be accessible at `https://<your-username>.github.io/Certification_Tracker/`.

#### Option B: Netlify
1. Log in to [Netlify](https://www.netlify.com/).
2. Select **Add new site** > **Import an existing project** > connect your GitHub repo.
3. Leave build command empty, set publish directory to `.`.
4. Click **Deploy Site**.

---

### Backend Deployment (Self-Hosted n8n)

For production, run n8n on a Cloud VPS (DigitalOcean, AWS EC2, Hetzner, etc.) or home server with DuckDNS:

1. **Setup DuckDNS**:
   - Register a free domain (e.g. `snjbcoe.duckdns.org`) pointing to your server's public IP.
2. **Run n8n with Docker Compose**:
   ```yaml
   version: "3.8"
   services:
     n8n:
       image: docker.n8n.io/n8nio/n8n
       restart: always
       ports:
         - "5678:5678"
       environment:
         - N8N_HOST=snjbcoe.duckdns.org
         - N8N_PORT=5678
         - N8N_PROTOCOL=https
         - WEBHOOK_URL=https://snjbcoe.duckdns.org/
       volumes:
         - n8n_data:/home/node/.n8n

   volumes:
     n8n_data:
   ```
3. **Configure Reverse Proxy with SSL (Caddy / Nginx)**:
   Ensure HTTPS is configured to enable secure `fetch` operations from mobile browsers without mixed-content errors.

---

## 📊 Google Sheets & Drive Setup

### Google Sheets Schema

Create a Google Sheet with 3 separate sheets (tabs) titled exactly:
- `SY (Second Year)`
- `TY (Third Year)`
- `BY (Final Year)`

Each tab should include the following standard column headers:

| Col A | Col B | Col C | Col D | Col E | Col F | Col G |
|---|---|---|---|---|---|---|
| **Timestamp** | **Student Name** | **Roll Number** | **Platform** | **Certificate Name** | **Completion Date** | **Drive Link** |

### Google Drive Setup

Create a parent folder in Google Drive:
- `Certificates_Archive`
  - `SY_Certificates`
  - `TY_Certificates`
  - `BY_Certificates`

In the n8n Google Drive nodes, link each year branch to the respective target folder ID.

---

## ❓ Troubleshooting & FAQs

### Q: Why do certificates disappear on page reload?
> **Answer**: They don't! The portal uses **IndexedDB** to automatically store file binaries locally. If the browser tab reloads, an info banner will notify the student that their data and certificates were restored.

### Q: What happens if a file exceeds 50 MB?
> **Answer**: The frontend automatically evaluates aggregate byte sizes before dispatching. A friendly user error is displayed, advising the student to compress the PDF or submit fewer certificates at once.

### Q: Platforms are not appearing in the dropdown?
> **Answer**: Ensure your n8n workflow for `PLATFORM_OPTIONS_WEBHOOK_URL` is active and reachable over HTTPS. The app will automatically attempt a retry and present an inline retry indicator.

---

## 👤 Author & Credits

- **Developer**: [Arkan Shaikh](https://github.com/ssarkan07)
- **Institution**: SNJB's Late Sau. K. B. Jain College of Engineering, Chandwad
- **Department**: Department of Artificial Intelligence & Data Science
- **Project Repository**: [Certification_Tracker](https://github.com/ssarkan07/Certification_Tracker)

---

<p align="center">
  <b>Department of AI & Data Science</b> • <i>Empowering Students Through Continuous Learning</i>
</p>