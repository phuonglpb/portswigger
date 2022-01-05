# Lab: SQL injection UNION attack, retrieving multiple values in a single column

### PRACTITIONER

This lab contains an SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The database contains a different table called users, with columns called username and password.

To solve the lab, perform an SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the administrator user.  

```
Hint
```
You can find some useful payloads on our [SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet).

### Solution

1. Use Burp Suite to intercept and modify the request that sets the product category filter.

2. Determine the <a href="/web-security/sql-injection/union-attacks/lab-determine-number-of-columns">number of columns that are being returned by the query</a> and <a href="/web-security/sql-injection/union-attacks/lab-find-column-containing-text">which columns contain text data</a>. Verify that the query is returning two columns, only one of which contain text, using a payload like the following in the <code>category</code> parameter: <code>'+UNION+SELECT+NULL,'abc'--</code>

3. Use the following payload to retrieve the contents of the <code>users</code> table: <code>'+UNION+SELECT+NULL,username||'~'||password+FROM+users--</code>

4. Verify that the application's response contains usernames and passwords.
