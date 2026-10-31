# Examples

This directory contains example AI-DQL scripts. Each script is a standalone
query definition that can be supplied to an AI-DQL execution workflow together
with its referenced source data.

## Available examples

| File | Description |
| --- | --- |
| [`customers.aidql`](./customers.aidql) | Cleans and validates customer records, applies customer rules, and produces JSON output. |

## Running an example

1. Provide the AI-DQL specification from the repository root.
2. Provide the selected `.aidql` script.
3. Provide every source file referenced by the script. For
   `customers.aidql`, this is an Excel workbook named `customers.xlsx` with a
   `Customers` sheet.
4. Ask the execution environment to report validation errors and warnings
   instead of inventing missing source data.

Examples are documentation fixtures; they are not executable by this
repository on their own.
