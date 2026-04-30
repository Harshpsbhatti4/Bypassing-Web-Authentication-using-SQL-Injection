# Bypassing Web Authentication & Automated Data Exfiltration via SQL Injection

## Executive Summary
This project is a practical demonstration of **SQL Injection (SQLi)** vulnerabilities and their impact on web application security. I successfully progressed from manual authentication bypass techniques to professional-grade automated exploitation. Using a local lab environment (**XAMPP & DVWA**), I bypassed security controls, identified database metadata, and performed a full credential dump using **sqlmap**.


## Laboratory Setup
* **Host Environment:** Windows 10/11
* **Web Server Stack:** XAMPP (Apache 2.4.58 & MariaDB 10.4.32)
* **Target Application:** Damn Vulnerable Web Application (DVWA)
* **Security Level:** Manually forced to **Low**
* **Automation Tool:** sqlmap v1.10.4

## Phase 1: Manual Vulnerability Discovery

### 1. Error-Based Confirmation
By injecting a single quote (`'`) into the input fields, I successfully triggered a MariaDB syntax error. This confirmed that the application was directly passing raw input into the SQL engine without sanitization.

### 2. The Authentication Bypass (Comment Attack)
I bypassed the `login.php` authentication gate without a valid password by manipulating the backend SQL logic.
* **Payload:** `admin' #`
* **Technical Logic:** The backend query was structured as: 
  `SELECT * FROM users WHERE user = '$username' AND password = '$password';`
  The injected `'` closes the username string, and the `#` (MySQL/MariaDB comment character) instructs the server to ignore the rest of the query (the password verification logic).
* **Result:** The application verified that the user `admin` exists and granted full administrative access.

## Phase 2: Automated Analysis & Exploitation (sqlmap)

After manual verification, I utilized **sqlmap** to perform a professional-grade vulnerability assessment and automate the data exfiltration process.

### 1. Vulnerability Identification
The automated scan identified that the `id` parameter is susceptible to four distinct SQLi techniques:
* **Boolean-based blind:** Inferring data based on True/False page responses.
* **Error-based:** Extracting data through detailed DBMS error messages.
* **Time-based blind:** Using "sleep" commands to pause server responses.
* **UNION query:** Merging results from the `users` table directly into the web page output.

### 2. System Enumeration (Banner Grabbing)
The tool successfully performed "Banner Grabbing" to identify the backend environment:
* **Web Server:** Apache 2.4.58, PHP 8.0.30
* **Backend DBMS:** MariaDB 10.4.32

### 3. Full Database Exfiltration
I executed a targeted dump command to retrieve the `dvwa.users` table.
* **Command:** `python sqlmap.py -u "[URL]" --cookie="[SESSIONID]" -D dvwa -T users --dump`
* **Result:** Successfully retrieved all usernames and MD5 password hashes.
* **Hash Cracking:** Utilized a dictionary-based attack to recover plain-text passwords:
    * `admin` : `password`
    * `gordonb` : `abc123`
    * `pablo` : `letmein`

## Source Code Analysis (Vulnerable Implementation)
The following code snippet demonstrates the lack of sanitization that makes this attack possible:
```php
$user = $_POST[ 'username' ]; // Direct POST data assignment
$pass = $_POST[ 'password' ]; // No hashing or escaping applied
$query = "SELECT * FROM `users` WHERE user='$user' AND password='$pass';";
