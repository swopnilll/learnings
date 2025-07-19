## 🔐 What Is "Identity" in This Context?

In Windows-based systems, **every process runs as a specific Windows user account**.

This identity determines:

-   What **permissions** the process has
-   What **network resources** it can access
-   How it **authenticates to remote systems** like SQL Server or MSDTC
* * *

## ✅ Identity Scenarios in .NET Apps

There are 3 main deployment styles in enterprise .NET apps:

### 1\. ASP.NET / Web API App

Hosted in **IIS (Internet Information Services)**

💡 **The identity used = Application Pool Identity**

* * *

### 🔧 How does it work?

IIS hosts your web app in a process called **w3wp.exe**, and that process runs under a **Windows user account**.

Each **Application Pool** in IIS is assigned an identity. This identity is inherited by the web app.

You can configure this from:

> IIS Manager → Application Pools → YourAppPool → Advanced Settings → Identity

#### Options for Application Pool Identity:

| Option | Description |
| --- | --- |
| **ApplicationPoolIdentity** _(default)_ | A virtual identity like `IIS APPPOOL\MyAppPool` |
| **NetworkService** | Built-in low-privilege Windows account |
| **LocalSystem** | Powerful local account (not recommended) |
| **Custom account** | A specific Windows user account (e.g. `CORP\MyWebAppService`) |

* * *

### ✅ Why does this matter?

This identity is **used to authenticate** to:

-   SQL Server (when using Windows Integrated Auth)
-   Remote file shares
-   Remote MSDTC coordinators
-   Active Directory
-   Other Windows services (e.g., MSMQ)

If your app is using a **distributed transaction**, the Application Pool Identity is the identity that **MSDTC uses when coordinating with remote servers**.

* * *

### 💡 Real-life example:

```csharp
// Your ASP.NET app connects to SQL Server with Integrated Security 
var connection = new SqlConnection("Data Source=DB;Integrated Security=SSPI");`
```
-   This uses **Windows authentication**
-   SQL Server checks the identity of the process (e.g., `IIS APPPOOL\MyAppPool` or `CORP\MyWebAppService`)
-   If that identity has `db_datareader` and `db_datawriter`, the connection succeeds

✅ **Important**: If you are using `TransactionScope`, and this connection promotes to MSDTC, the same identity is used when talking to **remote MSDTC**.

* * *

## 2\. 🧾 Console App or 🛠️ Windows Service

Here, your app runs **directly as a Windows process** (not hosted by IIS).

🟢 **The identity used = The Windows account that runs the EXE or service**

* * *

### 🧾 Console App

If you're testing with a `.exe` file, it's running as **you** (the currently logged-in Windows user):

```powershell
`whoami`
```

That’s the identity used to access SQL Server, DTC, network shares, etc.

* * *

### 🛠️ Windows Service

If you're running a Windows service (e.g., background worker):

-   You **register the service** with `sc.exe` or `services.msc`
-   You **assign a Windows account** to the service

> Services → YourService → Properties → Log On

#### Typical choices:

| Option | Description |
| --- | --- |
| **LocalSystem** | Full local access, high privilege, access to network as machine |
| **NetworkService** | Low privilege, but has network access using machine account |
| **LocalService** | Least privilege, mostly local-only |
| **Custom Account** | Domain account like `CORP\MyWorkerUser` (recommended for secure auth) |

