# AZ-104 – Identities & Governance Lab

### 1. Users and groups creation, admin access management

Create 10 users, split into 5 groups :

* 1 user in Super Admin (tenant creator user)
* 1 user in Subscriptions Manager (existing user)
* 5 users in Team A
* 5 users in Team B

Users must be assigned permissions via groups only.

➡️ [Solution here](./lab_part1.md)

---

### 2. Management Groups and RBAC

Create 2 Management Groups, directly under the Root Management Group:

* Team A
* Team B

#### RBAC assignments:

* Team A users have the Contributor role on the Team A Management Group
* Team B users have the Contributor role on the Team B Management Group
* Super Admin users have Owner permissions on the Root Management Group
* Subscriptions Manager users have Contributor (or appropriate subscription management role) on the Root Management Group

---

### 3. Subscription Management

Create 2 subscriptions per Management Group :
- Dev
- Prod

```
🗂️ root management group
├─ 🗂️ team a
│  ├─ 🔑dev
│  ├─ 🔑prod
├─ 🗂️ team b
│  ├─ 🔑dev
│  ├─ 🔑prod
``` 

---

### 4. Azure Policies

#### Tag inheritance policies

Use Azure Policies to enforce tag inheritance:

* `team = A` applied at the Team A Management Group
* `team = B` applied at the Team B Management Group
* `env = dev` applied at Dev subscriptions
* `env = prod` applied at Prod subscriptions

Policy effect : modify or append

#### Location restriction policies  

Use Azure Policies to restrict resource deployment locations:

* Team A can deploy resources only in France Central
* Team B can deploy resources only in Switzerland North

Policy effect : deny
