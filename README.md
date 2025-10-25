

# VouchNet - **Curated Content Protocol (Clarity Smart Contract)**

## 📘 Overview

The **Curated Content Protocol** is a decentralized curation and reputation system built on the **Stacks blockchain**.
It allows participants to:

* **Submit** original content for curation.
* **Appraise** (upvote/downvote) content from others.
* **Reward** creators using STX gratuities (tips).
* **Flag** inappropriate or low-quality submissions.
* **Track** credibility (reputation) of users based on their appraisals.
* **Administrators** can manage topics, submission fees, and remove items.

This system promotes community-driven content curation where users collectively decide what deserves attention.

---

## ⚙️ Smart Contract Summary

| Feature              | Description                                              |
| -------------------- | -------------------------------------------------------- |
| **Language**         | Clarity (Stacks)                                         |
| **Primary Use Case** | Decentralized content submission, voting, and rewards    |
| **Admin Role**       | `PROTOCOL_ADMINISTRATOR` (the contract deployer)         |
| **Data Stored**      | Items, votes, reputation, topics, total submissions      |
| **Currency**         | STX (Stacks Token) for submission charges and gratuities |
| **Safety Checks**    | Balance validation, overflow prevention, access control  |

---

## 🧩 Contract Architecture

### **Constants**

| Constant                                                  | Description                             |
| --------------------------------------------------------- | --------------------------------------- |
| `PROTOCOL_ADMINISTRATOR`                                  | Contract deployer (administrator)       |
| `ERR_UNAUTHORIZED_ACCESS`, `ERR_INVALID_SUBMISSION`, etc. | Standardized error codes                |
| `MIN_HYPERLINK_LENGTH`                                    | Minimum character length for hyperlinks |
| `MAX_UINT`                                                | Max possible uint (for overflow safety) |

---

### **Data Variables**

| Variable                | Type                        | Description                             |
| ----------------------- | --------------------------- | --------------------------------------- |
| `submission-charge`     | `uint`                      | Fee (in STX) required to submit content |
| `aggregate-submissions` | `uint`                      | Counter for total number of submissions |
| `content-topics`        | `list 10 (string-ascii 20)` | List of available curation topics       |

---

### **Data Maps**

| Map                       | Key                                                 | Value                                                                                      | Purpose                          |
| ------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------ | -------------------------------- |
| `curated-items`           | `{ item-identifier: uint }`                         | Item metadata: originator, headline, hyperlink, topic, timestamp, votes, gratuities, flags | Stores curated content           |
| `participant-appraisals`  | `{ participant: principal, item-identifier: uint }` | `{ appraisal: int }`                                                                       | Tracks user’s appraisal per item |
| `participant-credibility` | `{ participant: principal }`                        | `{ metric: int }`                                                                          | Tracks participant’s reputation  |

---

## 🔍 Function Reference

### 🧾 **Public Functions**

#### 1. `contribute-item (headline hyperlink topic)`

Submits a new item for curation.

**Parameters**

* `headline`: `(string-ascii 100)` — title of the submission
* `hyperlink`: `(string-ascii 200)` — URL to content (must be ≥ `MIN_HYPERLINK_LENGTH`)
* `topic`: `(string-ascii 20)` — must exist in `content-topics`

**Logic**

* Checks input validity and sufficient balance.
* Transfers `submission-charge` STX to administrator.
* Registers new item with `item-identifier = aggregate-submissions + 1`.
* Emits a `print` event:

  ```clarity
  { type: "new-item", item-identifier, originator }
  ```

**Returns**

```clarity
(ok item-identifier)
```

---

#### 2. `appraise-item (item-identifier appraisal)`

Allows a user to **upvote (`1`)** or **downvote (`-1`)** an item.

**Logic**

* Validates item existence.
* Updates the appraisals total.
* Updates user credibility metric.
* Emits:

  ```clarity
  { type: "appraisal", item-identifier, appraiser, appraisal }
  ```

**Returns**

```clarity
(ok true)
```

---

#### 3. `reward-originator (item-identifier gratuity-amount)`

