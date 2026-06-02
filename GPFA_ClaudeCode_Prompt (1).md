# GPFA Scholarship Portal — Claude Code Prompt
# Paste the entire block below into your terminal after installing Claude Code:
# npm install -g @anthropic-ai/claude-code && claude

---

## PROMPT (copy everything from here down into Claude Code)

Build the **Gbowee Peace Foundation Africa (GPFA) Scholarship Portal** — a complete, production-ready web application that uses **GitHub as the entire backend**:

- **GitHub Issues** = scholarship application storage (one Issue per applicant)
- **GitHub Labels** = application status (pending, under-review, approved, rejected)
- **GitHub Releases / Assets** = uploaded documents (WASSCE, transcripts, etc.)
- **GitHub Pages** = static hosting for the frontend
- **GitHub Actions** = email notifications on status change
- **GitHub Milestones** = scholarship cycles (e.g., "2024–2025 Cycle")
- **GitHub REST API** (via Octokit) = all CRUD from the browser

---

### STEP 1 — Project scaffold

```bash
npx create-next-app@latest gpfa-portal \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*"
cd gpfa-portal
npm install @octokit/rest @octokit/auth-token \
  react-hook-form zod @hookform/resolvers \
  react-signature-canvas \
  react-dropzone \
  lucide-react \
  next-themes \
  sonner \
  clsx tailwind-merge
npx shadcn-ui@latest init -y
npx shadcn-ui@latest add button input label select textarea \
  card badge progress tabs dialog alert separator
```

---

### STEP 2 — Environment variables

Create `.env.local`:

```
NEXT_PUBLIC_GITHUB_OWNER=gbowee-peace-foundation
NEXT_PUBLIC_GITHUB_REPO=scholarship-applications
NEXT_PUBLIC_GITHUB_LABELS_REPO=scholarship-applications
GITHUB_ADMIN_TOKEN=ghp_REPLACE_WITH_ADMIN_PAT   # server-side only — full repo access
NEXT_PUBLIC_GITHUB_PUBLIC_TOKEN=ghp_REPLACE_WITH_PUBLIC_PAT  # read Issues + create Issues only
NEXT_PUBLIC_APP_URL=https://gbowee-peace-foundation.github.io/scholarship-portal
```

Create `.env.example` with the same keys but empty values. Add `.env.local` to `.gitignore`.

---

### STEP 3 — GitHub backend service (`src/lib/github.ts`)

Create a full GitHub service layer using `@octokit/rest`:

```typescript
// src/lib/github.ts
import { Octokit } from "@octokit/rest";

const OWNER = process.env.NEXT_PUBLIC_GITHUB_OWNER!;
const REPO  = process.env.NEXT_PUBLIC_GITHUB_REPO!;

// Public client — applicants use this to submit
export const publicOctokit = new Octokit({
  auth: process.env.NEXT_PUBLIC_GITHUB_PUBLIC_TOKEN,
});

// Admin client — only used in Server Actions / API routes
export const adminOctokit = new Octokit({
  auth: process.env.GITHUB_ADMIN_TOKEN,
});

// STATUS LABELS — created in GitHub repo beforehand
export const STATUS_LABELS = {
  pending:     { name: "status: pending",      color: "FFA500" },
  review:      { name: "status: under-review", color: "0075ca" },
  approved:    { name: "status: approved",     color: "2ea44f" },
  rejected:    { name: "status: rejected",     color: "d73a49" },
} as const;

// COUNTY LABELS
export const COUNTY_LABELS = [
  "Bomi","Bong","Gbarpolu","Grand Bassa","Grand Cape Mount",
  "Grand Gedeh","Grand Kru","Lofa","Margibi","Maryland",
  "Montserrado","Nimba","River Cess","River Gee","Sinoe",
].map(c => ({ name: `county: ${c}`, color: "e4e669" }));

// ─── APPLICATION SUBMISSION ─────────────────────────────────────────────────

export interface ApplicationData {
  // Step 1
  surname: string; firstName: string; middleName?: string;
  gender: string; citizenship: string; county: string;
  address: string; gradeLevel: string; phone: string;
  email: string; dob: string; prevApplied: boolean;
  desiredMajor: string; heardVia: string;
  docChecklist: string[];

  // Step 2
  motherName?: string; motherStatus?: string; motherOccupation?: string;
  motherPhone?: string; motherEmail?: string;
  fatherName?: string; fatherStatus?: string; fatherOccupation?: string;
  fatherPhone?: string; fatherEmail?: string;
  sponsorName?: string; sponsorRelation?: string; sponsorPhone?: string;
  relativeGpfa?: string; dismissalHistory?: string;

  // Step 3
  schoolHistory: Array<{ name: string; location: string; years: string; cert: string; gpa: string }>;
  honors?: string; activities?: string; workExperience?: string;

  // Step 4
  universitiesApplied?: string;
  // File uploads stored as GitHub Release asset URLs
  billingUrls?: string[]; gradeUrls?: string[];
  wassceUrls?: string[]; transcriptUrls?: string[];

  // Step 5
  fatherIncome?: string; motherIncome?: string; sponsorIncome?: string;
  totalIncome?: string; dependents?: number;
  housing?: string; vehicles?: string; debts?: string;
  tuitionCost?: string; efc?: string; amountRequested?: string;
  otherScholarships?: string; letterOfNeed?: string;
  financialDocUrls?: string[];

  // Step 6 (signature stored as base64 data URL)
  applicantSignature?: string; signDate?: string;
  applicantPrintedName?: string;
  guardianSignature?: string; guardianName?: string; guardianRelation?: string;
}

// Format application as a well-structured GitHub Issue body (Markdown)
function formatIssueBody(data: ApplicationData, refNumber: string): string {
  return `
