<p align="center">
<img src="https://i.imgur.com/Clzj7Xs.png" alt="osTicket logo"/>
</p>

</head>
<body>

<header>
  <h1>osTicket Lab — Azure VM (IIS) Deployment &amp; Configuration</h1>
  <p><strong>Summary</strong><br>
    This lab documents deploying an osTicket instance on a Windows Server VM in Microsoft Azure using IIS and PHP (FastCGI).
    It covers provisioning, IIS/PHP configuration, database setup, osTicket installation, administrative configuration
    (portals, roles, departments, teams, SLAs), verification, and basic hardening.</p>
</header>

<nav aria-label="Table of contents">
  <strong>Table of contents</strong>
  <ul>
    <li>Objective</li>
    <li>Prerequisites</li>
    <li>High-level overview</li>
    <li>Step-by-step deployment &amp; configuration</li>
    <li>Verification checklist</li>
    <li>Troubleshooting tips</li>
    <li>What I learned — Challenges — Next steps</li>
    <li>Suggested commit message / PR description</li>
  </ul>
</nav>

<section id="objective">
  <h2>Objective</h2>
  <p>Deploy an Azure Windows Server VM and host <strong>osTicket</strong> using IIS + PHP. Configure the application
  and verify a full ticket lifecycle to simulate a realistic helpdesk environment suitable for inclusion in a portfolio.</p>
</section>

<section id="prerequisites">
  <h2>Prerequisites</h2>
  <ul>
    <li>Azure subscription with permissions to create VMs, public IPs, and NSGs.</li>
    <li>Familiarity with Windows Server administration, RDP, and basic networking.</li>
    <li>Basic knowledge of PHP and MySQL.</li>
    <li>osTicket package (official release recommended).</li>
    <li>RDP access to the VM.</li>
  </ul>
</section>

<section id="overview">
  <h2>High-level overview</h2>
  <ul>
    <li><strong>Platform:</strong> Azure VM (Windows Server)</li>
    <li><strong>Web stack:</strong> IIS + PHP (FastCGI / CGI)</li>
    <li><strong>Database:</strong> MySQL (local or managed)</li>
    <li><strong>App:</strong> osTicket deployed as an IIS site or virtual directory</li>
    <li><strong>Security:</strong> NSG for least-privilege access, HTTPS for web traffic, file-permission hardening after install</li>
  </ul>
</section>

<section id="steps">
  <h2>Step-by-step deployment &amp; configuration</h2>
  <p><em>The instructions assume use of the Windows GUI (Server Manager / IIS Manager). PowerShell equivalents can be added on request.</em></p>

  <h3>1. Provision the Azure VM</h3>
  <ol>
    <li>Create resource group (optional) and provision a Windows Server VM sized for lab needs.</li>
    <li>Configure NSG rules to allow:
      <ul>
        <li>RDP (TCP 3389) — restrict to your IP if possible</li>
        <li>HTTP (TCP 80) and HTTPS (TCP 443) if external access is required</li>
      </ul>
    </li>
    <li>Connect to the VM via RDP or Azure Bastion.</li>
  </ol>

<img width="783" height="1134" alt="image" src="https://github.com/user-attachments/assets/c6c3a2a0-dd49-4599-9ce2-c13cbd42afc6" />


  <h3>2. Install IIS</h3>
  <p>Open <strong>Server Manager → Add roles and features → Web Server (IIS)</strong>. Include Management Tools if you plan on remote management.</p>

  <img width="409" height="354" alt="image" src="https://github.com/user-attachments/assets/c6cf6ce3-e213-4b3b-a579-ab7dc9b72a76" />

  <h3>3. Enable CGI / FastCGI</h3>
  <p>Under Application Development, enable <strong>CGI</strong>. FastCGI will be used to run PHP efficiently under IIS.</p>

  <img width="406" height="349" alt="image" src="https://github.com/user-attachments/assets/dedc7534-952a-468c-8cce-34aceb08f07f" />

  <h3>4. Install &amp; configure PHP</h3>
  <ol>
    <li>Download a supported PHP build for Windows (Thread Safe recommended for FastCGI).</li>
    <li>Install required Microsoft Visual C++ Redistributable if prompted.</li>
    <li>(Recommended) Install <strong>PHP Manager for IIS</strong> to register PHP more easily.</li>
    <li>Register the PHP executable in IIS (via PHP Manager or FastCGI Settings).</li>
    <li>Verify PHP by creating <code>phpinfo.php</code> in the web root:
      <pre><code>&lt;?php
