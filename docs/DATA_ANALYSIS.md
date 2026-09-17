# Data Analysis: AI Support-Ticket Decision Assistant

This document contains the empirical findings and domain discovery from `data/sample_test_cases.json`, `data/tickets.csv`, `docs/DATA_NOTES.md`, and the 6 markdown files in `knowledge_base/`.

---

## 1. Test Cases (`data/sample_test_cases.json`)

### 1.1 Overview & Structure
- **Count**: 5 test cases (`S01` to `S05`).
- **Data Format**: A flat JSON array containing 5 JSON objects with no nested structures.
- **Field Schema**:
  | Field Name | Type | Nullable | Example Values / Description |
  | :--- | :--- | :--- | :--- |
  | `case_id` | `string` | No | `"S01"`, `"S02"`, `"S03"`, `"S04"`, `"S05"` |
  | `message` | `string` | No | Customer ticket text / query |
  | `order_value_inr` | `integer` | No | `3500`, `1200`, `850`, `650`, `900` |
  | `days_since_delivery` | `integer` | Yes | `1`, `10`, `null`, `2`, `null` |
  | `days_since_dispatch` | `integer` | Yes | `null`, `null`, `9`, `null`, `null` |
  | `product_type` | `string` | No | `"non_food"`, `"mixed"`, `"food"`, `"unknown"` |
  | `opened_status` | `string` | No | `"opened"`, `"unopened"`, `"unknown"` |
  | `order_status` | `string` | No | `"delivered"`, `"dispatched"` |
  | `expected_action` | `string` | No | Canonical action label expected from decision engine |

### 1.2 Comprehensive Case Table
| ID | Ticket Text (First 90 Chars) | Expected Action | Other Expected Fields |
| :--- | :--- | :--- | :--- |
| **S01** | `My ₹3,500 order arrived damaged yesterday.` | `REQUEST_PHOTOS` | `order_value_inr`: 3500, `days_since_delivery`: 1, `days_since_dispatch`: null, `product_type`: `"non_food"`, `opened_status`: `"opened"`, `order_status`: `"delivered"` |
| **S02** | `I changed my mind about this unopened non-food product. It arrived 10 days ago.` | `APPROVE_RETURN` | `order_value_inr`: 1200, `days_since_delivery`: 10, `days_since_dispatch`: null, `product_type`: `"non_food"`, `opened_status`: `"unopened"`, `order_status`: `"delivered"` |
| **S03** | `My parcel has still not arrived and it was dispatched 9 days ago.` | `OPEN_SHIPPING_INVESTIGATION` | `order_value_inr`: 850, `days_since_delivery`: null, `days_since_dispatch`: 9, `product_type`: `"mixed"`, `opened_status`: `"unknown"`, `order_status`: `"dispatched"` |
| **S04** | `I ordered strawberry but received chocolate 2 days ago.` | `REPLACE_CORRECT_ITEM` | `order_value_inr`: 650, `days_since_delivery`: 2, `days_since_dispatch`: null, `product_type`: `"food"`, `opened_status`: `"unopened"`, `order_status`: `"delivered"` |
| **S05** | `I want to return this.` | `NEEDS_MORE_INFORMATION` | `order_value_inr`: 900, `days_since_delivery`: null, `days_since_dispatch`: null, `product_type`: `"unknown"`, `opened_status`: `"unknown"`, `order_status`: `"delivered"` |

---

## 2. Action Vocabulary

### 2.1 Distinct Action Labels & Counts
Analysis across both `data/sample_test_cases.json` (5 cases) and historical dataset `data/tickets.csv` (214 rows) identifies **15 total distinct action labels**:

