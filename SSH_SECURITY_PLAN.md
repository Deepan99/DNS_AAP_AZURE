# SSH Security Industrial Practice Plan

## Current Configuration Analysis

**Current Setup:**
```yaml
- StrictHostKeyChecking=no (security risk)
- UserKnownHostsFile=/dev/null (security risk)
- PubkeyAcceptedKeyTypes=+ssh-rsa (legacy but compatible)
- ProxyJump through bastion (good practice)
```

## Security Assessment

**Current Configuration: 4/10 Security Score**
- ✅ Uses SSH key authentication
- ✅ Implements bastion jump host pattern
- ✅ Uses private network for target VMs
- ❌ Disables host key verification (man-in-the-middle attack risk)
- ❌ Discards known hosts (no trust establishment)
- ❌ No key rotation strategy
- ❌ No audit trail for SSH key access

---

## Industrial Standard SSH Security Implementation

### Phase 1: Immediate Security Hardening (Baseline)

#### 1.1 Implement Host Key Verification
**Current:** `StrictHostKeyChecking=no`
**Target:** Use AAP credential-based known hosts management

**Implementation Approach:**
```yaml
# Option A: AAP Credential-Based Known Hosts
ansible_ssh_common_args: >-
  -o StrictHostKeyChecking=accept-new
  -o UserKnownHostsFile={{ lookup('env', 'ANSIBLE_SSH_KNOWN_HOSTS') | default('~/.ssh/known_hosts') }}

# Option B: AAP Job Template Credential Injection
# Configure known hosts as a credential in AAP and inject via extra_vars
```

**Rationale:** `accept-new` is industrial standard for dynamic environments - it accepts new host keys but validates existing ones, balancing security with automation needs.

#### 1.2 SSH Key Management Strategy
**Current:** Single key across all VMs
**Target:** Key rotation and separation of concerns

**Implementation:**
```yaml
# Separate keys for different security zones
# bastion_key: bastion access only
# dns_client_key: client access only
# management_key: administrative access
```

---

### Phase 2: Enterprise Security Integration (Advanced)

#### 2.1 AAP Credential Integration
**Target:** Use AAP's credential system for SSH key management

**Implementation Steps:**
1. **Create SSH Credentials in AAP:**
   - Credential Type: Machine
   - SSH Key: Upload private key
   - Username: azureuser
   - SSH Key Unlock: No password (or encrypted)

2. **Configure Known Hosts Management:**
   - Create credential for known_hosts file
   - Update AAP job templates to use credentials

3. **Vault Integration:**
   - Use HashiCorp Vault for SSH key storage
   - AAP retrieves keys dynamically from Vault

#### 2.2 Just-In-Time SSH Access
**Target:** Implement JIT access patterns

**Implementation:**
```yaml
# Use Microsoft Defender for Cloud JIT
# or integrate with CyberArk/HashiCorp Vault for SSH key issuance
```

---

### Phase 3: Compliance & Audit Trail (Production Ready)

#### 3.1 SSH Session Logging
**Target:** Complete audit trail of SSH access

**Implementation:**
```yaml
# Enable SSH session logging on all VMs
- name: Configure SSH session logging
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    line: "LogLevel VERBOSE"
    notify: Restart sshd

# Centralized logging to Log Analytics
- name: Configure SSH logging to syslog
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    line: "SyslogFacility AUTHPRIV"
    notify: Restart sshd
```

#### 3.2 Key Rotation Policy
**Target:** Automated key rotation

**Implementation:**
```yaml
# Implement 90-day key rotation policy
# Use AAP scheduler for key rotation playbooks
# Maintain backup of previous keys for recovery period
```

---

## Recommended Configuration for Your Environment

### Immediate Implementation (Balanced Security)

**File:** `group_vars/dns_clients.yml`
```yaml
---
# Industrial practice SSH configuration for AAP automation
# Security level: Balanced (appropriate for dynamic infrastructure)
ansible_ssh_common_args: >-
  -o StrictHostKeyChecking=accept-new
  -o UserKnownHostsFile={{ lookup('env', 'ANSIBLE_SSH_KNOWN_HOSTS') | default('~/.ssh/known_hosts') }}
  -o PubkeyAcceptedKeyTypes=+ssh-rsa
  -o ProxyJump=azureuser@{{ hostvars[groups['dns_servers'][0]].ansible_host }}
  -o ConnectTimeout=30
  -o ServerAliveInterval=60
  -o ServerAliveCountMax=3
```

### Security Explanation

**Configuration Choices:**
1. **StrictHostKeyChecking=accept-new**
   - Accepts new host keys on first connection
   - Validates existing host keys on subsequent connections
   - Industrial standard for dynamic cloud environments
   - Balances security with automation needs

2. **UserKnownHostsFile management**
   - Uses environment variable for flexibility
   - Falls back to default location
   - Enables centralized known hosts management

3. **Additional Security Parameters**
   - `ConnectTimeout=30`: Prevents hanging connections
   - `ServerAliveInterval=60`: Detects dead connections
   - `ServerAliveCountMax=3`: Terminates unresponsive connections

---

## Production-Ready Implementation Plan

### Step 1: AAP Credential Setup (Week 1)
1. Create SSH machine credential in AAP
2. Upload bastion SSH private key
3. Configure job templates to use credential
4. Test with `StrictHostKeyChecking=accept-new`

### Step 2: Known Hosts Management (Week 2)
1. Extract current known hosts from bastion
2. Upload as AAP credential
3. Update inventory to use managed known hosts
4. Enable StrictHostKeyChecking=yes

### Step 3: Key Rotation Strategy (Week 3)
1. Implement key rotation playbook
2. Set up AAP scheduler for 90-day rotation
3. Document key recovery procedures
4. Test rotation in non-production

### Step 4: Audit & Compliance (Week 4)
1. Enable SSH session logging
2. Configure centralized log collection
3. Set up alerts for unauthorized access attempts
4. Implement access review procedures

---

## Security vs Convenience Trade-offs

| Configuration | Security Level | Convenience | Recommendation |
|---------------|---------------|-------------|----------------|
| `StrictHostKeyChecking=no` | ❌ Low | ✅ High | ❌ Avoid in production |
| `StrictHostKeyChecking=accept-new` | ✅ Medium | ✅ High | ✅ Recommended for AAP |
| `StrictHostKeyChecking=yes` | ✅ High | ⚠️ Medium | ✅ Production with managed keys |
| `StrictHostKeyChecking=yes + managed keys` | ✅✅ Very High | ⚠️ Low | ✅ Enterprise standard |

---

## Alternative: Certificate-Based Authentication

**For maximum security, consider SSH certificates:**

```yaml
# Instead of SSH keys, use SSH certificates
# Requires Certificate Authority (CA) setup
# Benefits: Automatic expiration, revocation, audit trail
```

**Implementation:**
1. Set up SSH Certificate Authority
2. Issue certificates with short TTL (hours/days)
3. Configure AAP to request certificates on demand
4. Automate certificate rotation

---

## Final Recommendation for Your Environment

**Phase 1 (Immediate):** Implement the balanced configuration above
**Phase 2 (Next Sprint):** Integrate AAP credential management
**Phase 3 (Production):** Full enterprise security with managed keys

The balanced configuration provides good security while maintaining operational efficiency for your DNS automation project.