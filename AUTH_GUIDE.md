# VenueIQ Authentication & User Management Guide

## Overview

VenueIQ now features full Supabase authentication integration with persistent sessions, role-based access control, and user profile management.

---

## Authentication Features

### 1. Sign Up / Registration
- **Email-based registration** with password
- Optional full name capture
- User metadata stored in Supabase auth
- Automatic account creation

**Flow**:
```
User clicks "Create Account" 
→ Enters email, password, full name
→ Supabase creates auth user
→ Session automatically established
→ Redirected to app dashboard
```

### 2. Sign In / Login
- Standard email/password authentication
- Persistent sessions (cookies)
- Auto-refresh tokens
- Session recovery on app restart

**Flow**:
```
User clicks "Sign In"
→ Enters email and password
→ Supabase validates credentials
→ Session token issued
→ Automatic login on return visit
```

### 3. Session Management
- **Persistent Sessions**: User stays logged in across browser restarts
- **Auto Token Refresh**: Tokens automatically refreshed before expiry
- **Session Detection**: App detects existing session on load
- **Sign Out**: Clears session and redirects to login

### 4. User Profile Management
- **Profile Viewing**: Display in header with email and full name
- **Profile Editing**: Modal edit interface for full name
- **Profile Updates**: Changes sync to Supabase auth
- **Sign Out**: One-click logout from profile menu

---

## Implementation Details

### AuthContext (`src/contexts/AuthContext.tsx`)

**Provides**:
```typescript
{
  user: User | null,           // Current auth user
  loading: boolean,             // Auth loading state
  isStaff: boolean,            // Role-based flag
  isAdmin: boolean,            // Admin flag
  signUp: Function,            // Register new user
  signIn: Function,            // Login
  signOut: Function,           // Logout
  updateProfile: Function,     // Update user metadata
}
```

**Usage**:
```tsx
const { user, signIn, signOut } = useAuth();

if (!user) {
  return <LoginPage />;
}

return (
  <button onClick={signOut}>
    Logout
  </button>
);
```

### Supabase Configuration

**Client Setup** (`src/lib/supabase.ts`):
```typescript
const supabase = createClient(url, key, {
  auth: {
    persistSession: true,        // Save session to localStorage
    autoRefreshToken: true,      // Auto-refresh before expiry
    detectSessionInUrl: true,    // Detect callback URLs
  },
  realtime: {
    params: {
      eventsPerSecond: 10,       // Rate limiting
    },
  },
});
```

### Login Page (`src/pages/LoginPage.tsx`)

**Features**:
- Beautiful gradient background
- Tab toggle between Sign In / Sign Up
- Form validation
- Error message display
- Loading state indicators
- Responsive design

**States**:
- Empty form
- Loading (with spinner)
- Error display
- Success (redirects)

### User Profile Component (`src/components/UserProfile.tsx`)

**Features**:
- Profile card with avatar
- Email display
- Full name display (editable)
- Edit/Save/Cancel controls
- Sign out button
- Success/error messages

---

## User Roles

VenueIQ supports role-based access via user metadata:

### Attendee (Default)
- Access to all venue views
- No admin capabilities
- Can only view public data
- Restricted to their own user data

### Staff (`role: 'staff'`)
- Access to staff coordination view
- Can view task assignments
- Can acknowledge alerts
- Can update workload status

### Admin (`role: 'admin'`)
- Access to all views including admin panel
- Can update zone occupancy
- Can broadcast announcements
- Can manage staff assignments
- Can modify facility wait times

**Setting Roles**:
Currently roles are set via user metadata. To set an admin/staff role:

```typescript
// In Supabase dashboard, edit user metadata:
{
  "role": "admin"  // or "staff"
}
```

Or programmatically:
```typescript
await supabase.auth.updateUser({
  data: { role: 'admin' }
});
```

---

## Session Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    APP STARTUP                              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  AuthContext checks for existing session                    │
│  (from localStorage or previous auth)                       │
└─────────────────────────────────────────────────────────────┘
                          ↓
         ┌────────────────┴────────────────┐
         ↓                                 ↓
    SESSION FOUND                     NO SESSION
         ↓                                 ↓
    Load user data          ┌──────────────────────────┐
    Show app                │  Show LoginPage          │
                            │  - Sign In form          │
                            │  - Sign Up form          │
                            └──────────────────────────┘
                                      ↓
                         ┌─────────────┴─────────────┐
                         ↓                           ↓
                    SIGN UP                     SIGN IN
                         ↓                           ↓
              Create auth account       Validate credentials
                         ↓                           ↓
              Set session cookie        Set session cookie
                         ↓                           ↓
          Load app dashboard    Load app dashboard