| Action Label (Exact Spelling/Case) | Count in `sample_test_cases.json` | Count in `tickets.csv` | Total Occurrences |
| :--- | :---: | :---: | :---: |
| `APPROVE_REFUND_OR_REPLACEMENT` | 0 | 28 | 28 |
| `APPROVE_REPLACEMENT` | 0 | 12 | 12 |
| `APPROVE_RETURN` | 1 | 18 | 19 |
| `CANCEL_AND_REFUND` | 0 | 12 | 12 |
| `CANNOT_CANCEL_AFTER_DISPATCH` | 0 | 10 | 10 |
| `NEEDS_MORE_INFORMATION` | 1 | 6 | 7 |
| `OFFER_REPLACEMENT_OR_REFUND` | 0 | 10 | 10 |
| `OPEN_SHIPPING_INVESTIGATION` | 1 | 14 | 15 |
| `REJECT_FOOD_RETURN` | 0 | 10 | 10 |
| `REJECT_OPENED_ITEM` | 0 | 10 | 10 |
| `REJECT_OUTSIDE_WINDOW` | 0 | 30 | 30 |
| `REPLACE_CORRECT_ITEM` | 1 | 16 | 17 |
| `REQUEST_DEFECT_EVIDENCE` | 0 | 8 | 8 |
| `REQUEST_PHOTOS` | 1 | 16 | 17 |
| `WAIT_AND_TRACK` | 0 | 14 | 14 |
| **Total** | **5** | **214** | **219** |

### 2.2 Insufficient Information Action
- **Exact Label**: `NEEDS_MORE_INFORMATION`
- **Presence**: Appears in `sample_test_cases.json` (case `S05`) and 6 times in `tickets.csv`. Used whenever essential details (dates, product types, opened condition, or specific defects) are missing from the customer ticket.

### 2.3 Policy-Derived Meanings (Derived Exclusively from Knowledge Base)
Every action maps directly to concrete policy clauses in `knowledge_base/*.md`:

1. `APPROVE_REFUND_OR_REPLACEMENT`:
   - *Source*: `damaged_goods.md` (Rule 2)
   - *Meaning*: Approve refund or replacement without photos for damaged goods valued at ₹2,000 or less reported within 7 calendar days of delivery.
2. `APPROVE_REPLACEMENT`:
   - *Source*: `defective_products.md` (Rule 1 & 2)
   - *Meaning*: Approve replacement for functional defect reported within 14 calendar days of delivery on orders valued at ₹3,000 or less (or after evidence received).
3. `APPROVE_RETURN`:
   - *Source*: `returns.md` (Rule 1)
   - *Meaning*: Approve return for unopened non-food items requested within 14 calendar days of delivery.
4. `CANCEL_AND_REFUND`:
   - *Source*: `cancellations.md` (Rule 1)
   - *Meaning*: Cancel the order and issue a full refund when cancellation is requested before dispatch.
5. `CANNOT_CANCEL_AFTER_DISPATCH`:
   - *Source*: `cancellations.md` (Rule 2 & 3)
   - *Meaning*: Deny direct cancellation because order has already dispatched, advising customer to use return, damaged, or delivery policies once received.
6. `NEEDS_MORE_INFORMATION`:
   - *Source*: `cancellations.md` (Rule 4), `damaged_goods.md` (Rule 5), `defective_products.md` (Rule 5), `returns.md` (Rule 5), `shipping.md` (Rule 5), `wrong_item.md` (Rule 5)
   - *Meaning*: Prompt the customer for missing dates, statuses, product attributes, or evidence necessary to evaluate eligibility.
7. `OFFER_REPLACEMENT_OR_REFUND`:
   - *Source*: `shipping.md` (Rule 4), `wrong_item.md` (Rule 3)
   - *Meaning*: Offer customer choice of replacement or refund when an order is undelivered >10 days post-dispatch, or when a wrong item cannot be replaced due to stock unavailability.
8. `OPEN_SHIPPING_INVESTIGATION`:
   - *Source*: `shipping.md` (Rule 3)
   - *Meaning*: Initiate a carrier investigation for an undelivered order that is 8 to 10 days post-dispatch.
9. `REJECT_FOOD_RETURN`:
   - *Source*: `returns.md` (Rule 3)
   - *Meaning*: Reject return request for food products after delivery, regardless of whether opened or unopened.
10. `REJECT_OPENED_ITEM`:
    - *Source*: `returns.md` (Rule 2)
    - *Meaning*: Reject change-of-mind return request for opened non-food products.
11. `REJECT_OUTSIDE_WINDOW`:
    - *Source*: `damaged_goods.md` (Rule 4), `defective_products.md` (Rule 3), `returns.md` (Rule 1), `wrong_item.md` (Rule 4)
    - *Meaning*: Reject request because reporting window has lapsed (>7 days for damaged goods and wrong items; >14 days for returns and defective items).
