# MuleSoft Generic File Transfer Application Analysis

This document outlines the purpose and key configurations found across the provided MuleSoft project files, which implement a **generic service for receiving and storing files** via both standard **HTTP API** and **AS2 (Applicability Statement 2)**.

---

## 1. Global Configuration (`global.xml`)

This file defines the **shared, reusable components** for the application, ensuring consistency and preventing configuration duplication.

| Component       | Type                     | Purpose                                      | Key Configuration Details |
|-----------------|--------------------------|----------------------------------------------|----------------------------|
| **HTTP Listener** | `http:listener-config`   | Configures the HTTP endpoint for receiving requests. | Listens on `${http.host}:${http.port}` (e.g., `8081`). Uses TLS config. |
| **AS2 Listener**  | `as2-mule4:listener-config` | Configures the secure AS2 endpoint. | Uses properties for self-config (`as2.request.self.name`, `x509Alias`, `email`) and partner-config. Includes `check-for-duplicate-messages` using the `Object_store`. |
| **Object Store**  | `os:object-store`        | Used by the AS2 connector for message deduplication. | Named `Object_store`. Configured with `entryTtl` and `expirationInterval` of 30 seconds. |
| **File Config**   | `file:config`            | Defines the connection parameters for the File Write operations. | Named `File_Config`. Uses a `workingDir` property (e.g., `D:\Learning\P...`) for local file operations. |

---

## 2. Common Implementation Sub-Flows (`common-imp.xml`)

This file contains **reusable sub-flows** for standard operations like logging and file processing, promoting the **DRY (Don't Repeat Yourself)** principle.

### 🧩 `common-common-log` (Sub-Flow)

A **generic logging utility** designed to standardize log messages across the application.

- **Variables:**
  - `inboundDetails`: A DataWeave transformation capturing runtime details like:
    - `correlationId`
    - `transactionId` (UUID)
    - `timeStamp`
    - Incoming request details (`headers`, `queryParams`)
  - `correlationId`: Ensures a correlationId is present, defaulting to a new UUID if one is not already set.
- **Logging:** Logs the entire `inboundDetails` object at the `INFO` level.

---

### 📁 `generic-file-transfer-read-file` (Sub-Flow)

A sub-flow to **read a file** from the file system.

- **Operation:** Uses `file:read`
- **Path:** Reads from the path specified by `vars.filePath`

---

## 3. HTTP API Flow (`generic-file-transfer-api.xml`)

This flow handles **file uploads** via a standard HTTP endpoint.

| Component | Description |
|------------|--------------|
| **Listener** | Listens on `http:listener/api/transfer-file` (using the listener defined in `global.xml`). |
| **Error Handling** | Uses a standard `http:listener` error handler to return appropriate `httpStatusCode` and `responseHeaders` on failure. |
| **Set Variables** | Transforms inbound headers to set variables:<br>• `fileName` (from `attributes.headers.filename`)<br>• `fileType` (from `attributes.headers.'content-type'` or payload media type) |
| **File Write** | Performs `file:write` to persist the file content.<br>Destination path: `target/${vars.fileName}` using the pre-configured `File_Config`. |
| **Response Transform** | Sets `httpStatusCode` (default `200`) and constructs a success message payload including the final file path. |

---

## 4. AS2 Transfer Flow (`generic-file-transfer-as2.xml`)

This flow handles **secure file exchange using AS2**, leveraging the AS2 Listener configuration from `global.xml`.

| Component | Description |
|------------|--------------|
| **Listener** | Listens on `as2-mule4:listener/receive/file` using the configured AS2 listener. |
| **Log (Start)** | Calls `common-common-log` to capture and log request details. |
| **Sub-Flow Reference** | Invokes `generic-file-transfer-as2-file-write` sub-flow to process the received file. |
| **Log (End)** | Logs completion of the AS2 operation. |

---

### 🧩 `generic-file-transfer-as2-file-write` (Sub-Flow)

This sub-flow mirrors the file writing logic of the HTTP API flow but is tailored for the AS2 context.

- **Set Variables:**  
  - `fileName` (from `attributes.fileName`)  
  - `fileType` (from `attributes.'content-type'` or payload media type)
- **File Write:**  
  Writes the file to `target/${vars.fileName}` using `File_Config`.
- **Response:**  
  Returns the original AS2 attributes (including MDN-related properties) as the final payload.

---

## 5. Configuration Properties

The application uses **external YAML configuration files** for environment-specific settings.

| File | Section | Key Properties | Purpose |
|------|----------|----------------|----------|
| **common-properties.yaml** | `http:` | `host: 0.0.0.0`, `port: 8081`, `basepath: /` | Defines the listener configuration used by HTTP API and AS2 endpoints in `global.xml`. |
| **dev-properties.yaml** | `http:request:` | `host: 0.0.0.0`, `port: 8082`, `basepath: /api` | Defines configuration for making outbound HTTP requests (e.g., sending files to external parties). Includes specific paths and timeouts. |
| **dev-properties.yaml** | `partner-as2:` | `partner:`, `self:`, `keystore-password:` | Defines parameters for outbound AS2 connection (partner = PROWESS, self = MULETRAINS). Includes keystore paths and passwords. |
| **dev-properties.yaml** | `as2:` | `partner:`, `self:` | Defines parameters for the inbound AS2 listener in `global.xml`. |

---

📘 **Summary**

The MuleSoft Generic File Transfer Application provides a **unified mechanism** for:
- Accepting and storing files via **HTTP** and **AS2**
- Maintaining consistent **logging**, **configuration**, and **file handling**
- Supporting **reusability** through common sub-flows and environment-based properties

---
