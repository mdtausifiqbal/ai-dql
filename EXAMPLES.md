# AI-DQL --- Practical Usage Examples

**Version:** 1.0\
**Purpose:** Copy-ready examples showing how to write AI-DQL query
scripts using the specification in the [AI-DQL
repository](https://github.com/mdtausifiqbal/ai-dql).

> These are illustrative query scripts, not executable SQL. Give the AI
> both the AI-DQL specification (for example, the README) and the
> relevant source file or content. Ask it to follow the script, use only
> accessible source data, and report unsupported operations or
> validation findings.

------------------------------------------------------------------------

## 1. Basic text extraction

**Use case:** Extract key information from a paragraph, email, or
plain-text document.

``` text
[LANGUAGE]
NAME = AI-DQL
PURPOSE = Extract customer contact details from supplied text.

[DATA SOURCE]
SOURCE = TEXT
LOCATION = USER_SUPPLIED_TEXT
CONTENT = RECORDS
SCOPE = Entire supplied text

[COLUMNS]
CUSTOMER_NAME: TEXT
PHONE: TEXT
EMAIL: TEXT

[SCRIPT]
STEP 1: records = EXTRACT_RECORDS(SOURCE, {
  CUSTOMER_NAME: TEXT,
  PHONE: TEXT,
  EMAIL: TEXT
})
RETURN records

[VALIDATION]
MODE = NORMAL
REQUIRED_COLUMNS = [CUSTOMER_NAME]
ALLOW_NULL = true
ON_ERROR = REPORT

[OUTPUT]
FORMAT = TABLE
COLUMNS = [CUSTOMER_NAME, PHONE, EMAIL]
INCLUDE_HEADERS = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Prompt to provide with it:**\
"Use this AI-DQL script on the text below. Do not guess missing phone
numbers or email addresses."

------------------------------------------------------------------------

## 2. Extract tables from a PDF

**Use case:** Read a service register, invoice, statement, or tabular
PDF.

``` text
[DATA SOURCE]
SOURCE = PDF
LOCATION = uploaded service-register.pdf
CONTENT = TABLE
SCOPE = Pages 1-10
OPTIONS = {
  PAGE_RANGE = "1-10",
  OCR = true
}

[SCRIPT]
STEP 1: tables = EXTRACT_TABLES(SOURCE)
STEP 2: service_table = EXTRACT_TABLE(tables, "Service Register")
STEP 3: result = SELECT_COLUMNS(service_table, [
  "Call No",
  "Customer Name",
  "Registered Phone",
  "Status",
  "Call Logged Date"
])
RETURN result

[VALIDATION]
MODE = NORMAL
REQUIRED_COLUMNS = ["Call No", "Customer Name"]
ALLOW_NULL = true
CHECK_TYPES = true
ON_ERROR = REPORT

[OUTPUT]
FORMAT = TABLE
COLUMN_ORDER = DEFINED
INCLUDE_HEADERS = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Note:** Change the table name and page range to match the actual PDF.
If the PDF has no extractable table, ask for OCR/text extraction and
report that limitation rather than inventing rows.

------------------------------------------------------------------------

## 3. Clean and standardize an Excel service register

**Use case:** Normalize names, phone numbers, status text, and column
names.

``` text
[DATA SOURCE]
SOURCE = EXCEL
LOCATION = uploaded service-register.xlsx
CONTENT = TABLE
SCOPE = Entire worksheet
OPTIONS = {
  SHEET_NAME = "Sheet1",
  HEADER_ROW = 1
}

[COLUMNS]
CALL_NO: TEXT {
  SOURCE = "Call No"
  REQUIRED = true
}
CUSTOMER_NAME: TEXT {
  SOURCE = "Customer Name"
}
REGISTERED_PHONE: TEXT {
  SOURCE = "Registered Phone"
}
STATUS: UPPERCASE {
  SOURCE = "Status"
}
PINCODE: TEXT {
  SOURCE = "Pincode"
}

[SCRIPT]
STEP 1: source_table = EXTRACT_TABLE(SOURCE, "Sheet1")
STEP 2: selected = SELECT_COLUMNS(source_table, [
  "Call No", "Customer Name", "Registered Phone", "Status", "Pincode"
])
STEP 3: cleaned = ADD_COLUMN(selected, "CUSTOMER_NAME_CLEAN",
  PROPER(TRIM("Customer Name")))
STEP 4: cleaned = ADD_COLUMN(cleaned, "STATUS_CLEAN",
  UPPER(TRIM("Status")))
STEP 5: cleaned = REORDER_COLUMNS(cleaned, [
  "Call No", "CUSTOMER_NAME_CLEAN", "Registered Phone",
  "STATUS_CLEAN", "Pincode"
])
RETURN cleaned

[VALIDATION]
MODE = NORMAL
REQUIRED_COLUMNS = ["Call No"]
UNIQUE_KEY = ["Call No"]
ALLOW_NULL = true
CHECK_TYPES = true
CHECK_ROW_COUNT = true
PRESERVE_SOURCE = true
ON_ERROR = REPORT

[OUTPUT]
FORMAT = XLSX
INCLUDE_HEADERS = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Important:** Keep phone numbers and pincodes as `TEXT` so leading
zeroes are not lost. The example uses derived columns rather than
overwriting the original values.

------------------------------------------------------------------------

## 4. Filter pending service calls and sort by date

**Use case:** Produce a daily pending-call list.

``` text
[DATA SOURCE]
SOURCE = CSV
LOCATION = uploaded service-register.csv
CONTENT = TABLE
SCOPE = All rows
OPTIONS = {
  ENCODING = "UTF-8",
  HEADER_ROW = 1
}

[SCRIPT]
STEP 1: calls = EXTRACT_TABLE(SOURCE, "service-register")
STEP 2: pending = FILTER_ROWS(calls,
  UPPER(TRIM(Status)) = "PENDING")
STEP 3: ordered = SORT_ROWS(pending, "Call Logged Date", "ASC")
STEP 4: result = SELECT_COLUMNS(ordered, [
  "Call No",
  "Customer Name",
  "Registered Phone",
  "Product Category",
  "Call Logged Date",
  "CVO"
])
RETURN result

[VALIDATION]
MODE = NORMAL
REQUIRED_COLUMNS = ["Call No", "Status"]
ALLOW_NULL = true
ON_ERROR = REPORT

[OUTPUT]
FORMAT = CSV
COLUMN_ORDER = DEFINED
INCLUDE_HEADERS = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Adaptation:** Replace `"PENDING"` with the exact status used in your
source file, such as `"OPEN"` or `"IN PROGRESS"`.

------------------------------------------------------------------------

## 5. Apply business rules to classify calls

**Use case:** Derive a call category from source fields without changing
the original data.

``` text
[DATA SOURCE]
SOURCE = EXCEL
LOCATION = uploaded calls.xlsx
CONTENT = TABLE
SCOPE = All rows
OPTIONS = {
  SHEET_NAME = "Calls",
  HEADER_ROW = 1
}

[RULES]
RULE "Classify warranty calls" {
  PRIORITY = 1
  WHEN Warranty Status = "In Warranty"
  THEN SET Call Category = "WARRANTY"
  ELSE CONTINUE
}

RULE "Classify out-of-warranty calls" {
  PRIORITY = 2
  WHEN Warranty Status = "Out of Warranty"
  THEN SET Call Category = "CHARGEABLE"
  ELSE CONTINUE
}

RULE "Classify unknown warranty status" {
  PRIORITY = 3
  WHEN Warranty Status IS NULL
  THEN SET Call Category = "VERIFY"
  ELSE CONTINUE
}

[SCRIPT]
STEP 1: calls = EXTRACT_TABLE(SOURCE, "Calls")
STEP 2: classified = ADD_COLUMN(calls, "Call Category", "UNCLASSIFIED")
STEP 3: result = APPLY_RULES(classified)
RETURN result

[VALIDATION]
MODE = AUDIT
REQUIRED_COLUMNS = ["Call No", "Warranty Status"]
ALLOW_NULL = true
CHECK_RULES = true
ON_WARNING = REPORT

[OUTPUT]
FORMAT = TABLE
INCLUDE_SOURCE_VALUES = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Note:** This demonstrates the intended rule structure. The AI should
flag any ambiguity in how the rule action is represented or applied
rather than silently assuming unsupported syntax.

------------------------------------------------------------------------

## 6. Extract structured fields from WhatsApp chat text

**Use case:** Convert exported chat messages into structured
service-call records.

``` text
[DATA SOURCE]
SOURCE = TEXT
LOCATION = uploaded WhatsApp chat export.txt
CONTENT = RECORDS
SCOPE = All messages containing service-call details

[COLUMNS]
SENT_DATE: DATETIME
BRAND: TEXT
CALL_NO: TEXT
REGISTERED_PHONE: TEXT
ALTERNATE_PHONE: TEXT
ACCOUNT_NAME: TEXT
SYMPTOMS: TEXT
WARRANTY_STATUS: TEXT

[SCRIPT]
STEP 1: messages = EXTRACT_TEXT(SOURCE)
STEP 2: records = EXTRACT_RECORDS(messages, {
  SENT_DATE: DATETIME,
  BRAND: TEXT,
  CALL_NO: TEXT,
  REGISTERED_PHONE: TEXT,
  ALTERNATE_PHONE: TEXT,
  ACCOUNT_NAME: TEXT,
  SYMPTOMS: TEXT,
  WARRANTY_STATUS: TEXT
})
STEP 3: records = ADD_COLUMN(records, "WARRANTY_STATUS_CLEAN",
  CASE(
    [CONTAINS(UPPER(WARRANTY_STATUS), "PRE-SALES")] = "STOCK REPAIR",
    [CONTAINS(UPPER(WARRANTY_STATUS), "IN WARRANTY")] = "IN WARRANTY",
    [CONTAINS(UPPER(WARRANTY_STATUS), "OUT OF WARRANTY")] = "OUT OF WARRANTY"
  ))
RETURN records

[VALIDATION]
MODE = LENIENT
REQUIRED_COLUMNS = ["SENT_DATE", "CALL_NO"]
ALLOW_NULL = true
CHECK_TYPES = true
ON_ERROR = NULL_AND_REPORT
ON_WARNING = REPORT

[OUTPUT]
FORMAT = XLSX
COLUMNS = [
  SENT_DATE, BRAND, CALL_NO, REGISTERED_PHONE,
  ALTERNATE_PHONE, ACCOUNT_NAME, SYMPTOMS, WARRANTY_STATUS_CLEAN
]
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Important:** Chat exports are inconsistent. Treat the mapping as a
starting point; verify the actual message patterns and refine the
extraction rules. Do not infer a call number or warranty status when the
message does not provide one.

------------------------------------------------------------------------

## 7. Find duplicate call numbers

**Use case:** Audit a register for repeated call IDs.

``` text
[DATA SOURCE]
SOURCE = CSV
LOCATION = uploaded service-register.csv
CONTENT = TABLE
SCOPE = All rows

[SCRIPT]
STEP 1: calls = EXTRACT_TABLE(SOURCE, "service-register")
STEP 2: unique_calls = DISTINCT(calls, ["Call No"])
STEP 3: duplicate_audit = CHECK_UNIQUE(calls, ["Call No"])
STEP 4: errors = REPORT_ERRORS()
RETURN {
  UNIQUE_CALLS = unique_calls,
  DUPLICATE_AUDIT = duplicate_audit,
  VALIDATION_ERRORS = errors
}

[VALIDATION]
MODE = AUDIT
UNIQUE_KEY = ["Call No"]
ALLOW_NULL = false
CHECK_ROW_COUNT = true
PRESERVE_SOURCE = true
ON_ERROR = REPORT

[OUTPUT]
FORMAT = REPORT
INCLUDE_SOURCE_VALUES = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Expected result:** A duplicate audit and validation findings. Do not
assume `DISTINCT` itself identifies which duplicate row should be
retained.

------------------------------------------------------------------------

## 8. Group calls by brand and status

**Use case:** Create an operational summary for a service-centre
manager.

``` text
[DATA SOURCE]
SOURCE = EXCEL
LOCATION = uploaded service-register.xlsx
CONTENT = TABLE
SCOPE = All rows
OPTIONS = {
  SHEET_NAME = "Register",
  HEADER_ROW = 1
}

[SCRIPT]
STEP 1: calls = EXTRACT_TABLE(SOURCE, "Register")
STEP 2: summary = AGGREGATE(
  calls,
  ["Brand", "Status"],
  {
    TOTAL_CALLS = COUNT("Call No")
  }
)
STEP 3: ordered = SORT_ROWS(summary, "TOTAL_CALLS", "DESC")
RETURN ordered

[VALIDATION]
MODE = NORMAL
REQUIRED_COLUMNS = ["Brand", "Status", "Call No"]
ALLOW_NULL = false
ON_ERROR = REPORT

[OUTPUT]
FORMAT = TABLE
INCLUDE_HEADERS = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Note:** `COUNT` is shown here as an illustrative aggregate expression.
If the current AI-DQL implementation does not define aggregate
expressions such as `COUNT`, the AI should report that gap and ask for a
supported alternative.

------------------------------------------------------------------------

## 9. Join two files using a common key

**Use case:** Add technician details to service-call records.

``` text
[DATA SOURCE]
SOURCE = ARRAY
LOCATION = Two user-supplied tables: Calls and Technicians
CONTENT = TABLE
SCOPE = All rows from both tables

[SCRIPT]
STEP 1: calls = EXTRACT_TABLE(SOURCE, "Calls")
STEP 2: technicians = EXTRACT_TABLE(SOURCE, "Technicians")
STEP 3: combined = JOIN_TABLES(
  calls,
  technicians,
  ["Technician ID"],
  "LEFT"
)
STEP 4: result = SELECT_COLUMNS(combined, [
  "Call No",
  "Customer Name",
  "Technician ID",
  "Technician Name",
  "Technician Phone"
])
RETURN result

[VALIDATION]
MODE = NORMAL
REQUIRED_COLUMNS = ["Call No", "Technician ID"]
ALLOW_NULL = true
ON_ERROR = REPORT

[OUTPUT]
FORMAT = XLSX
INCLUDE_HEADERS = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Note:** Confirm that the join key has the same meaning and compatible
values in both tables. Unmatched technician records should remain
visible as nulls in a left join.

------------------------------------------------------------------------

## 10. Extract data from a web page

**Use case:** Read a publicly accessible HTML page or a page already
available to the AI environment.

``` text
[DATA SOURCE]
SOURCE = URL
LOCATION = "https://example.com/products"
CONTENT = TABLE
SCOPE = Product listing section
OPTIONS = {
  SELECTOR = ".product-list"
}

[SCRIPT]
STEP 1: page_text = EXTRACT_TEXT(SOURCE)
STEP 2: products = EXTRACT_RECORDS(page_text, {
  PRODUCT_NAME: TEXT,
  PRODUCT_CODE: TEXT,
  PRICE: CURRENCY
})
RETURN products

[VALIDATION]
MODE = NORMAL
REQUIRED_COLUMNS = ["PRODUCT_NAME"]
ALLOW_NULL = true
ON_ERROR = REPORT

[OUTPUT]
FORMAT = JSON
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

**Important:** A URL is not automatically accessible. The execution
environment must be able to retrieve the page. Login-protected pages,
dynamic content, blocked requests, and inaccessible selectors must be
reported.

------------------------------------------------------------------------

## 11. Validate dates and required fields

**Use case:** Audit records before importing them into another system.

``` text
[DATA SOURCE]
SOURCE = CSV
LOCATION = uploaded calls.csv
CONTENT = TABLE
SCOPE = All rows
OPTIONS = {
  ENCODING = "UTF-8",
  HEADER_ROW = 1
}

[SCRIPT]
STEP 1: calls = EXTRACT_TABLE(SOURCE, "calls")
STEP 2: calls = ADD_COLUMN(calls, "DATE_VALID",
  IS_DATE("Call Logged Date", "dd-MM-yyyy"))
STEP 3: calls = ADD_COLUMN(calls, "PHONE_VALID",
  IS_MATCH("Registered Phone", "^[0-9]{10}$"))
STEP 4: required_check = CHECK_REQUIRED(calls, [
  "Call No", "Customer Name", "Call Logged Date"
])
STEP 5: unique_check = CHECK_UNIQUE(calls, ["Call No"])
STEP 6: findings = REPORT_ERRORS()
RETURN {
  CHECKED_RECORDS = calls,
  REQUIRED_FIELD_CHECK = required_check,
  UNIQUE_KEY_CHECK = unique_check,
  FINDINGS = findings
}

[VALIDATION]
MODE = AUDIT
REQUIRED_COLUMNS = ["Call No", "Customer Name", "Call Logged Date"]
UNIQUE_KEY = ["Call No"]
ALLOW_NULL = false
CHECK_TYPES = true
CHECK_RULES = true
PRESERVE_SOURCE = true
ON_ERROR = REPORT

[OUTPUT]
FORMAT = REPORT
INCLUDE_SOURCE_VALUES = true
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

------------------------------------------------------------------------

## 12. Convert data to a clean JSON structure

**Use case:** Prepare records for an API or application import.

``` text
[DATA SOURCE]
SOURCE = JSON
LOCATION = uploaded raw-calls.json
CONTENT = RECORDS
SCOPE = All call records

[SCRIPT]
STEP 1: source_data = EXTRACT_JSON(SOURCE)
STEP 2: records = EXTRACT_RECORDS(source_data, {
  call_no: TEXT,
  customer_name: TEXT,
  phone: TEXT,
  status: TEXT
})
STEP 3: cleaned = ADD_COLUMN(records, "status",
  UPPER(TRIM(status)))
STEP 4: result = SELECT_COLUMNS(cleaned, [
  "call_no", "customer_name", "phone", "status"
])
RETURN result

[VALIDATION]
MODE = STRICT
REQUIRED_COLUMNS = ["call_no", "customer_name"]
ALLOW_NULL = false
CHECK_TYPES = true
ON_ERROR = STOP

[OUTPUT]
FORMAT = JSON
INCLUDE_VALIDATION = true
INCLUDE_SUMMARY = true
```

------------------------------------------------------------------------

## Reusable AI-DQL request template

Copy this template when starting a new task. Replace the placeholders
and remove options that do not apply.

``` text
[LANGUAGE]
NAME = AI-DQL
PURPOSE = Describe the data-processing objective in one sentence.

[DATA SOURCE]
SOURCE = FILE
LOCATION = "uploaded-file-name.ext"
CONTENT = AUTO
SCOPE = Specify sheets, pages, tables, columns, or rows
OPTIONS = {
  SHEET_NAME = "optional sheet",
  PAGE_RANGE = "optional page range",
  HEADER_ROW = 1
}

[COLUMNS]
OUTPUT_FIELD_1: TEXT {
  SOURCE = "Source Field 1"
  REQUIRED = true
}
OUTPUT_FIELD_2: DATE {
  SOURCE = "Source Field 2"
  REQUIRED = false
}

[RULES]
# Add RULE blocks only when conditional business logic is needed.

[SCRIPT]
STEP 1: source_data = EXTRACT_TABLE(SOURCE, "optional table name")
STEP 2: result = SELECT_COLUMNS(source_data, ["Source Field 1", "Source Field 2"])
RETURN result

[VALIDATION]
MODE = NORMAL
REQUIRED_COLUMNS = ["OUTPUT_FIELD_1"]
UNIQUE_KEY = []
ALLOW_NULL = true
ON_ERROR = REPORT
ON_WARNING = REPORT
CHECK_TYPES = true
CHECK_RULES = true
PRESERVE_SOURCE = true

[OUTPUT]
FORMAT = TABLE
COLUMNS = [OUTPUT_FIELD_1, OUTPUT_FIELD_2]
COLUMN_ORDER = DEFINED
INCLUDE_HEADERS = true
INCLUDE_VALIDATION = true
INCLUDE_SOURCE_VALUES = false
INCLUDE_SUMMARY = true
```

------------------------------------------------------------------------

## Practical guidance and current specification notes

1.  **Provide the specification and the data.** The AI-DQL README
    defines the language; the uploaded file, text, or accessible URL
    provides the source data.
2.  **Be precise about scope.** Name the worksheet, page range, table,
    selector, or fields to process.
3.  **Declare output columns and types.** Use `TEXT` for phone numbers,
    pincodes, IDs, and other identifiers that may contain leading
    zeroes.
4.  **Preserve source values.** Prefer derived columns for cleaned or
    classified values so the original data remains auditable.
5.  **Choose validation deliberately.** Use `STRICT` for critical
    imports, `NORMAL` for routine processing, `LENIENT` for messy source
    files, and `AUDIT` when you only want checks and findings.
6.  **Ask for evidence.** Require the AI to report missing columns,
    inaccessible sources, conversion failures, unsupported functions,
    and ambiguous mappings.
7.  **Do not treat examples as proof of implementation.** The README
    lists supported function signatures, but some example expressions
    above (such as `COUNT`, `CONTINUE`, or richer object/array
    expressions) may need syntax clarification or implementation
    support. The AI should identify these rather than silently inventing
    behavior.

**Suggested instruction to prepend to every task:**

"Follow the AI-DQL v1.0 specification. Interpret the script as a
data-processing specification, not as SQL or executable code. Use only
supported functions and accessible source data. Do not fabricate values,
rows, counts, or file outputs. If any syntax, source mapping, function,
or operation is unsupported or ambiguous, explain the issue and propose
the smallest correction before proceeding."
