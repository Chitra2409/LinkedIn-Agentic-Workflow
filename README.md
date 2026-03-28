# LinkedIn Agentic Workflow — Complete Build Guide

## Overview

This document describes a fully automated LinkedIn outreach and cold-emailing system built on **n8n**. Given a job description, the workflow automatically finds relevant LinkedIn profiles, extracts contact information, discovers email addresses, generates personalized cold emails, saves results to Google Sheets, and creates Gmail drafts.

---

## Architecture

```
Chat Trigger → OpenAI (Boolean Query) → SerpAPI (Google Search) → Code (Parse Results)
  → OpenAI (Extract Names/Domains) → Code (Clean Data) → Hunter.io (Find Emails)
  → Code (Merge Data) → OpenAI (Write Cold Email) → Google Sheets → Gmail Draft
```

**Total Nodes: 11**

---

## Prerequisites

| Service | Purpose | Free Tier |
|---------|---------|-----------|
| n8n Cloud | Workflow automation platform | 14-day free trial |
| OpenAI API | AI text generation (GPT-4o-mini) | Pay-as-you-go (~$0.15/1M tokens) |
| SerpAPI | Google search without getting blocked | 100 searches/month free |
| Hunter.io | Email address finder | 25 searches/month free |
| Google Account | Sheets + Gmail access | Free |

---

## Credentials Required in n8n

1. **OpenAI** — API key from platform.openai.com
2. **Google Sheets OAuth2** — Google account OAuth connection
3. **Gmail OAuth2** — Google account OAuth connection (separate from Sheets)
4. **Hunter API** — API key from hunter.io
5. **Header Auth** (legacy, no longer needed with SerpAPI)

---

## Node-by-Node Documentation

### Node 1: When Chat Message Received (Trigger)

**Type:** `@n8n/n8n-nodes-langchain.chatTrigger`
**Purpose:** Entry point for the workflow. Receives a job description via chat.
**Configuration:** Default settings, no changes needed.
**Output:** `{ chatInput: "the job description text" }`

---

### Node 2: Message a Model (Boolean Search Generator)

**Type:** `@n8n/n8n-nodes-langchain.openAi`
**Purpose:** Converts a job description into a Google Boolean search string optimized for finding LinkedIn profiles.
**Credential:** OpenAI account
**Model:** GPT-4O-MINI

**System Message:**
```
You are a recruitment search expert. Given a job description, generate a Google Boolean search string to find relevant LinkedIn profiles. The format should be: site:linkedin.com/in/ AND ("keyword1" OR "keyword2") AND ("keyword3" OR "keyword4"). Focus on job titles, skills, and industries mentioned in the job description. Return ONLY the search string, nothing else.
```

**User Message (Expression):**
```
={{ $json.chatInput }}
```

**Output Example:**
```
site:linkedin.com/in/ AND ("Software Engineer" OR "Developer") AND ("Python" OR "React" OR "AWS" OR "GCP") AND ("web applications" OR "REST APIs") AND ("United States")
```

---

### Node 3: HTTP Request (SerpAPI Google Search)

**Type:** `n8n-nodes-base.httpRequest`
**Purpose:** Performs a Google search using SerpAPI to find LinkedIn profiles matching the boolean query. SerpAPI is used instead of direct Google scraping to avoid captcha/blocking issues.

**Method:** GET

**URL (Expression):**
```
=https://serpapi.com/search.json?q={{ encodeURIComponent($json.output[0].content[0].text) }}&engine=google&api_key=YOUR_SERPAPI_KEY&num=10
```

> **IMPORTANT:** Replace `YOUR_SERPAPI_KEY` with your actual SerpAPI key.

**Authentication:** None (API key is in URL)
**Headers:** None needed
**Response Format:** JSON (set in Options → Response → Response Format)

**Output:** Full SerpAPI JSON response including `organic_results` array with LinkedIn profile links, titles, snippets, and rich snippets containing company info.

---

### Node 4: Code in JavaScript (Parse SerpAPI Results)

**Type:** `n8n-nodes-base.code`
**Purpose:** Extracts LinkedIn profile URLs, context text, and company names from SerpAPI's structured JSON response.