## Application Reference: ${refNumber}
> **Submitted:** ${new Date().toISOString()}

---

## 👤 Personal Information
| Field | Value |
|-------|-------|
| **Full Name** | ${data.surname}, ${data.firstName} ${data.middleName ?? ""} |
| **Gender** | ${data.gender} |
| **Citizenship** | ${data.citizenship} |
| **County of Origin** | ${data.county} |
| **Date of Birth** | ${data.dob} |
| **Grade / Year** | ${data.gradeLevel} |
| **Phone** | +231 ${data.phone} |
| **Email** | ${data.email} |
| **Desired Major** | ${data.desiredMajor} |
| **Heard Via** | ${data.heardVia} |
| **Previous GPFA Applicant** | ${data.prevApplied ? "Yes" : "No"} |
| **Home Address** | ${data.address} |

**Document Checklist:** ${data.docChecklist.join(", ")}

---

## 👨‍👩‍👧 Family & Guardian Information
### Mother
| Field | Value |
|-------|-------|
| Name | ${data.motherName ?? "—"} |
| Status | ${data.motherStatus ?? "—"} |
| Occupation | ${data.motherOccupation ?? "—"} |
| Phone | ${data.motherPhone ? "+231 " + data.motherPhone : "—"} |
| Email | ${data.motherEmail ?? "—"} |

### Father
| Field | Value |
|-------|-------|
| Name | ${data.fatherName ?? "—"} |
| Status | ${data.fatherStatus ?? "—"} |
| Occupation | ${data.fatherOccupation ?? "—"} |
| Phone | ${data.fatherPhone ? "+231 " + data.fatherPhone : "—"} |
| Email | ${data.fatherEmail ?? "—"} |

### Sponsor / Guardian
| Field | Value |
|-------|-------|
| Name | ${data.sponsorName ?? "—"} |
| Relationship | ${data.sponsorRelation ?? "—"} |
| Phone | ${data.sponsorPhone ? "+231 " + data.sponsorPhone : "—"} |

**Relative with prior GPFA scholarship:** ${data.relativeGpfa ?? "None"}
**Dismissal/Suspension history:** ${data.dismissalHistory ?? "None"}

---

## 🎓 Academic Information

### School History
| School | Location | Years | Certificate | GPA |
|--------|----------|-------|-------------|-----|
${(data.schoolHistory ?? []).map(s => `| ${s.name} | ${s.location} | ${s.years} | ${s.cert} | ${s.gpa} |`).join("\n")}

**Honors & Distinctions:** ${data.honors ?? "—"}
**Extracurricular & Community Service:** ${data.activities ?? "—"}
**Work Experience:** ${data.workExperience ?? "—"}

---

## 📎 Uploaded Documents
${data.billingUrls?.length    ? "**Semester Billing:** " + data.billingUrls.join(", ")    : ""}
${data.gradeUrls?.length      ? "**Grade Sheets:** " + data.gradeUrls.join(", ")          : ""}
${data.wassceUrls?.length     ? "**WASSCE Results:** " + data.wassceUrls.join(", ")       : ""}
${data.transcriptUrls?.length ? "**Transcripts:** " + data.transcriptUrls.join(", ")     : ""}