Sends a STX gratuity (tip) to the content creator.

**Logic**

* Verifies item existence and sufficient sender balance.
* Updates `gratuities` total.
* Transfers STX from sender → originator.
* Emits:

  ```clarity
  { type: "reward", item-identifier, from, to, amount }
  ```

**Returns**

```clarity
(ok true)
```

---

#### 4. `flag-item (item-identifier)`

Reports a content item as inappropriate or invalid.

**Logic**

* Item must exist.
* Originator **cannot flag** their own content.
* Increments `flags`.
* Emits:

  ```clarity
  { type: "flag", item-identifier, flagger }
  ```

**Returns**

```clarity
(ok true)
```

---

### 🧠 **Read-Only Functions**

| Function                                                       | Description                                             |
| -------------------------------------------------------------- | ------------------------------------------------------- |
| `retrieve-item-details (item-identifier)`                      | Returns the metadata of an item.                        |
| `retrieve-participant-appraisal (participant item-identifier)` | Gets a participant’s appraisal of a given item.         |
| `retrieve-aggregate-submissions`                               | Returns total number of curated submissions.            |
| `retrieve-participant-credibility (participant)`               | Returns the participant’s reputation metric.            |
| `retrieve-top-items (limit)`                                   | Fetches a list of top-rated items (limited by `limit`). |

---

### 🛠️ **Admin Functions**

#### 1. `adjust-submission-charge (new-charge)`

Updates the STX submission fee.

**Access:** Only `PROTOCOL_ADMINISTRATOR`

**Returns:**

```clarity
(ok true)
```

---

#### 2. `expunge-item (item-identifier)`

Deletes a curated item (e.g., if flagged or invalid).

**Access:** Only `PROTOCOL_ADMINISTRATOR`

**Returns:**

```clarity
(ok true)
```

---

#### 3. `introduce-topic (new-topic)`

Adds a new topic to the list of valid content categories.

**Access:** Only `PROTOCOL_ADMINISTRATOR`

**Constraints**

* Max of 10 topics allowed.
* Topic name must be non-empty.

**Returns:**

```clarity
(ok true)
```

---

## 🧮 Helper Functions (Private)

| Function                        | Purpose                                                |
| ------------------------------- | ------------------------------------------------------ |
| `item-exists (item-identifier)` | Checks if an item exists in `curated-items`.           |
| `not-none`                      | Filters out `none` values.                             |
| `retrieve-item-if-valid`        | Returns only items with non-negative appraisal scores. |
| `get-item-ids (count)`          | Generates a list of numeric item IDs up to a limit.    |
| `enumerate (n)`                 | Creates a list of numbers from `1` to `n` (up to 10).  |
| `is-non-zero`                   | Helper to remove zero values from lists.               |

---

## 💰 Token Flow

| Action                              | Sender | Receiver   | Description                    |
| ----------------------------------- | ------ | ---------- | ------------------------------ |
| Submit content                      | User   | Admin      | Pays submission fee            |
| Reward creator                      | User   | Originator | Sends gratuity (tip)           |
| Admin adjusts fees or deletes items | Admin  | -          | No transfer, governance action |

---

## 🔐 Access Control

| Role                                  | Permissions                                  |
| ------------------------------------- | -------------------------------------------- |
| **User (any principal)**              | Submit, appraise, reward, flag, read         |
| **Administrator (contract deployer)** | Adjust fees, expunge items, introduce topics |

---

## 🧾 Example Interactions

### Submit an item

```clarity
(contract-call? .curation contribute-item "Decentralized News" "https://newsdao.org" "Technology")
```

### Appraise (Upvote)

```clarity
(contract-call? .curation appraise-item u1 1)
```

### Reward creator with 50 STX

```clarity
(contract-call? .curation reward-originator u1 u50000000)
```

### Retrieve content details

```clarity
(contract-call? .curation retrieve-item-details u1)
```

### Adjust submission fee (Admin only)

```clarity
(contract-call? .curation adjust-submission-charge u25)
```

---

## 🧾 License

MIT License © 2025
Developed for open, transparent, and community-driven curation on Stacks.
