# AS3 Troubleshooting: Start Here

If your AS3 declaration failed, do not panic. AS3 provides specific feedback to help locate the syntax or logic error preventing deployment. Follow this guide to interpret the error codes and resolve the issue.

## 1. The "Two Codes" Distinction: HTTP vs. JSON
Before interpreting the error, you must distinguish between the **HTTP Status Code** (Transport) and the **AS3 JSON Response Code** (Application Logic).

*   **HTTP Status Code:** This is the standard web response received by your API client (e.g., Postman, cURL, Ansible). It indicates the success or failure of the *transmission* and the API framework's ability to handle the request [1].
*   **JSON Response Code:** Found inside the JSON body of the response (e.g., `"results": [{"code": 422...}]`). This represents the specific outcome of the AS3 declaration processing [2].

> **Rule of Thumb:**
> *   If the **HTTP Code** is **200 OK**, the request reached AS3. You must then check the **JSON body** to see if the declaration was actually applied or if it failed validation [3].
> *   If the **HTTP Code** is **202 Accepted**, the request is valid but processing asynchronously. You must poll the specific Task ID provided in the response to get the final success/failure result [4], [5], [6].

---

## 2. HTTP Status Code Reference
*Use the HTTP code to determine the category of the problem.*

| HTTP Code | Meaning | Likely Cause & Next Step |
| :--- | :--- | :--- |
| **200 OK** | **Success** (mostly) | The API call finished. Check the JSON body. If `"message": "success"`, you are done. If `"message": "no change"`, the config already matched the declaration [3]. |
| **202 Accepted** | **Processing** | The declaration is large or the system is busy. AS3 is processing this in the background. **Action:** Poll the task URL provided (`/mgmt/shared/appsvcs/task/<id>`) to see the final result [4], [5], [6]. |
| **400 Bad Request** | **Client Error** | Malformed JSON syntax (missing brackets, commas) or invalid request parameters [7], [8]. **Action:** Validate your JSON syntax. |
| **401 Unauthorized** | **Auth Failure** | Invalid credentials (Basic Auth) or expired token (`X-F5-Auth-Token`) [9]. **Action:** Re-authenticate. |
| **404 Not Found** | **Missing Endpoint** | Usually means AS3 is not installed, not running, or you are hitting the wrong URI [1], [10]. **Action:** Check `/mgmt/shared/appsvcs/info`. |
| **422 Unprocessable**| **Logic Error** | **Most Common.** The JSON is valid, but the configuration logic is invalid (e.g., missing mandatory fields, name conflicts, schema violations) [2], [11], [12]. **Action:** Read the specific error message in the JSON body. |
| **500 Internal Error**| **Server Error** | System-level failure, often timeouts or memory issues on the F5 device [7], [13]. **Action:** Check system resources (see Section 4). |
| **503 Unavailable** | **Busy** | The device is busy processing another declaration [14]. **Action:** Wait and retry, or enable "Burst Handling". |

---

## 3. Common JSON Error Messages (422 Errors)
*If you receive a 422 code, the JSON body `response` or `errors` field will tell you exactly what is wrong [2].*

**Error: "declaration is invalid" / "should have required property"**
*   **Cause:** You are violating the AS3 JSON Schema. You are missing a mandatory field (e.g., `virtualPort` in a Service object) or using an incorrect type (string vs. integer) [2], [15], [16].
*   **Fix:** Use a validator (like Visual Studio Code) to check your JSON against the AS3 Schema before deploying [17].

**Error: "The requested object name (...) is invalid"**
*   **Cause:** You used characters not allowed in object names [18].
*   **Fix:** Ensure names contain only alphanumeric characters, hyphens (-), or underscores (_) [19].

**Error: "The requested ... was not found"**
*   **Cause:** You referenced an object (like a Pool, Profile, or iRule) that does not exist in the declaration or on the BIG-IP [20], [21].
*   **Fix:**
    *   If the object is defined *inside* your AS3 declaration, use the `"use": "objectName"` pointer.
    *   If the object exists on the BIG-IP (e.g., in `/Common`), use the `"bigip": "/Common/objectName"` pointer [22], [23].

**Error: "Virtual Server ... illegally shares destination address..."**
*   **Cause:** You are trying to create a Virtual Server on an IP:Port combination that is already claimed by another Tenant or Partition on the device [24], [25].
*   **Fix:** Ensure your IP:Port combinations are unique, or use the `shareAddresses` property if you intend to reuse an IP across ports/tenants [25], [26], [27].

**Error: "Tenant ... cannot be deleted because it contains configuration item..."**
*   **Cause:** Someone manually added an object (via GUI or CLI) into the partition managed by AS3. AS3 cannot delete the Tenant because it doesn't know about that manual object [28], [29], [30].
*   **Fix:** Manually delete the specific object mentioned in the error using the BIG-IP GUI or CLI, then resubmit your AS3 declaration.

---

## 4. Dealing with System Errors (500/503)
*These errors indicate the BIG-IP is struggling to process the request, not necessarily that your JSON is wrong.*

**Error: "AsyncContext timeout" or "save sys config failed" (500)**
*   **Cause:** The declaration is too large, taking too long to process, or the device is under heavy load [7], [13].
*   **Fixes:**
    1.  **Use Async:** Append `?async=true` to your POST URL (e.g., `/declare?async=true`). This offloads processing and prevents HTTP timeouts [7], [5], [6].
    2.  **Restart Services:** The REST API services might be stuck. Run `bigstart restart restnoded restjavad` on the BIG-IP CLI [31].
    3.  **Memory:** If this happens frequently, the `restjavad` process may need more memory allocated (`sys db provision.extramb`).

**Error: "Public URI path not registered" (404)**
*   **Cause:** AS3 is not installed or has crashed [1].
*   **Fix:** Verify installation by sending a GET to `/mgmt/shared/appsvcs/info`. If it fails, reinstall the AS3 RPM via **iApps > Package Management LX** [32].

---

## 5. Debugging Tools
If the error message is vague, use these tools to dig deeper.

1.  **Validate in VS Code:**
    *   Do not guess. Paste your declaration into Visual Studio Code and reference the AS3 Schema. It will underline errors (missing fields, wrong types) in real-time before you even try to deploy [17].

2.  **Dry-Run:**
    *   Send your declaration with `"action": "dry-run"` in the AS3 class. This asks the BIG-IP "Would this work?" without actually changing any configuration. It allows you to catch logic errors safely [33], [34], [35].

3.  **Check the Logs:**
    *   **AS3 Log:** `/var/log/restnoded/restnoded.log` – This is the specific log for AS3. Look here for specific worker errors [36], [37].
    *   **API Log:** `/var/log/restjavad.0.log` – Look here if you are getting 500 errors or connection issues [28], [29].