**Universities Applied To:** ${data.universitiesApplied ?? "—"}

---

## 💰 Financial Aid Application
| Field | Value |
|-------|-------|
| Father's Income | ${data.fatherIncome ?? "—"} |
| Mother's Income | ${data.motherIncome ?? "—"} |
| Sponsor Income | ${data.sponsorIncome ?? "—"} |
| Total Household Income | ${data.totalIncome ?? "—"} |
| Number of Dependents | ${data.dependents ?? "—"} |
| Housing | ${data.housing ?? "—"} |
| Vehicles | ${data.vehicles ?? "—"} |
| Outstanding Debts | ${data.debts ?? "—"} |
| Estimated Tuition | ${data.tuitionCost ?? "—"} |
| Expected Family Contribution | ${data.efc ?? "—"} |
| **GPFA Amount Requested** | **${data.amountRequested ?? "—"}** |
| Other Scholarships | ${data.otherScholarships ?? "—"} |

### Letter of Need
${data.letterOfNeed ?? "—"}

${data.financialDocUrls?.length ? "**Financial Documents:** " + data.financialDocUrls.join(", ") : ""}

---

## ✍️ Declaration & Signature
| Field | Value |
|-------|-------|
| Applicant Printed Name | ${data.applicantPrintedName ?? "—"} |
| Date Signed | ${data.signDate ?? "—"} |
| Guardian Name | ${data.guardianName ?? "—"} |
| Guardian Relationship | ${data.guardianRelation ?? "—"} |

${data.applicantSignature ? `**Applicant Signature (base64):** [Signature on file]` : ""}
${data.guardianSignature  ? `**Guardian Signature (base64):** [Signature on file]`  : ""}

