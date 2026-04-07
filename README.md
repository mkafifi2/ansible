| Rank | Module | Category | Primary Use Case |
| :--- | :--- | :--- | :--- |
| 1 | **file** | Files | Manage permissions, ownership, and state (directory/file/link) |
| 2 | **copy** | Files | Copy files from local machine to remote nodes |
| 3 | **template** | Files | Deploy dynamic config files using Jinja2 templates |
| 4 | **service** | System | Start, stop, restart or enable/disable OS services |
| 5 | **command** | Commands | Execute simple commands on remote nodes |
| 6 | **shell** | Commands | Execute commands with shell features (pipes, redirects) |
| 7 | **set_fact** | Logic | Create or update variables dynamically during a play |
| 8 | **apt** | Packages | Manage packages on Debian/Ubuntu systems |
| 9 | **lineinfile** | Files | Ensure a specific single line exists in a file |
| 10 | **copy** | Files | Transfer static files to remote system |
| 11 | **yum** | Packages | Manage packages on RHEL/CentOS systems |
| 12 | **stat** | Files | Retrieve status/metadata of a file (existence, checksum) |
| 13 | **debug** | Utility | Print variables or messages for troubleshooting |
| 14 | **get_url** | Network | Download files from HTTP, HTTPS, or FTP |
| 15 | **user** | User | Manage user accounts (UID, shell, groups) |
| 16 | **group** | User | Manage user groups |
| 17 | **unarchive** | Files | Unpack archives (.zip, .tar.gz) on remote host |
| 18 | **git** | Source | Clone or update Git repositories |
| 19 | **uri** | Network | Interact with HTTP/HTTPS APIs (REST calls) |
| 20 | **package** | Packages | OS-agnostic package manager (calls apt, yum, etc.) |
| 21 | **assert** | Logic | Fail the play if specific conditions aren't met |
| 22 | **include_tasks** | Logic | Dynamically include a task file |
| 23 | **import_tasks** | Logic | Statically import a task file at parse-time |
| 24 | **systemd** | System | Manage systemd services specifically (daemon-reload) |
| 25 | **pip** | Packages | Manage Python library dependencies |
| 26 | **cron** | System | Manage crontab entries for scheduled tasks |
| 27 | **mount** | System | Configure and manage disk mount points in fstab |
| 28 | **sysctl** | System | Manage kernel parameters (e.g., ip_forward) |
| 29 | **wait_for** | Network | Wait for a port, file, or string before continuing |
| 30 | **blockinfile** | Files | Insert or update a multi-line block of text |
| 31 | **dnf** | Packages | Package manager for modern RHEL/Fedora/CentOS 8+ |
| 32 | **hostname** | System | Set or update the system hostname |
| 33 | **apt_repository**| Packages | Add/remove APT repos (e.g., PPAs) |
| 34 | **apt_key** | Packages | Add/remove GPG keys for APT verification |
| 35 | **yum_repository** | Packages | Add/remove YUM repository files |
| 36 | **authorized_key**| Security | Add/remove SSH public keys for users |
| 37 | **ping** | Network | Verify connectivity and Python availability |
| 38 | **setup** | Utility | Gather hardware and OS facts (auto-gathered usually) |
| 39 | **raw** | Commands | Execute SSH commands without requiring Python |
| 40 | **script** | Commands | Run a local script on the remote node |
| 41 | **fetch** | Files | Pull files from remote node to local controller |
| 42 | **find** | Files | Search for files based on patterns/age/size |
| 43 | **replace** | Files | Regex-based find and replace within a file |
| 44 | **archive** | Files | Compress files into a .tgz, .zip, or .bz2 archive |
| 45 | **fail** | Logic | Stop execution with a custom error message |
| 46 | **pause** | Logic | Set a timer or wait for manual user input |
| 47 | **slurp** | Files | Read base64 content of a remote file |
| 48 | **tempfile** | Files | Create temporary files or directories |
| 49 | **iptables** | Security | Manage Linux firewall rules |
| 50 | **firewalld** | Security | Manage RHEL/CentOS firewalld rules |
| 51 | **ufw** | Security | Manage Ubuntu "Uncomplicated Firewall" |
| 52 | **mysql_user** | Database | Create/Manage MySQL/MariaDB users |
| 53 | **mysql_db** | Database | Create/Manage MySQL/MariaDB databases |
| 54 | **postgresql_user**| Database | Create/Manage PostgreSQL users |
| 55 | **postgresql_db**| Database | Create/Manage PostgreSQL databases |
| 56 | **rpm_key** | Packages | Import/Remove GPG keys from RPM DB |
| 57 | **selinux** | Security | Set SELinux global state (Enforcing/Permissive) |
| 58 | **seboolean** | Security | Toggle specific SELinux boolean flags |
| 59 | **known_hosts** | Security | Add/Remove hosts from SSH known_hosts file |
| 60 | **modprobe** | System | Load or unload Linux kernel modules |
| 61 | **timezone** | System | Configure system timezone |
| 62 | **docker_container**| Containers | Start, stop, or remove Docker containers |
| 63 | **docker_image** | Containers | Pull or build Docker images |
| 64 | **k8s** | Containers | Manage Kubernetes clusters and resources |
| 65 | **win_shell** | Windows | Run PowerShell commands on Windows |
| 66 | **win_command** | Windows | Run basic commands on Windows |
| 67 | **win_package** | Windows | Install .msi or .exe installers on Windows |
| 68 | **win_service** | Windows | Manage Windows services |
| 69 | **win_file** | Windows | Manage files/directories on Windows |
| 70 | **win_copy** | Windows | Copy files to Windows hosts |
| 71 | **win_user** | Windows | Manage local Windows user accounts |
| 72 | **ec2_instance** | Cloud/AWS | Manage AWS EC2 virtual machines |
| 73 | **s3_bucket** | Cloud/AWS | Create or delete AWS S3 buckets |
| 74 | **azure_rm_vm** | Cloud/Azure | Manage Azure virtual machines |
| 75 | **homebrew** | Packages | Manage macOS packages via Homebrew |
| 76 | **npm** | Packages | Manage Node.js packages |
| 77 | **gem** | Packages | Manage RubyGems |
| 78 | **alternatives** | System | Manage system "alternatives" (symlink versions) |
| 79 | **assemble** | Files | Concatenate file fragments into one file |
| 80 | **add_host** | Inventory | Add a host to inventory during runtime |
| 81 | **group_by** | Inventory | Create dynamic groups based on facts |
| 82 | **meta** | Logic | Meta-actions (flush_handlers, refresh_inventory) |
| 83 | **snmp_facts** | Utility | Fetch hardware info via SNMP |
| 84 | **debconf** | Packages | Set Debian package configurations |
| 85 | **patch** | Files | Apply standard patch files to remote files |
| 86 | **xml** | Files | Edit XML file elements and attributes |
| 87 | **ini_file** | Files | Manage settings in standard .ini files |
| 88 | **htpasswd** | Security | Manage web server password files |
| 89 | **java_cert** | Security | Manage certificates in Java keystores |
| 90 | **pamd** | Security | Manage PAM (Pluggable Auth Modules) rules |
| 91 | **django_manage** | Web App | Run Django management commands |
| 92 | **composer** | Packages | Manage PHP dependencies |
| 93 | **make** | Development | Run Makefiles and targets |
| 94 | **supervisorctl** | System | Manage processes via Supervisord |
| 95 | **zypper** | Packages | Manage packages on SLES/OpenSUSE |
| 96 | **pacman** | Packages | Manage packages on Arch Linux |
| 97 | **apk** | Packages | Manage packages on Alpine Linux |
| 98 | **slack** | Utility | Send notification messages to Slack |
| 99 | **mail** | Utility | Send automated emails |
| 100 | **synchronize** | Files | Efficiently sync directories using rsync |
