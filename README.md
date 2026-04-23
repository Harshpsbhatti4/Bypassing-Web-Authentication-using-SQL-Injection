# Bypassing-Web-Authentication-using-SQL-Injection

## Executive Summary
This project is a detailed proof-of-concept (PoC) demonstrating how **SQL Injection (SQLi)** can be exploited to bypass web authentication and exfiltrate database records. Utilizing a local lab environment (**XAMPP** & **DVWA**), I navigated through standard security defenses, modified backend source code to simulate a vulnerable environment, and successfully gained administrative access using a Tautology-based attack.

## Laboratory Environment
* **Operating System:** Windows 10
* **Server Stack:** XAMPP (Apache & MariaDB)
* **Application:** Damn Vulnerable Web Application (DVWA)
* **Tools Used:** VS Code (Source Code Analysis), Google Chrome (Testing/Exploitation)

## Phase 1: Environment Configuration
To begin the testing, I configured the application to its lowest security posture:
1.  **Database Connection:** Updated `config.inc.php` to connect to the local MariaDB instance (`root` user, no password).
2.  **Security Level:** Manually edited the configuration file to force the default security level to `low`.
    ```php
    $_DVWA[ 'default_security_level' ] = 'low';
    ```
3.  **Database Initialization:** Executed the DVWA setup script to populate the `users` table.

## Phase 2: Vulnerability Research & Discovery
Initial testing confirmed that the application was communicating directly with the database without sufficient input sanitization.

### 1. Error-Based Discovery
By injecting a single quote (`'`) into the User ID field, the application returned a MariaDB syntax error. This indicated that the input was being interpreted as part of the SQL command rather than literal data.

### 2. Tautology Attack (Data Leakage)
In the SQL Injection lab, I used the payload `' OR 1=1 #`.
* **Logic:** `1=1` is a constant truth. 
* **Result:** The query returned all rows from the `users` table, exposing names and surnames for every account in the database.

## Phase 3: Authentication Bypass (The Core Exploit)

### The Challenge
Initially, the `login.php` page remained secure due to the use of `mysqli_real_escape_string()` and `md5()` hashing, which neutralized injection characters.

### The Source Code Modification
To demonstrate a vulnerable authentication flow, I modified `login.php` to remove these security layers:
* **Removed:** `mysqli_real_escape_string()` (allowed quotes to pass).
* **Removed:** `md5()` (allowed the SQL comment character `#` to remain intact).

### The Final Payload
**Username:** `admin' #`  
**Password:** (Left Blank)

**Backend Query Transformation:**
* **Intended:** `SELECT * FROM users WHERE user='admin' AND password='...'`
* **Injected:** `SELECT * FROM users WHERE user='admin' #' AND password='...'`

The `#` symbol commented out the password verification logic. The database engine only checked for the existence of the `admin` user, granting immediate access to the dashboard.

## Remediation & Best Practices
To secure an application against these attacks, developers must move away from simple input sanitization and implement **Prepared Statements**.

### Secure Coding Example (PHP PDO):
```php
// The database treats input strictly as data, never as code.
$stmt = $pdo->prepare('SELECT * FROM users WHERE user = :username');
$stmt->execute(['username' => $user_input]);
$user = $stmt->fetch();