---
*Submitted via GPFA Scholarship Portal — ${process.env.NEXT_PUBLIC_APP_URL}*
`.trim();
}

// Submit a new application as a GitHub Issue
export async function submitApplication(data: ApplicationData) {
  const year = new Date().getFullYear();
  const rand = Math.floor(100 + Math.random() * 900);
  const refNumber = `GPFA-${year}-${rand}`;

  const labels = [
    STATUS_LABELS.pending.name,
    `county: ${data.county}`,
    "scholarship-application",
  ];

  const response = await publicOctokit.issues.create({
    owner: OWNER,
    repo: REPO,
    title: `[${refNumber}] ${data.surname}, ${data.firstName} — ${data.county} — ${data.desiredMajor}`,
    body: formatIssueBody(data, refNumber),
    labels,
  });

  return { refNumber, issueNumber: response.data.number, issueUrl: response.data.html_url };
}

// ─── ADMIN: FETCH ALL APPLICATIONS ─────────────────────────────────────────

export async function fetchApplications(filters?: {
  status?: string; county?: string; search?: string;
}) {
  const labels = ["scholarship-application"];
  if (filters?.status) labels.push(`status: ${filters.status}`);
  if (filters?.county) labels.push(`county: ${filters.county}`);

  const response = await adminOctokit.issues.listForRepo({
    owner: OWNER, repo: REPO,
    labels: labels.join(","),
    state: "open",
    per_page: 100,
    sort: "created", direction: "desc",
  });

  return response.data.map(issue => {
    const titleMatch = issue.title.match(/\[([^\]]+)\] (.+?) — ([^—]+) — (.+)/);
    return {
      issueNumber: issue.number,
      refNumber:   titleMatch?.[1] ?? issue.number.toString(),
      name:        titleMatch?.[2] ?? "Unknown",
      county:      titleMatch?.[3]?.trim() ?? "—",
      major:       titleMatch?.[4]?.trim() ?? "—",
      status:      issue.labels.find((l: any) => l.name?.startsWith("status:"))?.name?.replace("status: ","") ?? "pending",
      createdAt:   issue.created_at,
      body:        issue.body ?? "",
      labels:      issue.labels.map((l: any) => l.name),
    };
  });
}

// ─── ADMIN: UPDATE STATUS ───────────────────────────────────────────────────

export async function updateApplicationStatus(
  issueNumber: number,
  newStatus: keyof typeof STATUS_LABELS
) {
  // Remove all existing status labels
  const existingLabels = await adminOctokit.issues.listLabelsOnIssue({
    owner: OWNER, repo: REPO, issue_number: issueNumber,
  });
  const statusLabels = existingLabels.data
    .filter(l => l.name.startsWith("status:"))
    .map(l => l.name);
  
  for (const label of statusLabels) {
    await adminOctokit.issues.removeLabel({
      owner: OWNER, repo: REPO,
      issue_number: issueNumber, name: label,
    });
  }

  // Add new status label
  await adminOctokit.issues.addLabels({
    owner: OWNER, repo: REPO,
    issue_number: issueNumber,
    labels: [STATUS_LABELS[newStatus].name],
  });

  // Add comment with status change note
  await adminOctokit.issues.createComment({
    owner: OWNER, repo: REPO,
    issue_number: issueNumber,
    body: `**Status updated to: ${newStatus.toUpperCase()}**\n\n*Updated by GPFA Admin on ${new Date().toLocaleString()}*`,
  });

  if (newStatus === "rejected") {
    await adminOctokit.issues.update({
      owner: OWNER, repo: REPO,
      issue_number: issueNumber, state: "closed",
    });
  }
}

// ─── FILE UPLOAD (GitHub Issues Comments with base64 fallback) ──────────────
// For documents: upload file content as a gist or encode as base64 data URL
// stored in Issue body. For production: swap this for GitHub Releases upload.

export async function uploadDocumentAsGist(
  file: File, applicantName: string
): Promise<string> {
  const buffer = await file.arrayBuffer();
  const base64 = btoa(String.fromCharCode(...new Uint8Array(buffer)));
  
  // Store as a secret Gist — returns URL
  const gist = await publicOctokit.gists.create({
    description: `GPFA Document: ${file.name} — ${applicantName}`,
    public: false,
    files: {
      [file.name]: { content: `data:${file.type};base64,${base64}` },
    },
  });

  return gist.data.html_url ?? "";
}

// ─── BOOTSTRAP: Create required labels in repo ──────────────────────────────
export async function bootstrapRepoLabels() {
  const allLabels = [
    ...Object.values(STATUS_LABELS),
    ...COUNTY_LABELS,
    { name: "scholarship-application", color: "0B2545" },
  ];
  
  for (const label of allLabels) {
    try {
      await adminOctokit.issues.createLabel({
        owner: OWNER, repo: REPO, ...label,
      });
    } catch {
      // Label already exists — skip
    }
  }
}
```

---

### STEP 4 — Server Actions (`src/app/actions.ts`)

```typescript
"use server";
import { submitApplication, updateApplicationStatus, fetchApplications, bootstrapRepoLabels } from "@/lib/github";
import type { ApplicationData } from "@/lib/github";

export async function submitApplicationAction(data: ApplicationData) {
  return await submitApplication(data);
}

export async function updateStatusAction(issueNumber: number, status: "pending" | "review" | "approved" | "rejected") {
  return await updateApplicationStatus(issueNumber, status);
}

export async function fetchApplicationsAction(filters?: { status?: string; county?: string }) {
  return await fetchApplications(filters);
}

export async function bootstrapAction() {
  return await bootstrapRepoLabels();
}
```

---

### STEP 5 — GitHub Actions workflow (`.github/workflows/notify.yml`)

