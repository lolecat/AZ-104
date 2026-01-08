# 4. Azure Policies

**Lab instructions reminder:**

#### Tag inheritance policies
Use Azure Policies to enforce tag inheritance:
- `team = A` applied at the Team A Management Group
- `team = B` applied at the Team B Management Group
- `env = dev` applied at Dev subscriptions
- `env = prod` applied at Prod subscriptions

Policy effect: modify or append

#### Location restriction policies
Use Azure Policies to restrict resource deployment locations:
- Team A can deploy resources only in France Central
- Team B can deploy resources only in Switzerland North

Policy effect: deny

Cmdlets used in this lab :
- [ex1](https://)
- [ex2](https://)

<br>

## 1. Tag Inheritance Policies

### Understanding Tag Policies

**Why tag inheritance?**
- Automatic tagging of all resources based on their location in the hierarchy
- Cost tracking by team and environment
- Compliance and governance automation

**Policy effects:**
- **Append**: Adds tags if they don't exist (won't override existing tags)
- **Modify**: Adds or updates tags (can override existing tags)

For this lab, we'll use **Modify** to ensure consistent tagging.

<br>

### Step 1: Create Tag Policies at Management Group Level

[À développer...]

## 2. Location Restriction Policies

[À développer...]

## 3. Testing and Validation

[À développer...]