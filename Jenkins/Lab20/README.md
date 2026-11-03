## Objective

Restrict access to Jenkins using the **Role-based Authorization Strategy** plugin:

- Create two users: `user1` and `user2`.
- Assign the **admin** role to `user1`.
- Assign a **read-only** role to `user2`.

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| Jenkins (Docker container) | CI/CD server |
| Role-based Authorization Strategy plugin | Defines roles and assigns them to users |
| Jenkins' own user database | Local user management |
| Docker Desktop / WSL | Hosts the Jenkins container |

## Architecture

```
                 +----------------------+
                 |       Jenkins        |
                 |  Role-Based Strategy |
                 +----------+-----------+
                            |
          +-----------------+-----------------+
          |                                   |
   +------+-------+                   +-------+------+
   | Role: admin  |                   | Role: read-  |
   | Administer   |                   | only         |
   |              |                   | Overall/Read |
   +------+-------+                   +-------+------+
          |                                   |
        user1                               user2
```

## Steps

### Step 1: Install the plugin

`Manage Jenkins` -> `Plugins` -> `Available plugins`, search for **Role-based Authorization Strategy**, then install it.

### Step 2: Enable the Security Realm and Authorization strategy

`Manage Jenkins` -> `Security`:

- **Security Realm:** Jenkins' own user database
- **Authorization:** Role-Based Strategy

Save the configuration.

### Step 3: Create the users

`Manage Jenkins` -> `Users` -> `Create User`. Create:

| Username | Purpose |
|----------|---------|
| user1 | Administrator |
| user2 | Read-only user |

### Step 4: Create the roles

`Manage Jenkins` -> `Manage and Assign Roles` -> `Manage Roles` -> `Global roles` -> `Add Role`.

| Role | Permissions |
|------|-------------|
| admin | Overall / Administer |
| read-only | Overall / Read, Job / Read |

### Step 5: Assign the roles

`Manage and Assign Roles` -> `Assign Roles` -> `Global roles` -> `Add User`.

| User | Role |
|------|------|
| rewan | admin |
| user1 | admin |
| user2 | read-only |

The original administrator account is also assigned `admin` to avoid being locked out.

<img width="1275" height="677" alt="Screenshot 2026-11-03 171923" src="https://github.com/user-attachments/assets/381a22b5-80c1-46d2-87fe-2d81a7a907f0" />

### Step 6: Verify

| Login as | Expected result |
|----------|-----------------|
| user1 | Full access: can create jobs, configure Jenkins, manage plugins |
| user2 | View only: no New Item, Build, or Configure options |


## Conclusion

Role-based authorization was configured in Jenkins. `user1` holds the admin role with full control, while `user2` holds a read-only role limited to viewing. This follows the principle of least privilege and is a basic access-control practice for a DevSecOps workflow.