**Code:**
```javascript
const response = $input.first().json;
const organicResults = response.organic_results || [];

const results = [];
const seen = new Set();

for (const item of organicResults) {
  const link = item.link || '';
  if (link.includes('linkedin.com/in/') && !seen.has(link)) {
    seen.add(link);
    const extensions = (item.rich_snippet && item.rich_snippet.top && item.rich_snippet.top.extensions) || [];
    const company = extensions[2] || extensions[1] || '';
    results.push({
      json: {
        linkedinUrl: link,
        context: (item.title || '') + ' ' + (item.snippet || ''),
        company: company
      }
    });
  }
}

return results;
```

**Output per item:**
```json
{
  "linkedinUrl": "https://www.linkedin.com/in/username",
  "context": "Name - Title | Skills | Experience ...",
  "company": "Company Name"
}
```

---

### Node 5: Message a Model 1 (Extract Names & Domains)

**Type:** `@n8n/n8n-nodes-langchain.openAi`
**Purpose:** Uses AI to extract structured data (first name, last name, company domain) from the LinkedIn profile context and company name.
**Credential:** OpenAI account
**Model:** GPT-4O-MINI

**System Message:**
```
Given a LinkedIn profile URL and surrounding context from Google search results, extract the person's first name, last name, and company domain (e.g. google.com, amazon.com). Use the context to determine the company. Return ONLY valid JSON in this format: {"firstName": "", "lastName": "", "companyDomain": "", "context": ""}. If you cannot determine a field, leave it empty.
IMPORTANT: Return ONLY raw JSON. No markdown, no backticks, no code blocks.
Also extract the company domain from the context. For example, if someone works at Wells Fargo, the domain is wellsfargo.com. If at Amazon, it's amazon.com. If at Capgemini, it's capgemini.com.
The company name is provided. Convert it to a domain. For example: Walmart = walmart.com, Wells Fargo = wellsfargo.com, Amazon = amazon.com, Optum = optum.com, Eightfold AI = eightfold.ai, Social Security Administration = ssa.gov. If unsure, leave companyDomain empty.
```

**User Message (Expression):**
```
=LinkedIn URL: {{ $json.linkedinUrl }}
Company: {{ $json.company }}
Context from search results: {{ $json.context }}
```

**Output Example:**
```json
{"firstName": "Suryansh", "lastName": "Patel", "companyDomain": "wellsfargo.com", "context": "Software Developer | Java, Spring Boot, React"}
```

---

### Node 6: Code in JavaScript 1 (Clean & Validate Data)

**Type:** `n8n-nodes-base.code`
**Purpose:** Cleans OpenAI output (removes markdown formatting), validates fields, and filters out profiles with insufficient data (single-letter last names, missing domains).

**Code:**
```javascript
const items = $input.all();
const results = [];

for (let i = 0; i < items.length; i++) {
  try {
    let text = items[i].json.output[0].content[0].text;
    text = text.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
    const parsed = JSON.parse(text);
    
    const lastName = (parsed.lastName || '').replace(/\./g, '').trim();
    const firstName = (parsed.firstName || '').trim();
    const domain = (parsed.companyDomain || '').trim();
    
    if (firstName.length >= 2 && lastName.length >= 2 && domain.length >= 3) {
      results.push({
        json: {
          firstName: firstName,
          lastName: lastName,
          companyDomain: domain,
          context: parsed.context || '',
          linkedinUrl: ''
        }
      });
    }
  } catch (e) {}
}

return results;
```

**Validation Rules:**
- First name must be 2+ characters
- Last name must be 2+ characters (filters out "K", "R." etc.)
- Company domain must be 3+ characters
- Removes markdown code block formatting from OpenAI responses
- Silently skips unparseable items

---

### Node 7: Hunter (Find Email Addresses)

**Type:** `n8n-nodes-base.hunter`
**Purpose:** Uses Hunter.io's Email Finder API to discover email addresses based on first name, last name, and company domain.
**Credential:** Hunter API account
**Operation:** Email Finder

