---
title: "Securing Azure Identities: The \"New\" Perimeter in Cybersecurity"
date: 2025-03-18 08:48:00 +0000
categories: [Azure, Identity]
tags: [azure, entra-id, identity, security, mfa]
author: Jere Haavisto
image:
  path: /assets/img/posts/2025/03/UntitledImage.png
---

As cloud computing becomes more integrated into our daily operations, the importance of securing identities in Azure can't be overstated. Gone are the days when a strong firewall was enough to protect our networks. With cloud-native services now accessed over the internet, the key to security lies in safeguarding identities and access credentials.

## Types of Identity in Azure

As identity is the thing to secure, you should be familiar with the different types of identities in Azure. Good thing about the variety in identities is that one size never fits all and there are plenty of use cases to consider. Bad thing is that you can compromise the security of a service if you choose poorly.

- **User Identity**
  - Member users – Created and managed within the Entra ID tenant or Active Directory and synced to cloud with Entra ID Connect.
  - Guest users – External users invited via Azure AD B2B (Business-to-Business) collaboration to access specific resources.
  - Consumer users – Managed through Entra ID B2C for applications requiring customer authentication.

- **Service Principals**
  - Application based – Application Registrations in Azure.
  - Managed Identities
    - User assigned – Created independently of resources and can be assigned to multiple resources.
    - System assigned – Automatically created and managed by Azure for a specific resource. When the resource is deleted, the identity is removed.

- **Other**
  - Device Identity – Entra ID registered, Entra ID joined, Hybrid joined devices.
  - External identities – Federated.
  - Group identities – Entra ID Security groups and Microsoft 365 Groups.
  - Role based identities – Azure RBAC roles.
  - Temporary identities – TAP or temporary access pass.

As you can see there are a lot of different types of identities used in Azure. One might argue that a group for example is not an identity but since you can access something based on the group you are in, I'd argue that it is an identity to be guarded just as a normal account should be.

## Quick tips to secure Identities

Securing identities doesn't have to be overwhelming as there are a lot of small things that improve security by a mile. I´ll do a division between normal users and service principals/managed identities, since they work in different roles.

**Users:**

- **Enable Multi-Factor Authentication (MFA):** If you do one thing, do this. It's the simplest way to stop password-based attacks in their tracks.
- **Use phishing resistant methods for privileged roles** – FIDO2 keys or Windows Hello for Business for anyone with admin roles.
- **Review and remove stale accounts** – Entra ID Access Reviews make this easy to automate.
- **Implement Conditional Access policies** – Require compliant devices, block legacy auth, and enforce risk-based access.
- **Enable Privileged Identity Management (PIM)** – No standing access to sensitive roles. Just-in-time access only.

**Service Principals / Managed Identities:**

- **Prefer Managed Identities over Service Principals with secrets** – No secrets to rotate or accidentally expose.
- **Apply least privilege** – It is often the case that during development, the application or workload identity has a pretty "strong" role just to make development easier and faster. These roles should be minimized before going to production.
- **Do not assign owners to Application Registrations** – Let´s say that you have a normal user with no roles and that user is owner of an application that has `Directory.ReadWrite.All`. The application provides a path to privilege escalation for that user since, as an owner, that user can access the directory through the app.
- **Monitor and Review Regularly** – Review guest accounts and app permissions to ensure everything aligns with the principle of least privilege.

## Conclusion

Securing Azure identities isn't just a technical task—it's a mindset shift. As attackers get smarter and more relentless, we need to focus on identities as the foundation of our security strategies. It's not just about protecting the data but ensuring our teams can work securely and confidently in the cloud.

Identity is the new perimeter, and securing it is our best chance to stay ahead of the curve. With Microsoft's robust tools and a clear strategy, we can do just that.

---

Check my post about securing PaaS resources: [Securing Azure PaaS Resources With Network Security Perimeter](https://jerehaavisto.com/azure/networks/2025/03/19/securing-azure-paas-resources.html)
