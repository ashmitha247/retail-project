# retail-project
# 🛡️ VendorLadon

**Pre‑Shipment EDI Validator for Walmart India**  
(Proof-of-Concept Hackathon Prototype — not production‑ready)

---

## 🎯 What It Solves

Vendors sending shipments to Walmart India must submit an **Advance Shipment Notice (ASN)** using the **EDI X12‑856** format. Common issues include:

- ❌ Invalid **GSTIN** or incorrect state code  
- ❌ Missing or mis‑placed **PO number** (must be in `PRF` segment)  
- ❌ Unregistered product IDs (GTIN‑14 / Walmart Item Number)  
- ❌ ASN sent too early or too late (OTP/ITF rule: exactly 24 hrs before ship)  
- ❌ Expired or missing **AS2** certificate

These errors often result in warehouse rejections, OTIF penalties, and long disputes via Retail Link. VendorGuard catches them **before** submission, using a rule‑based, real‑time validator.

---

## 🧰 Tech Stack

| Component              | Tool/Library                        | Purpose |
|------------------------|-------------------------------------|---------|
| EDI Parsing            | `spaceavocado‑x12`, `pyx12`, or `badX12` 1 | Parses X12 856 EDI files with nested loops & proper delimiters |
| Product GTIN Lookup    | Local CSV + GS1 / GDSN API (future) | Validates 14‑digit GTINs |
| Walmart Item Lookup    | `requests` → Retail Link / Walmart API | Validates Item Numbers (WIN) |
| GSTIN Validation       | `gstin‑validator` or regex + checksum 2 | Checks format, state prefix, and checksum digit |
| ASN Timing Check       | Python `datetime`                   | Enforces 24‑hr before, never after |
| AS2 Cert Validation    | OpenSSL CLI via `subprocess`        | Ensures cert trusted by Walmart root |
| UI / Demo App          | Streamlit                           | Interactive file upload & report UI |
| Data Handling          | Pandas                              | CSV or JSON data handling |

---

## 🧩 How the Code Works (Overview)

1. **Upload ASN (EDI 856)** via Streamlit  
2. **Parse** using robust X12 parser → builds internal tree structure  
3. **Validate** in sequence:
   - Check for required segments: `ST`, `BSN`, `HL`, `PRF`, `TD5`, `SE`
   - Extract `PO number` from `PRF`
   - Validate **GTIN-14** with GS1 logic (currently CSV-based)
   - Lookup **Walmart Item Number** (WIN) via API
   - Validate **GSTIN** format, state code, checksum digit
   - Confirm **ASN Timing** is within 0–24h of shipping
   - Verify **AS2 certificate** against Walmart’s root store
4. **Collect** errors in list, return a Retail‑Link‑style JSON  
5. In Streamlit, show ✅ or ❌ per rule + detailed JSON report

---

## 🔄 Simulation vs Production

| Feature | Simulation (Now) | Production (Real-World) |
|--------|-----------------|-------------------------|
| EDI Parsing | String split or basic parser | Full X12 parser handling ISA/GS/ST, nested loops 3 |
| GTIN/GS1 Checks | Static CSV lookup | Real-time GTIN validation via GS1 GDSN API |
| Win Validation | Local CSV or mock | Live call to Walmart’s Item Maintenance API |
| GSTIN Validation | Format/state prefix | Adds PAN/state/city lookup + full checksum 4 |
| ASN Timing | Based on simple timestamp | Handles vendor timezones, DST edge cases |
| AS2 Validation | Checks only expiry | Validates certificate against Walmart root CA |
| Business Rules | Hard-coded per-state | Config-driven (YAML or DB) with dynamic updates |
| Scalability | Single-threaded, small files | Distributed/cloud processing, async/parallel |
| Input Sources | Local file uploads | AS2/FTP submission, DB, real-time EDI flow |

---

## 🛠️ Critical Faults to Address Before Submission

1. **Parser**: Don’t rely on `split('~')`; install and integrate `spaceavocado-x12` or `pyx12` 5  
2. **GTIN Lookup**: Plan for GS1/GDSN APIs instead of CSV  
3. **WIN Validation**: Integrate with Walmart’s API  
4. **GSTIN Check**: Use `gstin-validator` or implement full regex + checksum 6  
5. **AS2 Trust**: Use `openssl verify -CAfile walmart_root.pem vendor_cert.pem`  
6. **Rule Config**: Move from hardcoded `if state=='MH'` to YAML/DB  
7. **Testing**: Add pytest with ITS edge cases (missing SE, timezone offsets)  
8. **Performance**: Evaluate Dask / async I/O for large EDI documents

---


🧠 What is VendorLadon doing, exactly?
VendorGuard reads EDI shipment files (sent by Walmart India vendors), checks them for common mistakes (like missing PO or invalid GSTIN), and shows a simple dashboard to flag errors before submission. It does this using 100% rule-based logic.

