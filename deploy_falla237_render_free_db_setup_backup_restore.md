
# Falla237 Deployment & PostgreSQL Backup/Restore on Render (Free DB) 

This documentation provides a complete guide for developers to deploy a Django application using a free PostgreSQL database on Render. It covers handling free-tier limitations, creating superusers without shell access, and safely backing up and restoring the database.

The guide is divided into two main parts: **Deployment** and **Backup & Restore**, with step-by-step instructions, code examples, and troubleshooting tips.

---

# Table of Contents

## Part 1: Deployment
- [Deploying Django with New PostgreSQL on Render (Free DB)](#deploying-django-with-new-postgresql-on-render-free-db)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Step-by-Step Deployment Process](#step-by-step-deployment-process)
  - [Environment Variables Configuration on Render](#environment-variables-configuration-on-render)
  - [Django Configuration](#django-configuration)
  - [Custom User Model](#custom-user-model)
  - [Troubleshooting 500 Error](#troubleshooting-500-error)
  - [Creating Superuser Without Shell Access](#creating-superuser-without-shell-access)
  - [Database Management Commands](#database-management-commands)
  - [Deployment Checklist](#deployment-checklist)
  - [Important Notes](#important-notes)
  - [Verification Steps](#verification-steps)
  - [Common Issues and Solutions](#common-issues-and-solutions)

## Part 2: Backup & Restore
- [Complete Database Backup & Restore Guide for Django on Render](#complete-database-backup--restore-guide-for-django-on-render)
  - [Overview](#overview-1)
  - [Database URL Types](#database-url-types)
  - [Backup Process](#backup-process)
  - [Restore Process](#restore-process)
  - [PostgreSQL Version Management](#postgresql-version-management)
  - [Troubleshooting](#troubleshooting)
  - [Automated Scripts](#automated-scripts)
  - [Data Preservation Checklist](#data-preservation-checklist)
  - [Best Practices](#best-practices)
  - [Pro Tips](#pro-tips)
  - [Emergency Recovery](#emergency-recovery)
  - [Quick Reference Commands](#quick-reference-commands)

---

# Deploying Django with New PostgreSQL on Render (Free DB)

## Overview
This documentation covers the complete process of deploying a Django application with PostgreSQL database on Render, including troubleshooting a 500 error and creating a superuser directly in the database.

## Prerequisites
Django project with custom user model
Render account
PostgreSQL database on Render
Environment variables configured

## Step-by-Step Deployment Process

### 1. Database Setup on Render
Create PostgreSQL Database:

Go to Render Dashboard → New → PostgreSQL
Choose Free tier (expires after 30 days)
Note: Free instances don't support shell access
Database Connection URLs:

Internal Database URL: Use when connecting from within Render's network
External Database URL: Use when connecting from outside Render
Important: For Django applications deployed on Render, use the Internal Database URL in your environment variables.

## Environment Variables Configuration on Render
In Render Dashboard → Your Web Service → Environment Variables:

Set the following variables:

```
CLOUDINARY_API_KEY=your_cloudinary_api_key
CLOUDINARY_API_SECRET=your_cloudinary_api_secret
CLOUDINARY_CLOUD_NAME=your_cloudinary_cloud_name
DATABASE_URL=<your-new-postgres-internal-db-url>
DEBUG=False
RECAPTCHA_PRIVATE_KEY=your_recaptcha_private_key
RECAPTCHA_PUBLIC_KEY=your_recaptcha_public_key
SECRET_KEY=your_django_secret_key
```

Critical: When you create a new database, remember to update the DATABASE_URL environment variable in your Render project settings with the new database's internal URL.

## Django Configuration
settings.py Database Configuration:

```python
import dj_database_url
from decouple import config

DATABASES = {
    'default': dj_database_url.config(default=config('DATABASE_URL'))
}
```

## Custom User Model
models.py:

```python
from django.contrib.auth.models import AbstractBaseUser, PermissionsMixin, BaseUserManager
from django.db import models
from django.utils import timezone

class CustomUserManager(BaseUserManager):
    def create_user(self, email, username, full_name, password=None):
        if not email:
            raise ValueError("Users must have an email address")
        if not username:
            raise ValueError("Users must have a username")
        if not full_name:
            raise ValueError("Users must provide full name")

        email = self.normalize_email(email)
        user = self.model(email=email, username=username, full_name=full_name)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, username, full_name, password):
        user = self.create_user(email, username, full_name, password)
        user.is_staff = True
        user.is_superuser = True
        user.save(using=self._db)
        return user

class CustomUser(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    username = models.CharField(max_length=150)
    full_name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    date_joined = models.DateTimeField(default=timezone.now)

    objects = CustomUserManager()

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username', 'full_name']

    def __str__(self):
        return self.email
```

## Troubleshooting 500 Error
Common Causes:

Incorrect database URL (use Internal URL for Render deployments)
Missing environment variables
Database connection issues
Missing migrations
Solution:

Verify DATABASE_URL in environment variables uses Internal Database URL
Ensure all migrations are applied during deployment
Check build logs for errors

## Creating Superuser Without Shell Access
Since free Render instances don't provide shell access, create superuser directly in PostgreSQL:

Step 1: Connect to PostgreSQL Externally

```
psql <your-external-postgres-url>
```

Step 2: Generate Hashed Password (Locally)

```python
# In local Django shell
from django.contrib.auth.hashers import make_password
password = make_password("your_secure_password")
print(password)
# Example output: pbkdf2_sha256$1000000$a...
```

Step 3: Insert Superuser Directly into Database

```sql
INSERT INTO main_customuser 
(email, username, full_name, password, is_staff, is_superuser, is_active, date_joined)
VALUES
(
  '<your-email>',
  'admin',
  'Admin User',
  '<your-hashed-password-generated-above> e.g pbkdf2_sha256$1000000$aC...',
  true,
  true,
  true,
  now()
);
```

Step 4: Verify User Creation

```sql
SELECT * FROM main_customuser;
```

Expected Output:

```
 id |                                         password                                          |          last_login       | is_superuser |       email        | username | full_name  | is_active | is_staff |          date_joined          
----+-------------------------------------------------------------------------------------------+---------------------------+--------------+--------------------+----------+------------+-----------+----------+-------------------------------
  1 | pbkdf2_sha256$1000000$aCN5ANN1LxUi8AOnI0UDaF$QqE+9wytywMksme70cxnb8VN7UouVk21FNea1W1/OEA= | 2025-11-06 13:36:46.683338+00 | t            | admin@falla237.com | admin    | Admin User | t         | t        | 2025-11-06 13:33:04.585331+00
```

## Database Management Commands
Check Database Roles:

```
\du
```

List Databases (if needed):

```
\l
```

Update User Email (if needed):

```sql
UPDATE main_customuser
SET email = 'admin@falla237.com'
WHERE email = 'admin@example.com';
```

## Deployment Checklist
- [ ] Database migrations run successfully
- [ ] Environment variables configured correctly in Render dashboard
- [ ] DATABASE_URL updated with new database internal URL
- [ ] Static files collected
- [ ] Database connected properly
- [ ] Superuser created via direct SQL insertion
- [ ] Application accessible without 500 errors

## Important Notes
Free Tier Limitations:

PostgreSQL database expires after 30 days
No shell/SSH access on free instances
Limited resources
Consider upgrading for production use
When Creating New Database:

Create new PostgreSQL database in Render
Copy the Internal Database URL from new database settings
Update DATABASE_URL environment variable in your web service
Redeploy application
Create new superuser using the direct SQL method
Security Considerations:

Keep database credentials secure
Use environment variables in Render dashboard
Don't commit sensitive data to version control
Regular backups recommended
Use strong passwords for superuser accounts

## Verification Steps
Check Deployment:

Visit your Render URL (e.g., https://falla237.onrender.com)
Access Django admin at /admin
Login with superuser credentials
Verify all functionality works
Check Database Connection:

```sql
-- Verify user creation and permissions
SELECT email, username, is_staff, is_superuser, is_active FROM main_customuser;
```

## Common Issues and Solutions
500 Error: Check database connection and environment variables
Migration Errors: Ensure all migration files are committed and applied
Static Files: Verify WhiteNoise configuration for static files
Database Connection: Use Internal Database URL for Render deployments
Superuser Login Issues: Verify password hashing and user permissions in database

---

# Complete Database Backup & Restore Guide for Django on Render

Documentation for Falla237 PostgreSQL Database Management

## Overview

### The Problem
Render free PostgreSQL databases expire after 30 days
Need to migrate data to new database without data loss
Version mismatches between local and server PostgreSQL

### The Solution
Professional backup/restore process using PostgreSQL native tools.

## Database URL Types

### External Database URL
Purpose: Connect from OUTSIDE Render's network
Used For: Backup, Restore, Direct Database Access
Format: postgresql://user:pass@external-host.render.com:5432/dbname
Example: postgresql://falla237_db_user:pass@dpg-d4698i4hg0os73ee4vig-a.oregon-postgres.render.com:5432/falla237_db

### Internal Database URL
Purpose: Connect from WITHIN Render's network
Used For: Django Application Database Connection
Format: postgresql://user:pass@internal-host.render.com:5432/dbname
Example: postgresql://falla237_db_user:pass@dpg-d4698i4hg0os73ee4vig-a:5432/falla237_db

### Important Distinction
| Operation | URL Type to Use |
|-----------|-----------------|
| Backup | ✅ External URL |
| Restore | ✅ External URL |
| Django App | ✅ Internal URL |
| Direct SQL Access | ✅ External URL |

## Backup Process

### Prerequisites
PostgreSQL client tools installed
External Database URL from Render
WSL/Ubuntu environment recommended

### Step 1: Install PostgreSQL Client (if needed)
```bash
# For Ubuntu/WSL
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt update
sudo apt install postgresql-18 -y

# Verify installation
pg_dump --version
pg_restore --version
```

### Step 2: Create Backup (Using EXTERNAL URL)
```bash
# ✅ Using EXTERNAL Database URL with custom format (recommended)
pg_dump -Fc --no-acl --no-owner "postgresql://USER:PASS@EXTERNAL-HOST:5432/DB_NAME" > falla237_backup_YYYYMM.dump

# ✅ Example with actual EXTERNAL URL
pg_dump -Fc --no-acl --no-owner "postgresql://test_db:fsjdaggdfkagdkfagifa@dpg-gidugakjfdbkagdfka-etc" > falla237_backup_Nov2025.dump
```

### Step 3: Verify Backup
```bash
ls -lh falla237_backup_Nov2025.dump
# Expected: -rw-r--r-- 1 user user 69K Nov  8 14:07 falla237_backup_Nov2025.dump
```

### Step 4: Organize Backup Files
```bash
# Create backup directory
mkdir ~/backup

# Move backup file
mv falla237_backup_Nov2025.dump ~/backup/

# Optional: Copy to Windows for extra safety
cp ~/backup/falla237_backup_Nov2025.dump /mnt/c/Users/YourName/Desktop/
```

## Restore Process

### When to Restore
Current database approaching 30-day expiration
After creating new PostgreSQL database on Render

### Step 1: Create New Database on Render
Go to Render Dashboard → New → PostgreSQL
Choose Free tier
Copy NEW External Database URL and NEW Internal Database URL

### Step 2: Restore Backup (Using NEW EXTERNAL URL)
```bash
# Navigate to backup directory
cd ~/backup/

# ✅ Restore to new database using NEW EXTERNAL URL
pg_restore --clean --no-acl --no-owner "postgresql://NEW_USER:NEW_PASS@NEW-EXTERNAL-HOST:5432/NEW_DB" falla237_backup_Nov2025.dump

# ✅ Example with new EXTERNAL URL
pg_restore --clean --no-acl --no-owner "postgresql://falla237_v2_user:abc123@dpg-new-external-host.render.com:5432/falla237_v2_db" falla237_backup_Nov2025.dump
```

### Step 3: Update Django Settings (Using NEW INTERNAL URL)
Go to Render Dashboard → Your Web Service → Environment Variables
Update DATABASE_URL with NEW INTERNAL Database URL
Redeploy application

### Step 4: Verify Restore
```bash
# Connect to new database using EXTERNAL URL
psql "postgresql://NEW_USER:NEW_PASS@NEW-EXTERNAL-HOST:5432/NEW_DB"

# Check tables
\dt

# Count items (verify data transferred)
SELECT COUNT(*) FROM main_item;
SELECT COUNT(*) FROM main_customuser;
```

## PostgreSQL Version Management

### Version Mismatch Issue
```bash
# Error example:
pg_dump: error: server version: 17.6; pg_dump version: 14.19
pg_dump: error: aborting because of server version mismatch
```

### Solution: Install Latest PostgreSQL
```bash
# Add official PostgreSQL repository
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import signing key
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Install PostgreSQL 18
sudo apt update
sudo apt install postgresql-18 -y
```

### Alternative: Use Automated Script
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install postgresql-18 -y
```

## Troubleshooting

### Common Issues & Solutions

#### 1. Connection Refused During Backup
```bash
# ❌ Wrong: Using Internal URL for backup
pg_dump "postgresql://user@internal-host:5432/db"  # Fails

# ✅ Correct: Use External URL for backup
pg_dump "postgresql://user@external-host.render.com:5432/db"  # Works
```

#### 2. Permission Denied
Verify username/password in database URL
Check if user has necessary permissions

#### 3. Restore Fails
```bash
# Try without --clean flag
pg_restore --no-acl --no-owner "postgresql://NEW_USER:NEW_PASS@NEW-EXTERNAL-HOST:5432/NEW_DB" falla237_backup_Nov2025.dump

# Or use SQL format as fallback
psql "postgresql://NEW_USER:NEW_PASS@NEW-EXTERNAL-HOST:5432/NEW_DB" < falla237_backup.sql
```

#### 4. Django Connection Fails After Restore
Ensure you're using INTERNAL URL in Django environment variables
Check Render web service environment variables
Verify database migrations are applied

## Automated Scripts

### Backup Script (backup_database.py)
```python
#!/usr/bin/env python3
import os
import subprocess
from datetime import datetime
import argparse

def backup_database(external_db_url, backup_dir="~/backup"):
    """Create database backup using EXTERNAL database URL"""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_file = f"{backup_dir}/falla237_backup_{timestamp}.dump"
    
    cmd = [
        'pg_dump', '-Fc', '--no-acl', '--no-owner',
        external_db_url, '-f', backup_file
    ]
    
    try:
        subprocess.run(cmd, check=True)
        print(f"✅ Backup created: {backup_file}")
        return backup_file
    except subprocess.CalledProcessError as e:
        print(f"❌ Backup failed: {e}")
        return None

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument('--external-db-url', required=True, help='EXTERNAL Database URL')
    args = parser.parse_args()
    
    backup_database(args.external_db_url)
```

Usage:
```bash
python backup_database.py --external-db-url "postgresql://user:pass@external-host.render.com:5432/dbname"
```

## Data Preservation Checklist

### Before Backup:
- [ ] Get EXTERNAL Database URL from Render
- [ ] Verify all important data is in database
- [ ] Test application functionality
- [ ] Note down any custom configurations

### After Restore:
- [ ] Get NEW EXTERNAL URL and NEW INTERNAL URL from new database
- [ ] Verify item count matches using EXTERNAL URL
- [ ] Update Django environment variables with NEW INTERNAL URL
- [ ] Test user login functionality
- [ ] Check all application features work
- [ ] Test on live application

### Files to Keep Safe:
- falla237_backup_YYYYMM.dump - Database backup
- This documentation
- Environment variables list

## Best Practices

### Regular Backups
```bash
# Monthly backup using EXTERNAL URL (recommended)
0 2 1 * * pg_dump -Fc --no-acl --no-owner "EXTERNAL_DB_URL" > ~/backup/falla237_$(date +\%Y\%m).dump
```

### Multiple Storage Locations
- Local WSL directory
- Windows desktop
- Cloud storage (Google Drive, Dropbox)
- GitHub (private repository)

### URL Management
- Backup/Restore: Always use EXTERNAL URLs
- Django App: Always use INTERNAL URLs
- Document both URLs for each database

## Pro Tips
- Test Restore Process before database expiration using EXTERNAL URL
- Keep multiple backup versions in case of corruption
- Automate backups with cron jobs for production
- Monitor database size to anticipate storage needs
- Document any custom database configurations

## Emergency Recovery
If Backup Fails:
- Use Django dumpdata as alternative:
  ```bash
  python manage.py dumpdata > fallback_data.json
  ```
- Contact Render support for extended database access
- Consider upgrading to paid plan for permanent database

## Quick Reference Commands
Backup (Using EXTERNAL URL):
```bash
pg_dump -Fc --no-acl --no-owner "EXTERNAL_DB_URL" > backup.dump
```

Restore (Using NEW EXTERNAL URL):
```bash
pg_restore --clean --no-acl --no-owner "NEW_EXTERNAL_DB_URL" backup.dump
```

Update Django (Using NEW INTERNAL URL):
Update DATABASE_URL environment variable in Render dashboard

This documentation ensures you can safely migrate your Django application between Render PostgreSQL databases without data loss. Regular backups protect against free tier expiration and accidental data loss.

🔗 Useful Links
PostgreSQL Documentation
Render PostgreSQL Guide
Django Database Documentation

🎯 Key Takeaway: EXTERNAL URL for backup/restore, INTERNAL URL for Django application
