# DIVA (Damn Insecure and Vulnerable App) — Writeup

**App:** `jakhar.aseem.diva` (DIVA Beta)  
**Author:** Aseem Jakhar  
**Platform:** Android (tested on Genymotion emulator — vbox86p)  
**Tools Used:** ADB, JADX-GUI, SQLite3, PowerShell, Logcat  
**Date:** September 2026

---

## Table of Contents

1. [Challenge 1 — Insecure Logging](#challenge-1--insecure-logging)
2. [Challenge 2 — Hardcoding Issues Part 1](#challenge-2--hardcoding-issues-part-1)
3. [Challenge 3 — Insecure Data Storage Part 1](#challenge-3--insecure-data-storage-part-1)
4. [Challenge 4 — Insecure Data Storage Part 2](#challenge-4--insecure-data-storage-part-2)
5. [Challenge 5 — Insecure Data Storage Part 3](#challenge-5--insecure-data-storage-part-3)
6. [Challenge 6 — Insecure Data Storage Part 4](#challenge-6--insecure-data-storage-part-4)
7. [Challenge 7 — Input Validation Issues Part 1 (SQL Injection)](#challenge-7--input-validation-issues-part-1-sql-injection)
8. [Challenge 8 — Input Validation Issues Part 2 (Local File Read via WebView)](#challenge-8--input-validation-issues-part-2-local-file-read-via-webview)
9. [Challenge 9 — Access Control Issues Part 1 (API Credentials)](#challenge-9--access-control-issues-part-1-api-credentials)
10. [Challenge 10 — Access Control Issues Part 2 (API Credentials)](#challenge-10--access-control-issues-part-2-api-credentials)
11. [Challenge 11 — Hardcoding Issues Part 2 (Native Library)](#challenge-11--hardcoding-issues-part-2-native-library)

---

## Challenge 1 — Insecure Logging

**Objective:** Find out what is being logged, where, and how. Identify the vulnerable code.

**Hint:** Insecure logging occurs when developers intentionally or unintentionally log sensitive information such as credentials, session IDs, financial details, etc.

### Steps

1. Opened the **"1. Insecure Logging"** challenge in the DIVA app.
2. Entered a test credit card number (`77777`) in the input field and pressed **CHECK OUT**.
3. On the ADB shell, ran `logcat` with a grep filter to capture sensitive data:

   ```bash
   logcat | grep -iE "card|credit|payment|number"
   ```

4. Observed the following entries in the log output:

   ```
   09-09 03:56:10.106 1861 1861 E diva-log: Error while processing transaction with credit card: 77777
   09-09 04:01:55.119 1861 1861 E diva-log: Error while processing transaction with credit card: 7777777
   ```

### Findings

- The app logs the **full credit card number in plaintext** to the Android system log under the tag `diva-log` at the `ERROR` level.
- Any app with `READ_LOGS` permission (or any user with ADB access) can read this data.

### Vulnerable Code

The `LogActivity` class uses `Log.e()` to write the user-supplied credit card number directly into logcat via `processCC()`.

### Remediation

- Never log sensitive data (credentials, PII, financial info).
- Use log-scrubbing or redaction utilities in production builds.
- Strip verbose/debug logging before release using ProGuard rules or `BuildConfig.DEBUG` checks.

---

## Challenge 2 — Hardcoding Issues Part 1

**Objective:** Find out what is hardcoded and where.

**Hint:** Developers sometimes hardcode sensitive information for ease.

### Steps

1. Opened the APK in **JADX-GUI** and navigated to `HardcodeActivity.java`.
2. Found the hardcoded key check on **line 19**:

   ```java
   if (hckey.getText().toString().equals("vendorsecretkey"))
   ```

3. Entered `vendorsecretkey` in the app's input field and pressed **ACCESS**.
4. Received a Toast message: **"Access granted!"**

### Findings

- The vendor secret key `vendorsecretkey` is **hardcoded in plaintext** in the Java source.
- Anyone who decompiles the APK (using JADX, apktool, etc.) can extract it immediately.

### Remediation

- Do not store secrets in client-side code.
- Use server-side authentication and token-based access.
- If local secrets are unavoidable, use the Android Keystore system or obfuscation (though obfuscation only raises the bar, it doesn't eliminate the risk).

---

## Challenge 3 — Insecure Data Storage Part 1

**Objective:** Find out where/how credentials are stored and the vulnerable code.

**Hint:** Insecure data storage results from storing confidential information insecurely — poor encryption, plain text, access control issues, etc.

### Steps

1. Opened the **"3. Insecure Data Storage - Part 1"** challenge.
2. Entered username `okay` and password `not okay`, then pressed **SAVE**.
3. On the ADB shell, navigated to the app's SharedPreferences directory:

   ```bash
   cd /data/data/jakhar.aseem.diva/shared_prefs
   cat jakhar.aseem.diva_preferences.xml
   ```

4. Output:

   ```xml
   <?xml version='1.0' encoding='utf-8' standalone='yes' ?>
   <map>
       <string name="password">not okay</string>
       <string name="user">okay </string>
   </map>
   ```

### Findings

- Credentials are stored in **plaintext** in `SharedPreferences` XML files.
- On a rooted device, these files are directly readable from the filesystem.

### Vulnerable Code

`InsecureDataStorage1Activity` writes user input directly to `SharedPreferences` without any encryption.

### Remediation

- Encrypt sensitive data before storing it (e.g., using `EncryptedSharedPreferences` from Jetpack Security).
- Use the Android Keystore for key management.
- Avoid storing credentials locally when possible; prefer tokens with expiry.

---

## Challenge 4 — Insecure Data Storage Part 2

**Objective:** Find out where/how credentials are stored and the vulnerable code.

### Steps

1. Opened the **"4. Insecure Data Storage - Part 2"** challenge.
2. Entered credentials and pressed **SAVE**.
3. Used ADB to pull the SQLite database from the device:

   ```
   adb -s 192.168.98.101:5555 pull /data/data/jakhar.aseem.diva/databases/ids2
   ```

4. Opened the database with `sqlite3` and inspected the tables:

   ```
   sqlite3.exe ids2
   .tables
   → android_metadata  myuser
   SELECT * FROM myuser;
   → kihaalhai|vadhiya
   → cool|not cool
   ```

### Findings

- Credentials are stored in a **local SQLite database (`ids2`)** in the `myuser` table with **no encryption**.
- The username and password are stored as plaintext columns.

### Remediation

- Use SQLCipher or equivalent to encrypt the database at rest.
- Avoid storing raw credentials; store hashed passwords if local auth is required.

---

## Challenge 5 — Insecure Data Storage Part 3

**Objective:** Find out where/how credentials are stored and the vulnerable code.

### Steps

1. Opened the **"5. Insecure Data Storage - Part 3"** challenge.
2. Entered username `hello` and password `not hello`, then pressed **SAVE**.
3. On the ADB shell, navigated to the app's data directory:

   ```bash
   cd /data/data/jakhar.aseem.diva
   ls
   ```

4. Found a temporary file: `uinfo6676645149894297904tmp`
5. Read its contents:

   ```bash
   cat uinfo6676645149894297904tmp
   → hello :not hello
   ```

### Findings

- Credentials are stored in a **temporary file** in the app's private data directory in **plaintext**.
- The file name follows a predictable pattern (`uinfo*tmp`).
- On a rooted device, these files are trivially accessible.

### Remediation

- Do not write credentials to temporary files.
- If temp files are necessary, encrypt the content and delete them immediately after use.

---

## Challenge 6 — Insecure Data Storage Part 4

**Objective:** Find out where/how credentials are stored and the vulnerable code.

### Steps

1. Opened the **"6. Insecure Data Storage - Part 4"** challenge.
2. Entered username `well` and password `not well`, then pressed **SAVE**.
3. On the ADB shell, navigated to the SD card:

   ```bash
   cd /sdcard
   ls -la
   ```

4. Found a hidden file: `.uinfo.txt` (14 bytes, created 2026-09-09)
5. Read its contents:

   ```bash
   cat .uinfo.txt
   → well:not well
   ```

### Findings

- Credentials are stored on the **external storage (SD card)** as a hidden `.uinfo.txt` file.
- External storage is **world-readable** — any app with `READ_EXTERNAL_STORAGE` permission can access it.
- This is the most severe insecure storage variant as external storage has no access control.

### Remediation

- Never write sensitive data to external storage.
- Use internal storage with proper file permissions.
- Encrypt all stored credentials using Android Keystore-backed keys.

---

## Challenge 7 — Input Validation Issues Part 1 (SQL Injection)

**Objective:** Access all user data without knowing any username. Three users exist by default; dump all of them with a single malicious search.

**Hint:** Improper or no input validation issues arise when input is not filtered or validated before use.

### Steps

1. Opened the **"7. Input Validation Issues - Part 1"** challenge.
2. First, searched for `admin` legitimately to confirm functionality. The app returned:

   ```
   User: (admin) pass: (passwd123) Credit card: (1234567812345678)
   ```

3. Analyzed the source code in JADX-GUI — `SQLInjectionActivity.java`:

   **Database setup (onCreate):**
   ```java
   this.mDB = openOrCreateDatabase("sqli", 0, null);
   this.mDB.execSQL("DROP TABLE IF EXISTS sqliuser;");
   this.mDB.execSQL("CREATE TABLE IF NOT EXISTS sqliuser(user VARCHAR, password VARCHAR, credit_card VARCHAR);");
   this.mDB.execSQL("INSERT INTO sqliuser VALUES ('admin', 'passwd123', '1234567812345678');");
   this.mDB.execSQL("INSERT INTO sqliuser VALUES ('diva', 'p@ssword', '1111222233334444');");
   this.mDB.execSQL("INSERT INTO sqliuser VALUES ('john', 'password123', '5555666677778888');");
   ```

   **Vulnerable search query (line 39):**
   ```java
   Cursor cr = this.mDB.rawQuery("SELECT * FROM sqliuser WHERE user = '" + srchtxt.getText().toString() + "'", null);
   ```

   The user input is **directly concatenated** into the SQL query with no sanitization or parameterization.

4. Entered the classic SQL injection payload:

   ```
   ' OR '1'='1
   ```

5. The app dumped **all three users' records**:

   ```
   User: (admin) pass: (passwd123) Credit card: (1234567812345678)
   User: (diva) pass: (p@ssword) Credit card: (1111222233334444)
   User: (john) pass: (password123) Credit card: (5555666677778888)
   ```

### Findings

- The app uses `rawQuery()` with **string concatenation**, making it trivially vulnerable to SQL injection.
- A single injection payload dumps all usernames, passwords, and credit card numbers.

### Remediation

- Use **parameterized queries** (prepared statements):
  ```java
  Cursor cr = this.mDB.rawQuery("SELECT * FROM sqliuser WHERE user = ?", new String[]{userInput});
  ```
- Validate and sanitize all user input.
- Apply the principle of least privilege to the database schema.

---

## Challenge 8 — Input Validation Issues Part 2 (Local File Read via WebView)

**Objective:** Access sensitive information apart from a web URL.

**Hint:** Improper or no input validation issues arise when input is not filtered or validated before use.

### Steps

1. Opened the **"8. Input Validation Issues - Part 2"** challenge.
2. The app presents a URL input field with a **VIEW** button (a WebView-based browser).
3. Instead of a web URL, entered a `file://` URI pointing to a local file within the app's data directory:

   ```
   file:///data/data/jakhar.aseem.diva/uinfo16940747295304647754tmp
   ```

4. Pressed **VIEW**. The WebView loaded and displayed the local file contents:

   ```
   jai ho mitra:babaji
   ```

### Findings

- The WebView does not validate or restrict the URI scheme — it allows `file://` protocol access.
- This enables reading **any file accessible to the app** on the local filesystem, including credentials stored by other challenges.
- Combined with knowledge of predictable file paths, this can be used to exfiltrate sensitive data.

### Remediation

- Validate URL input and restrict schemes to `http://` and `https://` only.
- Disable file access in WebView:
  ```java
  webView.getSettings().setAllowFileAccess(false);
  webView.getSettings().setAllowFileAccessFromFileURLs(false);
  ```
- Implement a URL allowlist if only specific domains should be loaded.

---

## Challenge 9 — Access Control Issues Part 1 (API Credentials)

**Objective:** Find hardcoded API credentials in the app.

### Steps

1. Opened the **Vendor API Credentials** screen in the app, which displayed:

   ```
   API Key: 123secretapikey123
   API User name: diva
   API Password: p@ssword
   ```

2. Analyzed the source code in JADX-GUI — `APICredsActivity.java`:

   ```java
   public class APICredsActivity extends AppCompatActivity {
       @Override
       protected void onCreate(Bundle savedInstanceState) {
           super.onCreate(savedInstanceState);
           setContentView(R.layout.activity_apicreds);
           TextView apicview = (TextView) findViewById(R.id.apicTextView);
           apicview.setText("API Key: 123secretapikey123\nAPI User name: diva\nAPI Password: p@ssword");
       }
   }
   ```

### Findings

- API credentials are **hardcoded directly in the Activity source code** as a string literal.
- Decompiling the APK immediately reveals all API keys and credentials.

### Remediation

- Never embed API keys in client-side code.
- Fetch credentials from a secure backend at runtime.
- Use API key restrictions (IP/referrer restrictions) and rotate keys regularly.

---

## Challenge 10 — Access Control Issues Part 2 (API Credentials)

**Objective:** Find hardcoded API credentials for a second service.

### Steps

1. Opened the **Tveeter API Credentials** screen in the app, which displayed:

   ```
   TVEETER API Key: secrettveeterapikey
   API User name: diva2
   API Password: p@ssword2
   ```

2. Analyzed the source code in JADX-GUI — `APICreds2Activity.java`:

   ```java
   public class APICreds2Activity extends AppCompatActivity {
       @Override
       protected void onCreate(Bundle savedInstanceState) {
           super.onCreate(savedInstanceState);
           setContentView(R.layout.activity_apicreds2);
           TextView apicview = (TextView) findViewById(R.id.apic2TextView);
           EditText pintext = (EditText) findViewById(R.id.aci2pinText);
           Button vbutton = (Button) findViewById(R.id.aci2button);
           Intent i = getIntent();
           boolean bcheck = i.getBooleanExtra(getString(R.string.chk_pin), true);
           if (!bcheck) {
               apicview.setText("TVEETER API Key: secrettveeterapikey\nAPI User name: diva2\nAPI Password: p@ssword2");
               return;
           }
           apicview.setText("Register yourself at http://payatu.com to get your PIN and then login with that PIN!");
           pintext.setVisibility(0);
           vbutton.setVisibility(0);
       }

       public void viewCreds(View view) {
           Toast.makeText(this, "Invalid PIN. Please try again", 0).show();
       }
   }
   ```

### Findings

- API credentials are again **hardcoded in the source code**.
- The PIN protection is entirely cosmetic — the `viewCreds()` method always shows "Invalid PIN", and the credentials are displayed when the activity is launched with the `chk_pin` boolean extra set to `false`.
- The access control can be **bypassed by launching the activity directly** with the correct intent extra, or simply by reading the decompiled source.

### Remediation

- Implement server-side access control for sensitive resources.
- Never rely on client-side checks for authorization.
- Store secrets on the backend; provide them only after proper authentication.

---

## Challenge 11 — Hardcoding Issues Part 2 (Native Library)

**Objective:** Find the hardcoded secret in the native library.

### Steps

1. Attempted to extract the APK on the device using `apktool` and `unzip`, but both tools were not available on the emulator:

   ```bash
   vbox86p:/ # apktool d diva-beta.apk -o diva_extracted
   → apktool: not found
   vbox86p:/ # unzip diva-beta.apk -d diva_extracted
   → can't open diva-beta.apk[.zip]
   ```

2. Extracted the APK on the host machine instead and navigated to the native library:

   ```
   C:\platform tools\platform-tools\diva_extracted\lib\x86\
   ```

   Found `libdivajni.so` (5,164 bytes, dated 2015-12-31).

3. Attempted to use `strings` and `cat` on Windows CMD — both unavailable. Switched to **PowerShell** and used `Select-String`:

   ```powershell
   Select-String -Path "libdivajni.so" -Pattern "olsd"
   ```

4. Found the string `olsdfgad` embedded in the binary data of the `.so` file.

5. Also opened `libdivajni.so` in a text editor, confirming the presence of the string `olsdfgad` among the binary symbols, along with JNI function names:
   - `Java_jakhar_aseem_diva_DivaJni_access`
   - `Java_jakhar_aseem_diva_DivaJni_initiateLaunchSequence`

### Findings

- The native library `libdivajni.so` contains the **hardcoded key `olsdfgad`** in its binary data.
- The JNI function `DivaJni_access` is used by `Hardcode2Activity` for key validation.
- While native libraries are harder to reverse-engineer than Java code, they are far from secure — tools like `strings`, `Ghidra`, or `IDA Pro` can extract embedded secrets.

### Remediation

- Do not embed secrets in native libraries.
- Use runtime key derivation or server-side validation.
- If obfuscation is used, combine it with anti-tampering and integrity checks (though no client-side protection is foolproof).

---

## Summary of Vulnerabilities

| # | Challenge | Vulnerability Type | Severity | Key Finding |
|---|---|---|---|---|
| 1 | Insecure Logging | Information Disclosure | High | Credit card numbers logged in plaintext via `logcat` |
| 2 | Hardcoding Issues Pt 1 | Hardcoded Secrets | High | Vendor key `vendorsecretkey` in Java source |
| 3 | Insecure Data Storage Pt 1 | Plaintext Storage | High | Credentials in SharedPreferences XML |
| 4 | Insecure Data Storage Pt 2 | Plaintext Storage | High | Credentials in unencrypted SQLite DB |
| 5 | Insecure Data Storage Pt 3 | Plaintext Storage | High | Credentials in temp file (`uinfo*tmp`) |
| 6 | Insecure Data Storage Pt 4 | Plaintext Storage | Critical | Credentials on external storage (SD card) |
| 7 | Input Validation Pt 1 | SQL Injection | Critical | `rawQuery()` with string concatenation dumps all records |
| 8 | Input Validation Pt 2 | Local File Inclusion | High | WebView allows `file://` URI to read local files |
| 9 | Access Control Pt 1 | Hardcoded API Keys | High | API key/credentials in Activity source |
| 10 | Access Control Pt 2 | Broken Access Control | High | PIN bypass + hardcoded Tveeter API credentials |
| 11 | Hardcoding Issues Pt 2 | Hardcoded Secrets (Native) | Medium | Key `olsdfgad` embedded in `libdivajni.so` |

---

## Tools & Environment

| Tool | Purpose |
|---|---|
| **Genymotion** | Android emulator (vbox86p, IP: 192.168.98.101:5555) |
| **ADB** | Device shell, file pull, logcat |
| **JADX-GUI** | APK decompilation and source code analysis |
| **SQLite3** | Database inspection |
| **PowerShell** | Binary string search (`Select-String`) |

---

*This writeup documents challenges completed in the DIVA (Damn Insecure and Vulnerable App) for educational and authorized security research purposes.*
