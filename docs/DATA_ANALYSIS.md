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