**Settings:**
- **On Error:** Continue Using Error Output (CRITICAL — prevents workflow from stopping when Hunter can't find an email)

**Field Mappings (Expression):**
- **Domain:** `={{ $json.companyDomain }}`
- **First Name:** `={{ $json.firstName }}`
- **Last Name:** `={{ $json.lastName }}`

**Output on success:** `{ data: { email: "found@email.com", first_name: "...", last_name: "...", domain: "..." } }`
**Output on error:** Error object with original input data

---

### Node 8: Code in JavaScript 2 (Merge Hunter Results)

**Type:** `n8n-nodes-base.code`
**Purpose:** Normalizes the output from Hunter regardless of whether it succeeded or failed. Ensures downstream nodes always receive consistent field names.

**Code:**
```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  const json = item.json;
  
  const firstName = json.firstName || (json.data && json.data.first_name) || '';
  const lastName = json.lastName || (json.data && json.data.last_name) || '';
  const email = (json.data && json.data.email) || json.email || '';
  const domain = json.companyDomain || (json.data && json.data.domain) || '';
  const linkedinUrl = json.linkedinUrl || (json.data && json.data.linkedin_url) || '';
  const context = json.context || '';
  
  if (firstName || lastName) {
    results.push({
      json: {
        firstName: firstName,
        lastName: lastName,
        email: email,
        companyDomain: domain,
        linkedinUrl: linkedinUrl,
        context: context
      }
    });
  }
}

if (results.length === 0) {
  return [{ json: { firstName: 'Unknown', lastName: 'Contact', email: '', companyDomain: '', linkedinUrl: '', context: '' } }];
}

return results;
```

**Why this node exists:** When Hunter succeeds, it outputs data at `json.data.email`, `json.data.first_name`, etc. When Hunter fails (error output), the original input fields (`json.firstName`, `json.lastName`) are passed through instead. This node unifies both cases into a single consistent format.

---

### Node 9: Message a Model 2 (Cold Email Generator)

**Type:** `@n8n/n8n-nodes-langchain.openAi`
**Purpose:** Generates a personalized cold email for each contact based on their name, email, company, and professional context.
**Credential:** OpenAI account
**Model:** GPT-4O-MINI

**System Message:**
```
You are an expert cold email writer. Given a person's name, email, LinkedIn URL, and context, write a short, personalized cold email. The email should be professional but warm, mention something specific about their background, and end with a clear call to action. Keep it under 150 words. Return the email with a subject line in this format:
Subject: [subject line]
Body: [email body]
```

**User Message (Expression):**
```
=Name: {{ $json.firstName }} {{ $json.lastName }}
Email: {{ $json.email }}
Company: {{ $json.companyDomain }}
Context: {{ $json.context }}
```

---

### Node 10: Append Row in Sheet (Save to Google Sheets)

**Type:** `n8n-nodes-base.googleSheets`
**Purpose:** Saves all extracted data and the generated cold email to a Google Sheet for tracking and review.
**Credential:** Google Sheets OAuth2 account
**Operation:** Append Row

**Document:** LinkedIn Outreach Results (connected via URL)
**Sheet:** Sheet1

**Google Sheet Column Headers (Row 1):**
| A | B | C | D | E |
|---|---|---|---|---|
| Name | Email | LinkedIn URL | Company Domain | Cold Email |

**Column Mappings (Expression):**
- **Name:** `={{ $json.firstName }} {{ $json.lastName }}`
- **Email:** `={{ $json.email }}`
- **LinkedIn URL:** `={{ $json.linkedinUrl }}`
- **Company Domain:** `={{ $json.companyDomain }}`
- **Cold Email:** `={{ $json.output[0].content[0].text }}`

---

### Node 11: Create a Draft (Gmail)

**Type:** `n8n-nodes-base.gmail`
**Purpose:** Creates a draft email in Gmail for each contact, ready for manual review and sending.
**Credential:** Gmail OAuth2 account
**Resource:** Draft
**Operation:** Create
**Settings:** On Error → Continue Using Error Output

**Field Mappings (Expression):**
- **To:** `={{ $json.Email }}`
- **Subject:** `={{ $json['Cold Email'].split("Subject: ")[1].split("\n")[0] }}`
- **Message:** `={{ $json['Cold Email'].split("Body: ")[1] }}`

> **Note:** Gmail receives data from Google Sheets, so field names use the Sheet column headers ("Email", "Cold Email") not the original field names.

---

## Data Flow Summary

```
Step 1:  User types job description
         ↓
Step 2:  OpenAI generates Boolean search string
         ↓ output[0].content[0].text = "site:linkedin.com/in/ AND ..."
Step 3:  SerpAPI searches Google
         ↓ organic_results = [{link, title, snippet, rich_snippet}]
Step 4:  Code parses results
         ↓ {linkedinUrl, context, company} × 10 items
Step 5:  OpenAI extracts names/domains
         ↓ output[0].content[0].text = '{"firstName":"...", "lastName":"...", "companyDomain":"..."}'
Step 6:  Code cleans & validates
         ↓ {firstName, lastName, companyDomain, context} (filtered)
Step 7:  Hunter finds emails
         ↓ {data: {email, first_name, last_name, domain}} or error
Step 8:  Code merges data
         ↓ {firstName, lastName, email, companyDomain, linkedinUrl, context}
Step 9:  OpenAI writes cold email
         ↓ output[0].content[0].text = "Subject: ...\nBody: ..."
Step 10: Google Sheets saves row
         ↓ {Name, Email, LinkedIn URL, Company Domain, Cold Email}
Step 11: Gmail creates draft
         ↓ Draft email ready for review
```

---

## Key Design Decisions

### Why SerpAPI instead of direct Google scraping?
The original workflow from Abhijay's guide uses direct HTTP requests to Google with browser cookies for authentication. This works when n8n is self-hosted on the same machine as the browser (matching IP addresses). On n8n Cloud, Google's servers see a different IP than where the cookies were created, triggering captcha/security challenges. SerpAPI handles all Google authentication internally and never gets blocked.

### Why a "Merge" Code node after Hunter?
Hunter's output format differs between success and failure cases. On success, data lives at `json.data.email`. On failure (error output), the original input fields pass through differently. The merge node normalizes both cases so downstream nodes always see the same field names.

### Why "Continue On Error" on Hunter?
Hunter can't find emails for everyone — missing domains, uncommon names, or private profiles all cause failures. Without "Continue On Error", one failed lookup stops the entire workflow. With it enabled, failed lookups are skipped and the workflow continues with the next profile.

### Why GPT-4O-MINI?
It's the cheapest OpenAI model that reliably follows structured output instructions (JSON formatting, boolean search syntax). GPT-4o would work better but costs significantly more for this use case.

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| SerpAPI returns no results | Bad boolean query | Check OpenAI system prompt; try simpler queries |
| Hunter errors on all profiles | Missing company domains | Improve OpenAI prompt for domain extraction; add domain mapping in Code node |
| Hunter "invalid_last_name" | Single letter last names like "K" | Code in JavaScript1 filters lastName < 2 chars |
| OpenAI returns markdown code blocks | Model wraps JSON in \`\`\`json | Code in JavaScript1 strips markdown formatting |
| Cold emails are generic | Empty name/email fields reaching OpenAI | Check merge Code node; ensure Hunter data flows correctly |
| Gmail "invalid email" | No email found by Hunter | Hunter "Continue On Error" skips these; Gmail also has error handling |
| Workflow stops unexpectedly | A node fails without error handling | Set "On Error" → "Continue Using Error Output" on Hunter and Gmail nodes |

---

## Cost Estimate (per run)

| Service | Usage | Cost |
|---------|-------|------|
| OpenAI (GPT-4o-mini) | ~3 calls × 10 profiles = ~30 calls | ~$0.01–0.05 |
| SerpAPI | 1 search | Free (100/month) |
| Hunter.io | Up to 10 lookups | Free (25/month) |
| n8n Cloud | 1 workflow execution | Free trial / included in plan |
| **Total per run** | | **~$0.01–0.05** |

---

## How to Run

1. Open the workflow in n8n editor
2. Click **"Chat"** button at the bottom of the canvas
3. Type a job description, e.g.:
   ```
   Software Engineer with 3-5 years experience in Python, React, and AWS. Based in the United States.
   ```
4. Press Send and watch the workflow execute
5. Check your **Google Sheet** for results
6. Check your **Gmail Drafts** for personalized emails

---

## Future Improvements

- Add a loop node to process more than 10 results (paginate SerpAPI)
- Add a "sender background" field to the cold email prompt for more personalized outreach
- Use LinkedIn's rich_snippet company data to improve domain extraction
- Add deduplication across multiple runs
- Connect to a CRM (HubSpot, Salesforce) instead of Google Sheets