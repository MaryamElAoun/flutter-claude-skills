# Firestore security rules templates

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() { return request.auth != null; }
    function isOwner(uid) { return signedIn() && request.auth.uid == uid; }
    function isAdmin() {
      return signedIn() && request.auth.token.admin == true; // custom claim
    }
    function onlyChanges(fields) {
      return request.resource.data.diff(resource.data).affectedKeys().hasOnly(fields);
    }

    // user profile: owner reads/writes, cannot make themself admin
    match /users/{uid} {
      allow read: if isOwner(uid) || isAdmin();
      allow create: if isOwner(uid) && !('role' in request.resource.data);
      allow update: if isOwner(uid)
                    && !request.resource.data.diff(resource.data).affectedKeys().hasAny(['role']);
      allow delete: if isAdmin();

      match /tokens/{token} {
        allow read, write: if isOwner(uid);
      }
    }

    // public catalog: everyone reads, only admins write
    match /products/{id} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // orders: user creates own, can't change status/total later
    match /orders/{id} {
      allow read: if isOwner(resource.data.userId) || isAdmin();
      allow create: if isOwner(request.resource.data.userId)
                    && request.resource.data.status == 'pending'
                    && request.resource.data.total is number;
      allow update: if isAdmin()
                    || (isOwner(resource.data.userId) && onlyChanges(['note']));
      allow delete: if false;
    }

    // deny everything else
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

## Review checklist

- No `allow read, write: if true;` outside truly public, read-only data.
- No `if request.auth != null` alone on user data (any logged-in user could read everyone's data).
- Validate types and required fields on `create`.
- Protect privileged fields (`role`, `status`, `total`, `balance`) from client updates.
- Remember `list` queries must be provably filtered by the rule (e.g. query must include `where('userId', '==', uid)`).
- Admin roles via custom claims set server-side, not via a field the client can write.
- Storage rules: limit `request.resource.size` and `contentType`.