phpinfo();
?&gt;</code></pre>
      Browse to <code>http://&lt;vm-ip&gt;/phpinfo.php</code> to validate.
    </li>
  </ol>

<img width="540" height="506" alt="image" src="https://github.com/user-attachments/assets/db5ce6c5-cff5-4f6e-8ae1-9e60ae2f7216" />

  <h3>5. Prepare the database</h3>
  <p>Install MySQL/MariaDB on the VM or provision a managed DB instance. Create the DB and a dedicated user for osTicket:</p>
  <pre><code>CREATE DATABASE osticket CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'osticketuser'@'localhost' IDENTIFIED BY 'StrongPasswordHere!';
GRANT ALL PRIVILEGES ON osticket.* TO 'osticketuser'@'localhost';
FLUSH PRIVILEGES;</code></pre>
  <p>Harden the DB: set root password, remove anonymous users, disable remote root login, remove test DB.</p>

  <h3>6. Deploy osTicket files</h3>
  <ol>
    <li>Download official osTicket release and extract the <code>upload</code> folder.</li>
    <li>Copy <code>upload</code> contents to an IIS site folder (e.g., <code>C:\inetpub\wwwroot\osticket</code>) or create a new site.</li>
    <li>Ensure the application pool identity (e.g., <code>ApplicationPoolIdentity</code>) or <code>IIS_IUSRS</code> has the required permissions:
      <ul>
        <li>Read for web files</li>
        <li>Write for config/upload directories during install (tighten permissions afterwards)</li>
      </ul>
    </li>
  </ol>

<img width="932" height="670" alt="image" src="https://github.com/user-attachments/assets/ad046095-347e-4c17-8b20-675ef4b92334" />

  <h3>7. Configure the IIS site</h3>
  <p>Create / assign an Application Pool (recommend <code>ApplicationPoolIdentity</code>), confirm Handler Mappings include <code>.php</code> mapped to the FastCGI php-cgi executable. Optionally enable URL Rewrite for friendly URLs or redirects.</p>

  <h3>8. Run the osTicket web installer</h3>
  <ol>
    <li>Point a browser to <code>http://&lt;vm-ip-or-hostname&gt;/osticket</code>.</li>
    <li>Follow the installer steps:
      <ul>
        <li>Validate system requirements (PHP version &amp; extensions)</li>
        <li>Provide DB connection details</li>
        <li>Create the initial admin user</li>
      </ul>
    </li>
    <li>After install, create or move <code>ost-config.php</code> per instructions, then remove or secure installer files.</li>
  </ol>

<img width="1600" height="856" alt="image" src="https://github.com/user-attachments/assets/adc5c4ee-0b4c-4c25-9bc2-86241fc39454" />

  <h3>9. Administrative configuration</h3>
  <ul>
    <li><strong>End User Portal:</strong> verify public ticket submission.</li>
    <img width="836" height="479" alt="image" src="https://github.com/user-attachments/assets/37a6bb7f-062c-4452-8677-8a7f9cf492cc" />
    <li><strong>Agent Portal:</strong> verify agent login and admin console.</li>
    <li><strong>Roles:</strong> create Agent / Supervisor roles and set permissions.</li>
    <img width="808" height="1167" alt="image" src="https://github.com/user-attachments/assets/8dcb84c0-9cf8-4a68-9d56-af0fa84d9a85" />
    <li><strong>Departments:</strong> create departments and routing rules.</li>
    <img width="546" height="748" alt="image" src="https://github.com/user-attachments/assets/e43be299-d9c5-4e7f-8ca1-969aecc8d426" />
    <li><strong>Teams:</strong> create teams and assign agents.</li>
    <img width="483" height="308" alt="image" src="https://github.com/user-attachments/assets/1262723e-5ebb-44f7-b2d6-1acfb812e388" />
    <li><strong>Users:</strong> create sample end users or import test data.</li>
    <img width="648" height="391" alt="image" src="https://github.com/user-attachments/assets/733d6a8b-93a2-4cc7-b4f6-724af821e32f" />
    <li><strong>SLA Plans:</strong> create SLA plans with schedules and escalation rules.</li>
    <img width="792" height="331" alt="image" src="https://github.com/user-attachments/assets/a868d10d-6122-4d50-8b0d-2162d1754fd6" />
  </ul>

  <h3>10. Validate ticket lifecycle</h3>
  <p>Create a ticket from the End User Portal and verify:</p>
  <img width="966" height="1207" alt="image" src="https://github.com/user-attachments/assets/8f6a595b-7733-473c-a706-e2bc6bee6047" />
  <ul>
    <li>Correct department/team assignment based on rules.</li>
    <li>SLA timers start and escalate per configuration.</li>
    <li>Agent can update status, add internal notes, and resolve the ticket.</li>
    <li>Email notifications are sent if SMTP is configured correctly.</li>
  </ul>

  <h3>11. Post-install hardening & maintenance</h3>
  <ul>
    <li>Remove installer files and tighten file permissions.</li>
    <li>Configure HTTPS (install certificate; consider Let’s Encrypt via a Windows ACME client).</li>
    <li>Configure backups or take an Azure VM snapshot.</li>
    <li>Consider migrating DB to Azure Database for MySQL for resiliency.</li>
  </ul>
