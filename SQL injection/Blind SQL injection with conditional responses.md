# Lab: Blind SQL injection with conditional responses

## PRACTITIONER

```
 This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs an SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and no error messages are displayed. But the application includes a "Welcome back" message in the page if the query returns any rows.

The database contains a different table called users, with columns called username and password. You need to exploit the blind SQL injection vulnerability to find out the password of the administrator user.

To solve the lab, log in as the administrator user. 
```

### Solution

1. Visit the front page of the shop, and use Burp Suite to intercept and modify the request containing the <code>TrackingId</code> cookie. For simplicity, let's say the original value of the cookie is <code>TrackingId=xyz</code>.

2. Modify the <code>TrackingId</code> cookie, changing it to: <code>TrackingId=xyz' AND '1'='1</code>. Verify that the "Welcome back" message appears in the response.

3. Now change it to: <code>TrackingId=xyz' AND '1'='2</code>. Verify that the "Welcome back" message does not appear in the response. This demonstrates how you can test a single boolean condition and infer the result.

4. Now change it to: <code>TrackingId=xyz' AND (SELECT 'a' FROM users LIMIT 1)='a</code>. Verify that the condition is true, confirming that there is a table called <code>users</code>.

5. Now change it to: <code>TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a</code>. Verify that the condition is true, confirming that there is a user called <code>administrator</code>.

6. The next step is to determine how many characters are in the password of the <code>administrator</code> user. To do this, change the value to: <code>TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)&gt;1)='a</code>. This condition should be true, confirming that the password is greater than 1 character in length.

7. Send a series of follow-up values to test different password lengths. Send: <code>TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)&gt;2)='a</code>. Then send: <code>TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)&gt;3)='a</code>. And so on. You can do this manually using <a href="/burp/documentation/desktop/tools/repeater">Burp Repeater</a>, since the length is likely to be short. When the condition stops being true (i.e. when the "Welcome back" message disappears), you have determined the length of the password, which is in fact 20 characters long.

8. After determining the length of the password, the next step is to test the character at each position to determine its value. This involves a much larger number of requests, so you need to use <a href="/burp/documentation/desktop/tools/intruder">Burp Intruder</a>. Send the request you are working on to Burp Intruder, using the context menu.

9. In the Positions tab of Burp Intruder, clear the default payload positions by clicking the "Clear §" button.

10. In the Positions tab, change the value of the cookie to: <code>TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a</code>. This uses the <code>SUBSTRING()</code> function to extract a single character from the password, and test it against a specific value. Our attack will cycle through each position and possible value, testing each one in turn.

11. Place payload position markers around the final <code>a</code> character in the cookie value. To do this, select just the <code>a</code>, and click the "Add §" button. You should then see the following as the cookie value (note the payload position markers): <code>TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§</code>

12. To test the character at each position, you'll need to send suitable payloads in the payload position that you've defined. You can assume that the password contains only lowercase alphanumeric characters. Go to the Payloads tab, check that "Simple list" is selected, and under "Payload Options" add the payloads in the range a - z and 0 - 9. You can select these easily using the "Add from list" drop-down.

13. To be able to tell when the correct character was submitted, you'll need to grep each response for the expression "Welcome back". To do this, go to the Options tab, and the "Grep - Match" section. Clear any existing entries in the list, and then add the value "Welcome back".

14. Launch the attack by clicking the "Start attack" button or selecting "Start attack" from the Intruder menu.

15. Review the attack results to find the value of the character at the first position. You should see a column in the results called "Welcome back". One of the rows should have a tick in this column. The payload showing for that row is the value of the character at the first position.

16. Now, you simply need to re-run the attack for each of the other character positions in the password, to determine their value. To do this, go back to the main Burp window, and the Positions tab of Burp Intruder, and change the specified offset from 1 to 2. You should then see the following as the cookie value: <code>TrackingId=xyz' AND (SELECT SUBSTRING(password,2,1) FROM users WHERE username='administrator')='a</code>

17. Launch the modified attack, review the results, and note the character at the second offset.

18. Continue this process testing offset 3, 4, and so on, until you have the whole password.

19. In your browser, click "My account" to open the login page. Use the password to log in as the <code>administrator</code> user.
