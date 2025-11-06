# Deploying Django with New PostgreSQL on Render (Free DB)

## Overview
This documentation covers the complete process of deploying a Django application with PostgreSQL database on Render, including troubleshooting a 500 error and creating a superuser directly in the database.

## Prerequisites
- Django project with custom user model
- Render account
- PostgreSQL database on Render
- Environment variables configured

## Step-by-Step Deployment Process

### 1. Database Setup on Render

**Create PostgreSQL Database:**
- Go to Render Dashboard → New → PostgreSQL
- Choose Free tier (expires after 90 days)
- Note: Free instances don't support shell access

**Database Connection URLs:**
- **Internal Database URL**: Use when connecting from within Render's network
- **External Database URL**: Use when connecting from outside Render

**Important**: For Django applications deployed on Render, use the **Internal Database URL** in your environment variables.

### 2. Environment Variables Configuration on Render

**In Render Dashboard → Your Web Service → Environment Variables:**

Set the following variables:
```env
CLOUDINARY_API_KEY=your_cloudinary_api_key
CLOUDINARY_API_SECRET=your_cloudinary_api_secret
CLOUDINARY_CLOUD_NAME=your_cloudinary_cloud_name
DATABASE_URL=<your-new-postgres-internal-db-url>
DEBUG=False
RECAPTCHA_PRIVATE_KEY=your_recaptcha_private_key
RECAPTCHA_PUBLIC_KEY=your_recaptcha_public_key
SECRET_KEY=your_django_secret_key
```

**Critical**: When you create a new database, remember to update the `DATABASE_URL` environment variable in your Render project settings with the new database's internal URL.

### 3. Django Configuration

**settings.py Database Configuration:**
```python
import dj_database_url
from decouple import config

DATABASES = {
    'default': dj_database_url.config(default=config('DATABASE_URL'))
}
```

### 4. Custom User Model

**models.py:**
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

### 5. Troubleshooting 500 Error

**Common Causes:**
- Incorrect database URL (use Internal URL for Render deployments)
- Missing environment variables
- Database connection issues
- Missing migrations

**Solution:**
- Verify DATABASE_URL in environment variables uses Internal Database URL
- Ensure all migrations are applied during deployment
- Check build logs for errors

### 6. Creating Superuser Without Shell Access

Since free Render instances don't provide shell access, create superuser directly in PostgreSQL:

**Step 1: Connect to PostgreSQL Externally**
```bash
psql <your-external-postgres-url>
```

**Step 2: Generate Hashed Password (Locally)**
```python
# In local Django shell
from django.contrib.auth.hashers import make_password
password = make_password("your_secure_password")
print(password)
# Example output: pbkdf2_sha256$1000000$a...
```

**Step 3: Insert Superuser Directly into Database**
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

**Step 4: Verify User Creation**
```sql
SELECT * FROM main_customuser;
```

**Expected Output:**
```
 id |                                         password                                          |          last_login       | is_superuser |       email        | username | full_name  | is_active | is_staff |          date_joined          
----+-------------------------------------------------------------------------------------------+---------------------------+--------------+--------------------+----------+------------+-----------+----------+-------------------------------
  1 | pbkdf2_sha256$1000000$aCN5ANN1LxUi8AOnI0UDaF$QqE+9wytywMksme70cxnb8VN7UouVk21FNea1W1/OEA= | 2025-11-06 13:36:46.683338+00 | t            | admin@falla237.com | admin    | Admin User | t         | t        | 2025-11-06 13:33:04.585331+00
```

### 7. Database Management Commands

**Check Database Roles:**
```sql
\du
```

**List Databases (if needed):**
```sql
\l
```

**Update User Email (if needed):**
```sql
UPDATE main_customuser
SET email = 'admin@falla237.com'
WHERE email = 'admin@example.com';
```

### 8. Deployment Checklist

- [ ] Database migrations run successfully
- [ ] Environment variables configured correctly in Render dashboard
- [ ] DATABASE_URL updated with new database internal URL
- [ ] Static files collected
- [ ] Database connected properly
- [ ] Superuser created via direct SQL insertion
- [ ] Application accessible without 500 errors

### 9. Important Notes

**Free Tier Limitations:**
- PostgreSQL database expires after 90 days
- No shell/SSH access on free instances
- Limited resources
- Consider upgrading for production use

**When Creating New Database:**
1. Create new PostgreSQL database in Render
2. Copy the **Internal Database URL** from new database settings
3. Update `DATABASE_URL` environment variable in your web service
4. Redeploy application
5. Create new superuser using the direct SQL method

**Security Considerations:**
- Keep database credentials secure
- Use environment variables in Render dashboard
- Don't commit sensitive data to version control
- Regular backups recommended
- Use strong passwords for superuser accounts

### 10. Verification Steps

**Check Deployment:**
- Visit your Render URL (e.g., https://falla237.onrender.com)
- Access Django admin at `/admin`
- Login with superuser credentials
- Verify all functionality works

**Check Database Connection:**
```sql
-- Verify user creation and permissions
SELECT email, username, is_staff, is_superuser, is_active FROM main_customuser;
```

## Common Issues and Solutions

1. **500 Error**: Check database connection and environment variables
2. **Migration Errors**: Ensure all migration files are committed and applied
3. **Static Files**: Verify WhiteNoise configuration for static files
4. **Database Connection**: Use Internal Database URL for Render deployments
5. **Superuser Login Issues**: Verify password hashing and user permissions in database

This documentation ensures you can redeploy your application successfully, handle database changes, and work within the limitations of Render's free tier services.