12. `REPLACE_CORRECT_ITEM`:
    - *Source*: `wrong_item.md` (Rule 1 & 2)
    - *Meaning*: Send replacement of correct item for wrong item/flavour reported within 7 calendar days of delivery.
13. `REQUEST_DEFECT_EVIDENCE`:
    - *Source*: `defective_products.md` (Rule 2)
    - *Meaning*: Request basic defect evidence for orders valued above ₹3,000 reported within 14 calendar days of delivery before replacement can be approved.
14. `REQUEST_PHOTOS`:
    - *Source*: `damaged_goods.md` (Rule 3)
    - *Meaning*: Request photographs of damaged product and packaging for orders valued above ₹2,000 reported within 7 calendar days of delivery before approving refund/replacement.
15. `WAIT_AND_TRACK`:
    - *Source*: `shipping.md` (Rule 2)
    - *Meaning*: Advise customer to wait and continue tracking shipment when order is 6 to 7 days post-dispatch.

---

## 3. Tickets Dataset (`data/tickets.csv`)

### 3.1 Schema & Columns
The file `data/tickets.csv` contains historical customer tickets with 12 columns:
1. `ticket_id`: Unique integer identifier for the ticket.
2. `customer_id`: Unique customer identifier string (e.g. `C1254`).
3. `customer_name`: Customer given name (e.g. `Vihaan`, `Tanvi`, `Aditi`).
4. `message`: Customer issue description / message text.
5. `order_value_inr`: Order monetary value in INR (integer).
6. `days_since_delivery`: Integer days elapsed since delivery, or empty/null if undelivered or missing.
7. `days_since_dispatch`: Integer days elapsed since order dispatch, or empty/null if not dispatched or missing.
8. `product_type`: Categorical item classification (`food`, `non_food`, `mixed`, `unknown`).
9. `opened_status`: Packaging status (`opened`, `unopened`, `unknown`).
10. `order_status`: Current lifecycle status (`processing`, `dispatched`, `delivered`).
11. `issue_type`: Pre-assigned issue category (`defective`, `damaged`, `return`, `cancellation`, `shipping`, `wrong_item`).
12. `resolved_action`: Historical decision/action label assigned to this ticket (one of the 15 canonical action labels).

### 3.2 Dataset Size & Sample Rows
- **Total Rows**: 214 ticket records.
- **Sample Rows**:

| ticket_id | customer_id | customer_name | message | order_value_inr | days_since_delivery | days_since_dispatch | product_type | opened_status | order_status | issue_type | resolved_action |
| :---: | :---: | :---: | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| `199` | `C1254` | `Vihaan` | The device is defective and stops working after a few minutes. | `3999` | `9` | *(empty)* | `non_food` | `opened` | `delivered` | `defective` | `REQUEST_DEFECT_EVIDENCE` |
| `27` | `C1125` | `Tanvi` | The product was broken when I opened the parcel. | `1799` | `4` | *(empty)* | `food` | `unopened` | `delivered` | `damaged` | `APPROVE_REFUND_OR_REPLACEMENT` |
| `58` | `C1129` | `Aditi` | I changed my mind. The non-food product is still unopened. | `2499` | `10` | *(empty)* | `non_food` | `unopened` | `delivered` | `return` | `APPROVE_RETURN` |

### 3.3 Relationship to Test Cases (`sample_test_cases.json`)
1. **Disjoint Sets**: None of the 5 test cases in `sample_test_cases.json` exist verbatim as messages in `tickets.csv`. The sample test cases are a curated out-of-sample evaluation probe covering 5 major distinct actions.
2. **Feature Alignment**:
   - Both datasets share the core feature set: `message`, `order_value_inr`, `days_since_delivery`, `days_since_dispatch`, `product_type`, `opened_status`, and `order_status`.
   - `tickets.csv` contains `ticket_id`, `customer_id`, `customer_name`, and `issue_type`. In contrast, test cases have `case_id` and omit customer metadata and pre-classified `issue_type`.
   - The production assistant cannot rely on an `issue_type` hint: it must infer the issue type and applicable policy directly from the ticket text and metadata using the RAG knowledge base.