🧰 Tech Stack (Beginner-Friendly)
Layer
Tool/Tech
Why We Use It
🐍 Backend logic
Python
Easy to write rules, parse files, and do data checks
📊 Data processing
Pandas
Handles tabular data (like CSVs or EDI segments)
🖥️ UI / Dashboard
Streamlit
Fast way to build interactive apps for file upload + results
📂 Input format
.txt, .csv, .json
Simulated EDI files or real Walmart test files
🔒 Certificate validation
OpenSSL (via Python)
Checks if AS2 certificate is present and not expired
🌐 APIs (Optional)
requests
To simulate Walmart product lookups or GS1 checks


🧩 How the Code Works (Step by Step)
Let’s break down each validation layer into how it would be implemented.

1️⃣ EDI X12 856 Structure Check
Problem:
Walmart expects EDI files to follow a structure like:
 ST → BSN → HL → PRF → TD5 → SE
Code:
required_segments = ['ST', 'BSN', 'HL', 'PRF', 'TD5', 'SE']
for seg in required_segments:
    if seg not in edi_file:
        errors.append(f"{seg} segment missing")

What this does:
It checks if all the necessary pieces (segments) are present in the file.
If the PRF is missing, it means no PO (purchase order) is given → Error!
Tool used: Python string parsing or split by ~ (since EDI files are like big strings).

2️⃣ Product Code Validation (GTIN + WIN)
Problem:
Walmart wants correct GTIN-14 codes (barcodes on cases/pallets) and internal item numbers.
Code:
if gtin not in gs1_csv["GTIN"].values:
    errors.append("GTIN not found in GS1 master list")

if win not in walmart_api_results:
    errors.append("Walmart Item Number not recognized")

What this does:
It loads a list of valid GTINs (from a CSV or simulated GS1 database).
Checks if the product in the shipment exists in that list.
Same for Walmart Item Numbers (WINs).
Tools used:
Pandas (to load GTIN master file like a spreadsheet)
Optional: Call a real/simulated Walmart product API using requests.

3️⃣ GSTIN & E-Way Bill Validation
Problem:
Indian GSTIN must:
Be 15 characters
Start with a valid state code (like 27 for Maharashtra)
E-Way Bill is needed if invoice > ₹50,000
Code:
if len(gstin) != 15 or gstin[:2] != selected_state_code:
    errors.append("GSTIN format or state mismatch")

if invoice_amount > 50000 and not has_eway_bill:
    errors.append("E-Way Bill required for invoices > ₹50K")

Tools used:
Python for simple rules
Streamlit select box lets user pick their state for validation

4️⃣ ASN Timing Validation (OTIF Rule)
Problem:
Walmart says ASN must arrive:
No more than 24 hours before shipping
Never after shipping
Code:
delta = ship_time - asn_time

if delta > timedelta(hours=24):
    errors.append("ASN submitted too early (>24 hrs)")

elif delta < timedelta(seconds=0):
    errors.append("ASN submitted after shipping")

What this does:
Calculates time difference between the ASN submission and the shipping time
Flags if it’s too early or too late
Tools used:
Python datetime module

5️⃣ AS2 Certificate Check
Problem:
Walmart requires a valid AS2 certificate for all EDI transfers.
Code:
if not edi_file.has_valid_certificate():
    errors.append("AS2 certificate missing or expired")

How it's done:
The code uses OpenSSL or cryptography library to check the cert expiration
If expired, the EDI is rejected

✅ Output JSON Report
The app generates a Retail Link-style report like this:
{
  "vendor_id": "WMTIN-REL100",
  "shipment_id": "ASN-20250712-8574",
  "errors": [
    { "segment": "GSTIN", "message": "Invalid prefix for Maharashtra" },
    { "segment": "PRF", "message": "PO number missing" }
  ],
  "ready_to_submit": false
}


📊 Streamlit Dashboard (Frontend)
The frontend app is built in Streamlit, which makes it easy to:
Upload EDI file
Select state for GST validation
Run all checks
View error messages
Export a JSON error report
Streamlit Code Sample:
st.file_uploader("Upload EDI File")
st.selectbox("Select Your State", ["MH", "TN", "KA"])

if st.button("Validate"):
    run_all_validations()
    st.json(output_report)


🧪 Summary: What Happens Behind the Scenes?
Here’s the full picture:
1. User uploads EDI file via Streamlit UI
2. Python code parses the file and breaks it into segments
3. Each validator runs one rule set:
   - Structure → Are the segments in the right order?
   - PO → Is PRF segment there?
   - GTIN → Do product codes exist in GS1 master?
   - GSTIN → Does it match format and state?
   - ASN Timing → Is ASN too early or late?
   - AS2 → Is certificate valid?
4. Errors collected → shown in browser
5. JSON report generated → vendor can fix & retry


Would you like me to generate example code files (like the validate_gstin() function or Streamlit app)? Or a visual architecture diagram showing how the pieces connect?