```yaml
name: GPFA Application Notifications

on:
  issues:
    types: [labeled]

jobs:
  notify-status-change:
    runs-on: ubuntu-latest
    if: startsWith(github.event.label.name, 'status:')

    steps:
      - name: Extract applicant email from issue body
        id: extract
        run: |
          EMAIL=$(echo "${{ github.event.issue.body }}" | grep -oP '(?<=\| Email \| )[^\s|]+')
          echo "email=$EMAIL" >> $GITHUB_OUTPUT
          STATUS=$(echo "${{ github.event.label.name }}" | sed 's/status: //')
          echo "status=$STATUS" >> $GITHUB_OUTPUT
          REF=$(echo "${{ github.event.issue.title }}" | grep -oP '\[GPFA-[^\]]+\]' | tr -d '[]')
          echo "ref=$REF" >> $GITHUB_OUTPUT

      - name: Send notification email
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: smtp.gmail.com
          server_port: 587
          username: ${{ secrets.SMTP_USERNAME }}
          password: ${{ secrets.SMTP_PASSWORD }}
          to: ${{ steps.extract.outputs.email }}
          from: "GPFA Scholarships <scholarships@gboweefoundation.org>"
          subject: "GPFA Scholarship Application Update — ${{ steps.extract.outputs.ref }}"
          body: |
            Dear Applicant,

            Your GPFA Scholarship application (${{ steps.extract.outputs.ref }}) 
            status has been updated to: ${{ steps.extract.outputs.status }}.

            If approved, our team will contact you within 5 business days 
            with next steps.

            With peace and hope,
            Gbowee Peace Foundation Africa
            Monrovia, Liberia
          html_body: |
            <div style="font-family:Georgia,serif;max-width:600px;margin:0 auto;padding:2rem">
              <div style="background:#0B2545;padding:1.5rem;text-align:center;color:#fff;border-radius:8px 8px 0 0">
                <h1 style="margin:0;font-size:1.4rem">Gbowee Peace Foundation Africa</h1>
                <p style="margin:.5rem 0 0;opacity:.8;font-size:13px">Scholarship Portal</p>
              </div>
              <div style="padding:2rem;border:1px solid #eee;border-top:none;border-radius:0 0 8px 8px">
                <p>Dear Applicant,</p>
                <p>Your GPFA Scholarship application <strong>${{ steps.extract.outputs.ref }}</strong> 
                   has been updated to:</p>
                <div style="background:#f4f6f8;border-left:4px solid #0B2545;padding:1rem;margin:1rem 0;font-size:1.2rem;font-weight:bold">
                  ${{ steps.extract.outputs.status }}
                </div>
                <p>If approved, our team will contact you within 5 business days with next steps.</p>
                <p>With peace and hope,<br><strong>Gbowee Peace Foundation Africa</strong><br>Monrovia, Liberia</p>
              </div>
            </div>

  notify-new-submission:
    runs-on: ubuntu-latest
    if: github.event.action == 'opened' && contains(github.event.issue.labels.*.name, 'scholarship-application')

    steps:
      - name: Notify admin of new application
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: smtp.gmail.com
          server_port: 587
          username: ${{ secrets.SMTP_USERNAME }}
          password: ${{ secrets.SMTP_PASSWORD }}
          to: ${{ secrets.ADMIN_EMAIL }}
          from: "GPFA Portal <scholarships@gboweefoundation.org>"
          subject: "New Scholarship Application — ${{ github.event.issue.title }}"
          body: |
            A new scholarship application has been submitted.
            
            View it here: ${{ github.event.issue.html_url }}
```

---

### STEP 6 — GitHub Pages deployment (`.github/workflows/deploy.yml`)

```yaml
name: Deploy GPFA Portal to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - name: Build Next.js (static export)
        run: npm run build
        env:
          NEXT_PUBLIC_GITHUB_OWNER: ${{ secrets.NEXT_PUBLIC_GITHUB_OWNER }}
          NEXT_PUBLIC_GITHUB_REPO: ${{ secrets.NEXT_PUBLIC_GITHUB_REPO }}
          NEXT_PUBLIC_GITHUB_PUBLIC_TOKEN: ${{ secrets.NEXT_PUBLIC_GITHUB_PUBLIC_TOKEN }}
          NEXT_PUBLIC_APP_URL: ${{ steps.deployment.outputs.page_url }}

      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./out

      - id: deployment
        uses: actions/deploy-pages@v4
```

Add to `next.config.ts`:
```typescript
const nextConfig = {
  output: "export",
  trailingSlash: true,
  images: { unoptimized: true },
};
export default nextConfig;
```

---

### STEP 7 — Full 6-step application form

Build the full multi-step form in `src/app/apply/page.tsx` with these exact specs:

**Step 1 — Personal Information**
Fields: surname, firstName, middleName, gender (radio), citizenship (radio: Liberian/Other), county (select with all 15 Liberian counties: Bomi, Bong, Gbarpolu, Grand Bassa, Grand Cape Mount, Grand Gedeh, Grand Kru, Lofa, Margibi, Maryland, Montserrado, Nimba, River Cess, River Gee, Sinoe), address, gradeLevel, phone (with +231 prefix input), email, dob, prevApplied (radio), desiredMajor, heardVia (radio: Radio, Friend/Family, School/Teacher, Social Media, Church/Mosque, Community Leader, Other), docChecklist (checkboxes: Birth Certificate, Liberian Passport/National ID, School Transcripts, WASSCE/WAEC Results, Reference Letters (2), Personal Essay, Financial Statement, Medical Certificate)

