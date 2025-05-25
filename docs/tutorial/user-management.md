---
sidebar_position: 1
---

# User Management

This guide covers how to manage users in Mini Gateway.

## User Types

Mini Gateway supports different types of users:
- **Administrators**: Full system access
- **Operators**: Can manage proxies and basic settings
- **Users**: Basic access to assigned resources

## Creating Users

### Using the Web Interface

1. Log in to the admin panel
2. Navigate to Users > Add User
3. Fill in the required information:
   - Username
   - Password
   - Email
   - Role
4. Click "Create User"

### Using the CLI

```bash
mini-gateway user create --username john --password secret --role operator --email john@example.com
```

## Managing User Permissions

1. Access the user's profile
2. Select the "Permissions" tab
3. Configure access levels for:
   - Proxy management
   - Gateway settings
   - User management
   - System monitoring

## User Authentication

### Password Policies

- Minimum 8 characters
- Must include numbers and special characters
- Password expiration: 90 days
- Failed login attempts: 5 before lockout

### Two-Factor Authentication

1. Enable 2FA in user settings
2. Scan QR code with authenticator app
3. Enter verification code
4. Save settings

## User Groups

Create groups to manage permissions efficiently:

1. Navigate to Groups > Create Group
2. Set group name and description
3. Assign permissions
4. Add users to the group

## Best Practices

- Regularly audit user access
- Implement least privilege principle
- Use strong password policies
- Enable 2FA for sensitive accounts
- Document all user management actions 