---

## 4. Knowledge Base, Per File

### 4.1 Character & Token Sizing Table
Calculated using exact UTF-8 character lengths with token estimates computed at `chars / 4.0`:

| Policy File | Headings | Character Count | Estimated Tokens (chars/4) |
| :--- | :--- | :---: | :---: |
| `cancellations.md` | `# Cancellation Policy` | 367 | 91.8 |
| `damaged_goods.md` | `# Damaged Goods Policy` | 621 | 155.2 |
| `defective_products.md` | `# Defective Product Policy` | 500 | 125.0 |
| `returns.md` | `# Returns Policy` | 509 | 127.2 |
| `shipping.md` | `# Shipping and Delivery Policy` | 509 | 127.2 |
| `wrong_item.md` | `# Wrong Item Policy` | 455 | 113.8 |
| **Total** | **6 Files** | **2,961** | **740.2** |

### 4.2 Concrete Policy Rules & Implied Actions

#### 1. `cancellations.md`
- **Heading**: `# Cancellation Policy`
- **Rule 1**: An order may be cancelled for a full refund before it is dispatched.
  - *Condition*: `order_status` in (`processing`, pending dispatch).
  - *Implied Action*: `CANCEL_AND_REFUND`
- **Rule 2**: Once an order has been dispatched, it cannot be cancelled through the cancellation process.
  - *Condition*: `order_status` in (`dispatched`, `delivered`).
  - *Implied Action*: `CANNOT_CANCEL_AFTER_DISPATCH`
- **Rule 3**: After dispatch, customer may use applicable returns, damaged-goods, wrong-item, or delivery policy.
  - *Condition*: Guidance / redirect to subsequent policies.
- **Rule 4**: If order dispatch status is unknown, request more information.
  - *Condition*: `order_status` missing or unknown.
  - *Implied Action*: `NEEDS_MORE_INFORMATION`

#### 2. `damaged_goods.md`
- **Heading**: `# Damaged Goods Policy`
- **Rule 1**: Damage must be reported within 7 calendar days of delivery.
  - *Condition*: Baseline eligibility window `days_since_delivery <= 7`.
- **Rule 2**: For damaged orders valued at ₹2,000 or less, customer may receive a refund or replacement without photographic evidence.
  - *Condition*: `order_value_inr <= 2000` and `days_since_delivery <= 7`.
  - *Implied Action*: `APPROVE_REFUND_OR_REPLACEMENT`
- **Rule 3**: For damaged orders valued above ₹2,000, photographs of the damaged product and packaging must be requested before a refund or replacement is approved.
  - *Condition*: `order_value_inr > 2000` and `days_since_delivery <= 7`.
  - *Evidence Required*: Photos of product and packaging.
  - *Implied Action*: `REQUEST_PHOTOS`
- **Rule 4**: Damage reported more than 7 days after delivery is not eligible under the standard damaged-goods policy.
  - *Condition*: `days_since_delivery > 7`.
  - *Implied Action*: `REJECT_OUTSIDE_WINDOW`
- **Rule 5**: If customer does not provide enough information to determine when the order was delivered or what was damaged, request more information.
  - *Condition*: Missing `days_since_delivery` or missing damage specifics.
  - *Implied Action*: `NEEDS_MORE_INFORMATION`

#### 3. `defective_products.md`
- **Heading**: `# Defective Product Policy`
- **Rule 1**: A functional defect reported within 14 calendar days of delivery is eligible for replacement.
  - *Condition*: `days_since_delivery <= 14` and `order_value_inr <= 3000`.
  - *Implied Action*: `APPROVE_REPLACEMENT`
- **Rule 2**: For orders valued above ₹3,000, basic evidence of the defect must be requested before replacement is approved.
  - *Condition*: `days_since_delivery <= 14` and `order_value_inr > 3000`.
  - *Evidence Required*: Basic evidence of defect.
  - *Implied Action*: `REQUEST_DEFECT_EVIDENCE`
- **Rule 3**: Defects reported more than 14 days after delivery are not eligible under the standard defect policy.
  - *Condition*: `days_since_delivery > 14`.
  - *Implied Action*: `REJECT_OUTSIDE_WINDOW`