**Step 2 — Family & Guardian Information**
Fields: motherName, motherStatus (radio: Alive/Deceased), motherOccupation, motherEmployer, motherPhone (+231), motherEmail, motherCounty; fatherName, fatherStatus (radio), fatherOccupation, fatherEmployer, fatherPhone (+231), fatherEmail, fatherCounty; sponsorName, sponsorRelation, sponsorPhone (+231), sponsorOccupation; relativeGpfa (radio + text), dismissalHistory (radio + textarea)

**Step 3 — Academic Information**
Dynamic school history table (4 rows: Primary, Junior High, Senior High, University) with columns: schoolName, location, yearsAttended, certificate, gpa. Fields: honors (textarea), activities (textarea), countriesVisited, currentGpa. Work experience table (2 rows): employer, jobTitle, dates, location.

**Step 4 — Additional Info & Documents**
Field: universitiesApplied (textarea). Four react-dropzone upload zones: Semester Billing, Grade Sheets/Progress Reports, WASSCE/WAEC Results & Certificates, Official Transcripts. Each upload calls uploadDocumentAsGist() and stores returned URLs in form state. Show file previews with remove buttons.

**Step 5 — Financial Aid Application**
Fields: fatherIncome, motherIncome, sponsorIncome, otherIncome, totalIncome (auto-sum), dependents. Housing select: Rented/Own home/Living with relatives/Government housing. monthlyRent, vehicles (text), land (text), debts (textarea), monthlyExpenses. tuitionCost, efc, amountRequested, otherScholarships. letterOfNeed (textarea, min 150 words). Upload zone for financial documents.

**Step 6 — Declaration & Signature**
Declaration text block (render the GPFA standard declaration paragraph). react-signature-canvas for applicant signature (converts to base64 PNG). signDate (date input, default today). applicantPrintedName. Second signature canvas for parent/guardian. guardianName, guardianRelation. Privacy notice box.

**Progress bar** at top: 6 numbered dots with labels, connector line, gold fill for completed steps, navy for active, gray for future.

**Auto-save to localStorage** on every field change (debounced 800ms). "Resume" toast on page load if draft found.

**Form validation** with Zod: step 1 requires surname, firstName, county, address, phone, dob. Step 6 requires applicantPrintedName, signDate.

On submit: call `submitApplicationAction()`, redirect to `/success?ref=GPFA-XXXX&issue=NNN`.

---

### STEP 8 — Admin dashboard (`src/app/admin/page.tsx`)

Build a full admin dashboard protected by a hardcoded admin password (env var `ADMIN_PASSWORD`) stored in localStorage after login.

Features:
- Login screen with GPFA branding
- Stats cards: Total Applications, Pending, Approved, Aid Committed (parsed from issue bodies)
- Applications table: columns = Ref#, Name, County, Major, Status pill, Date, Actions
- Filter bar: text search (searches issue titles), status dropdown, county dropdown
- Each row has "View" button → opens slide-over drawer with full parsed issue body displayed in clean sections (Personal, Family, Academic, Financial, Documents as clickable Gist links)
- Status buttons in drawer: Pending / Under Review / Approve / Reject — each calls `updateStatusAction()` with confirmation dialog
- Export CSV button: generates CSV from all fetched issues and triggers download
- On mount: call `fetchApplicationsAction()`, parse issue bodies into structured objects
- Sidebar navigation: Applications, Review Queue (filter to `status: under-review`), Statistics (county bar chart from label counts), Settings

---

### STEP 9 — Homepage (`src/app/page.tsx`)

