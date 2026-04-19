# ai-agent-qa-notes
# 🤖 AI Agent QA — Notes, Patterns & Approaches

> Practical QA notes from building and testing a real multi-agent AI system.  
> This repo documents approaches, patterns and lessons learned — not theory, but things that actually work in production.

---

## About This Project

I'm a Senior QA Engineer with 17 years of experience currently designing and testing a **multi-agent AI system** built on Claude API for a 3D printing business.

The system includes:
- 🗂 **Contact Management Agent** — business card OCR via Vision API, QR parsing, Telegram bot interface
- 🧪 **Chemical Research Agent** — web scraping of resin manufacturers, MSDS/TDS PDF analysis, CAS registry
- 📄 **Accounting Agent** — order processing, invoice generation, document automation

**Stack:** Claude API · Telegram Bot · Supabase (PostgreSQL) · Next.js · Playwright · Python

This repo captures QA-specific knowledge I'm building along the way.

---

## 📋 Contents

### 1. Testing AI Agents — Core Challenges

| Challenge | Why It's Hard | Approach |
|-----------|--------------|---------|
| Non-deterministic output | Same input → different output each time | Golden set + semantic comparison |
| Hallucinations | Model invents data that looks real | Confidence scoring + mandatory human verification |
| Tool call validation | Agent calls wrong tool or wrong params | Contract testing for each tool |
| Context drift | Long conversations lose context | Session boundary tests |
| Cascading errors | Error in agent A breaks agent B | Isolated agent testing + integration layer |

---

### 2. Confidence Score Pattern

One of the key patterns for protecting against LLM hallucinations — used in the contact management agent:

```python
# Agent returns confidence score alongside extracted data
{
  "first_name": "David",
  "last_name": "Hamacher", 
  "company": "Jessberger GmbH",
  "confidence": {
    "first_name": 0.97,
    "last_name": 0.95,
    "company": 0.92,
    "phone": 0.61,   # ← flagged for human review
    "email": 0.88
  }
}

# QA validation rules:
# >= 0.90 → display normally ✅
# 0.70–0.89 → display with ⚠️ warning, user must confirm
# < 0.70 → field left empty, user fills manually
```

**Why this matters for QA:**
- You can't test LLM output like deterministic code
- Instead, test the *confidence thresholds* and *downstream behavior*
- Test that low-confidence fields are actually flagged in the UI
- Test that users cannot bypass verification for flagged fields

---

### 3. Test Categories for AI Agents

#### 3.1 Input Validation Tests
```
✓ Clear, high-quality business card photo → correct extraction
✓ Blurry / rotated photo → graceful degradation, not crash
✓ Photo with no text → handled without error
✓ Non-business card photo → agent asks for clarification
✓ QR code (vCard format) → correct parsing
✓ QR code (meCard format) → correct parsing  
✓ Invalid QR → user notified, fallback offered
```

#### 3.2 Hallucination Detection Tests
```
✓ Company name on card = "Jessberger" → agent must not autocomplete to "Jessberger AG" if not on card
✓ Phone number partially visible → agent must flag, not guess
✓ Email not present → field empty, not invented
✓ Two people on one card → agent asks which contact to save
```

#### 3.3 Tool Selection Tests (Orchestrator)
```
✓ Photo sent → orchestrator calls contacts_agent (not filaments_agent)
✓ URL to formlabs.com sent → calls filaments_agent (not contacts_agent)
✓ "Invoice for Ivan on PLA 1kg" → calls accounting_agent
✓ Ambiguous message → orchestrator asks for clarification, does not guess
✓ Unknown input type → safe fallback, no crash
```

#### 3.4 Data Persistence Tests
```
✓ Contact saved → appears in Supabase with correct fields
✓ Confidence score stored → audit trail complete
✓ User correction logged → delta between AI output and human edit captured
✓ Duplicate detection → same contact from same event flagged
```

#### 3.5 Conversation Context Tests
```
✓ Multi-step flow completes correctly (photo → categories → event → save)
✓ User presses "Edit" mid-flow → correct field targeted
✓ User abandons flow → no partial record saved
✓ User sends new photo mid-flow → previous context cleared, new flow starts
```

---

### 4. Web Scraping Agent QA (Filaments Module)

Special challenges when the agent browses the web:

```
Test scenarios:
✓ Valid manufacturer URL → products found and parsed
✓ MSDS PDF link → section 3 extracted correctly
✓ CAS number extracted → verified against PubChem API
✓ CAS number not found in PubChem → flagged, not saved silently
✓ Percentages sum > 100% → validation error raised
✓ Site blocks scraper → graceful failure, user notified
✓ PDF password-protected → user asked to upload manually
✓ Conflicting data between MSDS versions → both versions flagged
```

**The PubChem verification pattern:**
```python
# Every CAS number extracted by AI is verified externally
def verify_cas(cas_number: str) -> dict:
    url = f"https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/name/{cas_number}/JSON"
    response = requests.get(url)
    if response.status_code == 200:
        return {"verified": True, "name": extract_name(response.json())}
    return {"verified": False, "cas": cas_number}

# QA tests:
# - Real CAS → verified = True, name matches
# - Invented CAS → verified = False, record flagged
# - Network failure → error handled, not silently passed
```

---

### 5. Regression Testing Strategy for AI Systems

Traditional regression: re-run same inputs, expect same outputs.  
AI regression: same inputs, outputs *will* differ — but quality must not drop.

**Approach I use:**

```
1. Golden Set — curated test cases with known-good outputs
   - 20 business card photos with manually verified extractions
   - 10 MSDS PDFs with manually verified compositions
   - Expected: confidence >= 0.85 on key fields

2. Semantic Diff — not exact match, but meaning match
   - "Dr. Jessberger GmbH" ≈ "Jessberger GmbH" → PASS
   - "Jessberger AG" ≠ "Jessberger GmbH" → FAIL

3. Regression Trigger — when to re-run golden set
   - After Claude model version update
   - After prompt changes
   - After any change to extraction logic

4. Metrics tracked over time
   - Average confidence score per field
   - % of fields requiring human correction
   - False positive rate (data that looked right but wasn't)
```

---

### 6. Tools & Setup

```bash
# Core testing tools used in this project
python-telegram-bot  # Telegram bot integration testing
pytest               # Test runner
httpx                # API testing (async)
playwright           # Browser automation for scraping tests
supabase-py          # Database state verification
```

---

## 🗺 Roadmap

- [x] Contact agent QA patterns documented
- [x] Confidence scoring pattern
- [x] Web scraping agent test scenarios
- [ ] Python test suite — contact extraction (in progress)
- [ ] Pytest fixtures for Supabase test isolation
- [ ] CI/CD pipeline with golden set regression
- [ ] LLM output comparison utilities

---

## About Me

**Serhii Karabuta** — Senior QA Engineer  
17 years in QA: web · mobile · desktop · API · hardware · medical devices

Currently expanding into: **AI Agent Testing · Python Automation · LLM QA**

📧 serhii.karabuta@gmail.com  
💼 [LinkedIn](https://linkedin.com/in/serhii-karabuta-629a41102)  
🌍 Kharkiv, Ukraine · Open to remote worldwide

---

*This repo is actively updated as the project evolves.*  
*Last updated: April 2026*