```

---

## Security Features

### ✅ Implemented

1. **Secure Password Storage**
   - Passwords are hashed with bcrypt by Supabase
   - Never transmitted in plaintext
   - HTTPS-only communication

2. **Session Security**
   - Session tokens stored in secure cookies
   - Auto-expiry after inactivity
   - Automatic token refresh

3. **Row-Level Security (RLS)**
   - Database data filtered by authenticated user
   - Users can only access their own data
   - Staff/admin get elevated access via policies

4. **CORS Protection**
   - API requests validated by Supabase
   - Restricted to authorized domains

5. **Environment Variables**
   - API keys never exposed in client code
   - Keys loaded from `.env` file
   - Different keys for different environments

### 🔒 Best Practices

**DO:**
- ✅ Use HTTPS in production
- ✅ Store sensitive data in Supabase (not localStorage)
- ✅ Validate on both client and server
- ✅ Use RLS policies for all tables
- ✅ Refresh tokens automatically

**DON'T:**
- ❌ Store passwords client-side
- ❌ Expose API keys in code
- ❌ Trust client-side validation alone
- ❌ Disable token refresh
- ❌ Store sensitive data in localStorage

---

## Testing Authentication

### Test Accounts

Since email verification is disabled, you can test with any email:

```
Email: test@example.com
Password: TestPassword123!
Full Name: Test User
```

### Test Flows

**Sign Up Flow**:
1. Click "Don't have an account? Sign Up"
2. Enter email, password, full name
3. Click "Create Account"
4. Redirected to dashboard

**Sign In Flow**:
1. Enter registered email
2. Enter password
3. Click "Sign In"
4. Redirected to dashboard

**Profile Management**:
1. Click username in header
2. Edit full name
3. Click "Save"
4. Changes persist to database

**Sign Out**:
1. Click username in header
2. Click "Sign Out"
3. Redirected to login page
4. Session cleared

---

## API Endpoints Used

### Auth Endpoints
- `auth.signUp()` - Register new user
- `auth.signInWithPassword()` - Login
- `auth.signOut()` - Logout
- `auth.getSession()` - Get current session
- `auth.updateUser()` - Update profile
- `auth.onAuthStateChange()` - Listen for auth changes

### Supabase Table Access
- Filtered by `auth.uid()` in RLS policies
- Data automatically filtered for logged-in user
- Admin/staff get elevated access via roles

---

## Troubleshooting

### Issue: "Missing Supabase environment variables"
**Solution**: Ensure `.env` file contains:
```
VITE_SUPABASE_URL=https://...
VITE_SUPABASE_ANON_KEY=...
```

### Issue: Login fails with "Invalid login credentials"
**Solution**: 
- Verify email is correct
- Check password matches
- Ensure user account exists

### Issue: Session not persisting across refreshes
**Solution**:
- Check browser allows cookies
- Verify `persistSession: true` in client config
- Check localStorage is not disabled

### Issue: Profile updates not showing
**Solution**:
- Refresh page after update
- Check browser console for errors
- Verify user has update permission

---

## Future Enhancements

1. **Multi-Factor Authentication (MFA)**
   - TOTP/authenticator app support
   - SMS verification options

2. **Social Authentication**
   - Google/Apple sign-in
   - OAuth provider integration

3. **Password Reset**
   - Email-based recovery
   - Reset link generation

4. **Advanced Roles**
   - Custom role permissions
   - Role-based UI hiding
   - Granular access control

5. **Audit Logging**
   - Track auth events
   - Login/logout history
   - Security event logging

6. **Session Management**
   - Device management
   - Active sessions list
   - Remote logout capability

---

## Database User Tables

VenueIQ integrates with Supabase's built-in `auth.users` table. No custom user table is needed.

**User Metadata Fields**:
```json
{
  "full_name": "John Doe",
  "role": "admin",
  "avatar_url": "https://..."
}
```

**RLS Policy Example**:
```sql
-- Users can only view their own profile
CREATE POLICY "Users can view own profile"
  ON users FOR SELECT
  TO authenticated
  USING (auth.uid() = id);

-- Admins can view all users
CREATE POLICY "Admins can view all"
  ON users FOR SELECT
  TO authenticated
  USING (
    (SELECT auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
  );
```

---

## Environment Setup

### Local Development
```bash
# Create .env file
VITE_SUPABASE_URL=your_supabase_url
VITE_SUPABASE_ANON_KEY=your_anon_key

# Run dev server
npm run dev
```

### Production
- Deploy to Vercel, Netlify, or similar
- Set environment variables in deployment config
- Ensure HTTPS is enabled
- Configure Supabase auth settings

---

## Support

For authentication issues:
1. Check browser console for errors
2. Verify Supabase connection
3. Check RLS policies
4. Review auth configuration
5. Contact Supabase support if needed

VenueIQ's authentication system ensures secure access to venue data while maintaining flexibility for different user roles.