- **Rule 4**: Cosmetic damage should be evaluated under the Damaged Goods Policy.
  - *Boundary Rule*: Distinguishes cosmetic damage (redirects to `damaged_goods.md`) from functional defects.
- **Rule 5**: If the nature of the defect or delivery date is missing, request more information.
  - *Condition*: Missing defect details or `days_since_delivery`.
  - *Implied Action*: `NEEDS_MORE_INFORMATION`

#### 4. `returns.md`
- **Heading**: `# Returns Policy`
- **Rule 1**: Unopened non-food products may be returned within 14 calendar days of delivery.
  - *Condition*: `product_type == "non_food"`, `opened_status == "unopened"`, `days_since_delivery <= 14`.
  - *Implied Action*: `APPROVE_RETURN`
- **Rule 2**: Opened non-food products are not eligible for a change-of-mind return.
  - *Condition*: `product_type == "non_food"`, `opened_status == "opened"`.
  - *Implied Action*: `REJECT_OPENED_ITEM`
- **Rule 3**: Food products are not eligible for change-of-mind returns after delivery, even if unopened.
  - *Condition*: `product_type == "food"`.
  - *Implied Action*: `REJECT_FOOD_RETURN`
- **Rule 4**: Damaged or defective products are handled under the Damaged Goods Policy rather than this policy.
  - *Precedence Rule*: Return policy is strictly for change-of-mind.
- **Rule 5**: If product type, opened/unopened status, or delivery date is missing and is necessary to decide eligibility, request more information.
  - *Condition*: Missing `product_type`, `opened_status`, or `days_since_delivery`.
  - *Implied Action*: `NEEDS_MORE_INFORMATION`

#### 5. `shipping.md`
- **Heading**: `# Shipping and Delivery Policy`
- **Rule 1**: Standard domestic orders are expected to arrive within 5 calendar days after dispatch.
  - *Baseline*: Expected delivery SLA.
- **Rule 2**: If an order has not arrived 6 or 7 days after dispatch, advise the customer to wait and continue tracking the shipment.
  - *Condition*: `order_status == "dispatched"` and `days_since_dispatch` in (`6`, `7`).
  - *Implied Action*: `WAIT_AND_TRACK`
- **Rule 3**: If an order has not arrived 8 to 10 days after dispatch, open a shipping investigation.
  - *Condition*: `order_status == "dispatched"` and `days_since_dispatch` in (`8`, `9`, `10`).
  - *Implied Action*: `OPEN_SHIPPING_INVESTIGATION`
- **Rule 4**: If an order has not arrived more than 10 days after dispatch, offer a replacement or refund.
  - *Condition*: `order_status == "dispatched"` and `days_since_dispatch > 10`.
  - *Implied Action*: `OFFER_REPLACEMENT_OR_REFUND`
- **Rule 5**: If dispatch date or delivery status is unclear, request more information.
  - *Condition*: Missing `days_since_dispatch` or `order_status`.
  - *Implied Action*: `NEEDS_MORE_INFORMATION`

#### 6. `wrong_item.md`
- **Heading**: `# Wrong Item Policy`
- **Rule 1**: A wrong item or wrong flavour must be reported within 7 calendar days of delivery.
  - *Condition*: `days_since_delivery <= 7`.
- **Rule 2**: Eligible reports should receive a replacement of the correct item.
  - *Condition*: Within 7 days, item available.
  - *Implied Action*: `REPLACE_CORRECT_ITEM`
- **Rule 3**: If the originally ordered item is unavailable, offer a refund.
  - *Condition*: Item out of stock.
  - *Implied Action*: `OFFER_REPLACEMENT_OR_REFUND`
- **Rule 4**: Reports submitted more than 7 days after delivery are not eligible under the standard wrong-item policy.
  - *Condition*: `days_since_delivery > 7`.
  - *Implied Action*: `REJECT_OUTSIDE_WINDOW`
- **Rule 5**: If the report does not identify what was ordered versus what was received, request more information.
  - *Condition*: Missing ordered/received item description.
  - *Implied Action*: `NEEDS_MORE_INFORMATION`