Build a stunning homepage:
- Sticky nav: GPFA logo (⚖️ icon + "Gbowee Peace Foundation Africa" in Playfair Display), links: Home, Apply, Admin
- Hero section: dark navy (#0B2545) background, subtle dot-grid texture via SVG background-image, gold badge "Empowering Liberian Students 🇱🇷", headline "Building a Brighter Future Through Education", subtext, two CTAs: "Apply for Scholarship" (crimson) + "Learn More" (ghost)
- Stats bar: 1,200+ Students, 15 Counties, $2.4M Aid, 94% Graduation Rate
- Scholarship programs grid (4 cards): University Scholarship, Leadership Award, Financial Aid, WASSCE Support
- About section (dark navy): Leymah Gbowee bio + Nobel Peace Prize quote
- How to Apply steps (3 cards)
- Full-width crimson CTA banner
- Footer: address, phone, email

Use Google Fonts: `Playfair Display` (headings) + `DM Sans` (body).
Color palette: `--navy: #0B2545`, `--crimson: #BF0A2A`, `--gold: #D4A017`, `--sage: #3D6B4F`, `--cream: #FAF7F0`.

---

### STEP 10 — Success page (`src/app/success/page.tsx`)

Read `?ref=` and `?issue=` from URL params. Show:
- ✅ Green checkmark icon
- "Application Submitted!" heading
- Reference number (large, bold, navy)
- "A confirmation email has been sent. Review takes 8–12 weeks."
- Link to view the GitHub Issue (opens in new tab)
- "Return to Home" button

---

### STEP 11 — Bootstrap script (`scripts/bootstrap-github.ts`)

```typescript
// Run once: npx ts-node scripts/bootstrap-github.ts
import { bootstrapRepoLabels } from "../src/lib/github";

async function main() {
  console.log("Creating GitHub labels for GPFA repo...");
  await bootstrapRepoLabels();
  console.log("✅ Done! Labels created in your GitHub repo.");
}
main().catch(console.error);
```

Add to `package.json`:
```json
"scripts": {
  "bootstrap": "ts-node scripts/bootstrap-github.ts",
  "dev": "next dev",
  "build": "next build",
  "start": "next start"
}
```

---

### STEP 12 — README.md

Write a comprehensive README:

```markdown
# GPFA Scholarship Portal

> Powered by Next.js + GitHub as a backend.

## Architecture

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 15 (App Router) + Tailwind CSS |
| Storage | GitHub Issues (one per application) |
| Status | GitHub Labels |
| Documents | GitHub Gists (private, base64) |
| Auth | GitHub PAT (public write-issues scope) |
| Email | GitHub Actions + SMTP |
| Hosting | GitHub Pages (static export) |

## Setup

### 1. Create GitHub repo
Create a NEW private GitHub repo named `scholarship-applications` in your org.

### 2. Create Personal Access Tokens
**Public PAT** (for applicants) — scope: `public_repo` + `gist`
**Admin PAT** (for admins only) — scope: `repo` + `admin:repo_hook`

### 3. Add repo secrets (Settings → Secrets → Actions)
- `NEXT_PUBLIC_GITHUB_OWNER`
- `NEXT_PUBLIC_GITHUB_REPO`
- `NEXT_PUBLIC_GITHUB_PUBLIC_TOKEN`
- `GITHUB_ADMIN_TOKEN`
- `ADMIN_PASSWORD` (for admin dashboard login)
- `SMTP_USERNAME` + `SMTP_PASSWORD` (Gmail app password)
- `ADMIN_EMAIL`

### 4. Bootstrap labels
\`\`\`bash
cp .env.example .env.local  # fill in your tokens
npm run bootstrap           # creates all labels in your repo
\`\`\`

### 5. Enable GitHub Pages
Repo Settings → Pages → Source: GitHub Actions

### 6. Push to main
```bash
git add . && git commit -m "Initial GPFA Portal" && git push origin main
```
The GitHub Actions workflow deploys automatically.

## Local development
```bash
npm install && npm run dev
```

## Application flow
1. Student visits `https://your-org.github.io/scholarship-portal`
2. Fills 6-step form → submits → GitHub Issue created with all data as Markdown tables
3. Admin visits `/admin` → logs in → sees all Issues as a dashboard
4. Admin updates status → label changes → GitHub Action fires → email sent to student
```

---

### Final verification checklist

After building, verify:
- [ ] `npm run dev` starts without errors
- [ ] Form navigates all 6 steps with validation
- [ ] Submitting creates a real GitHub Issue in the target repo
- [ ] Issue has correct labels (status: pending + county: X)
- [ ] Issue body has all data in Markdown tables
- [ ] Admin dashboard fetches and displays Issues
- [ ] Status update changes labels on the GitHub Issue
- [ ] `npm run build` produces static export in `./out`
- [ ] GitHub Pages workflow deploys successfully
- [ ] Notification workflow triggers on label changes

---

**Build this completely. Write every file. Do not scaffold — implement fully.**