</section>

<section id="checklist">
  <h2>Verification checklist</h2>
  <ul class="checklist">
    <li><label><input type="checkbox" disabled> Azure VM provisioned and RDP reachable.</label></li>
    <li><label><input type="checkbox" disabled> IIS installed and running.</label></li>
    <li><label><input type="checkbox" disabled> CGI / FastCGI enabled and PHP registered.</label></li>
    <li><label><input type="checkbox" disabled> <code>phpinfo()</code> confirms PHP active with required extensions.</label></li>
    <li><label><input type="checkbox" disabled> Database created with dedicated <code>osticket</code> user.</label></li>
    <li><label><input type="checkbox" disabled> osTicket files deployed and permissions set correctly.</label></li>
    <li><label><input type="checkbox" disabled> Web installer completed; admin account created.</label></li>
    <li><label><input type="checkbox" disabled> End user and agent portals accessible.</label></li>
    <li><label><input type="checkbox" disabled> Roles, departments, teams, users, and SLA configured.</label></li>
    <li><label><input type="checkbox" disabled> Ticket lifecycle validated end-to-end.</label></li>
    <li><label><input type="checkbox" disabled> HTTPS configured and installer artifacts removed.</label></li>
    <li><label><input type="checkbox" disabled> Backup/snapshot created.</label></li>
  </ul>
</section>

<section id="troubleshooting">
  <h2>Troubleshooting tips</h2>
  <ul>
    <li><strong>Blank pages / installer fails:</strong> enable detailed errors temporarily in IIS, check <code>C:\inetpub\logs\LogFiles</code> and the PHP error log. Confirm required PHP extensions are installed.</li>
    <li><strong>PHP files served as download / not executed:</strong> verify Handler Mappings point <code>.php</code> to the correct FastCGI php-cgi executable and that FastCGI is enabled.</li>
    <li><strong>DB connection errors:</strong> confirm DB host, port, credentials, and that the DB service is running and reachable from the VM.</li>
    <li><strong>Permission errors writing config files:</strong> temporarily grant write permissions to the App Pool identity during install and remove/restrict after.</li>
    <li><strong>Emails not sending:</strong> verify SMTP credentials, server, and outbound firewall rules (ports 25/465/587).</li>
    <li><strong>Performance issues:</strong> increase VM size or move DB to managed instance for better I/O and reliability.</li>
  </ul>
</section>

<section id="learned">
  <h2>What I learned — Challenges — Next steps</h2>

  <h3>What I learned</h3>
  <ul>
    <li>How to register and run PHP under IIS (FastCGI) on Windows Server.</li>
    <li>End-to-end deployment tasks for a PHP web application on an Azure VM.</li>
    <li>Key osTicket administration tasks (portals, roles, departments, teams, SLA).</li>
  </ul>

  <h3>Challenges</h3>
  <ul>
    <li>Matching PHP version and extensions required by the osTicket release.</li>
    <li>Setting correct file permissions for IIS identities on Windows.</li>
    <li>Reliable SMTP setup from a VM (outbound port restrictions, authentication).</li>
  </ul>

</body>
</html>
