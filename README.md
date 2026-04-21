## BIG-IP AS3 Generation/Validation and Push

## The Problem

Deploying application configurations to F5 BIG-IP devices manually is tedious and error-prone. Engineers spend 20–40 minutes per application defining virtual IPs, pools, pool members, SSL certificates, and health monitors through the GUI or TMSH — and each deployment risks configuration drift between environments. Mistakes aren't caught until the application fails to respond properly in production.

## How It Works

This repository uses Ansible and the F5 AS3 (Application Services 3) extension to define BIG-IP application configurations declaratively in JSON. You provide a set of variables describing the application (VIPs, pools, members, certificates), and the automation:

1. Generates an AS3 declaration from a Jinja2 template using your input
2. Validates the declaration against the AS3 schema using `as3ninja` before any changes are applied
3. Pushes the validated declaration to the target BIG-IP device
4. Provides cleanup capability to remove an AS3 tenant when an application is decommissioned
5. Emails deployment results for audit and tracking

Because AS3 is declarative, re-running the same declaration produces the same result every time — eliminating configuration drift.

## Requirements

- Ansible 2.12 or later
- Python 3.8+
- `as3ninja` Python package installed in your Ansible execution environment (`pip install as3ninja`)
- F5 BIG-IP device running TMOS 13.1.0 or later with the AS3 extension installed
- Credentials for the target BIG-IP with appropriate role permissions

## Usage

1. Define your application variables following the schema documented below (VIPs, pools, members, certs)
2. Run the deployment playbook: `ansible-playbook AS3_Deployment.yml -e @vars/my_app.yml`
3. The playbook validates the AS3 declaration against the schema before pushing — any errors halt execution before touching the F5
4. To decommission: `ansible-playbook AS3_Cleanup.yml -e @vars/my_app.yml`

See the Variables and JSON Example sections below for full schema details.

### **Variables**

This document defines all key/value combinations for BIG-IP application VIP deployment. Provide your own AS3 templates in Jinja2 format and add additional variables as needed.

| Variable            | Example            | Description                                                                                      |
| :------------------ | :----------------- | :----------------------------------------------------------------------------------------------- |
| bigip_data          | { ... }            | List containing BIG-IP data objects.                                                             |
| tenant_id           | "TENANT12345"      | The application TENANT ID prepended with the letters TENANT.                                     |
| f5_env              | "dev_shared"       | The F5 ansible inventory group to use for application deployment                                 |
| vip_list            | [ ... ]            | Array of VIP definitions.                                                                        |
| vip_ip              | "10.0.0.1"         | The IP address to use for the VIP.                                                               |
| vip_port            | 443                | The port number to use for the VIP.                                                              |
| vip_type            | "https"            | The protocol to use for the VIP. Allowed: https, tcp                                             |
| cert_name           | "mycert.example.com" | The certificate to apply to the VIP for SSL. Required if vip_type is https.                    |
| pool_name           | "app-pool"         | The pool name (defined in the pool_list) that this VIP should reference.                         |
| pool_list           | [ ... ]            | Array of pool definitions.                                                                       |
| pool_lb_method      | "round-robin"      | The load balancing method used by the pool. See schema for full list.                            |
| pool_monitor        | ["tcp"]            | The health monitor(s) type to be used for this pool. Allowed: tcp, https                         |
| pool_members        | [ ... ]            | Array of pool member definitions.                                                                |
| member_addr         | "10.0.0.2"         | The IPV4 IP address of the pool member.                                                          |
| member_service_port | 8080               | The listening port number of the pool member.                                                    |
| member_state        | "enable"           | The state of the pool member. Allowed: enable, disable, offline. Enabled by default.             |

> **Note:** Some fields are arrays of objects. See the schema for nested structure details and required fields.

## **Basic JSON Example**

Below is the most minimal valid example for this schema:

```json
{
  "bigip_data": {
    "tenant_id": "TENANT12345",
    "f5_env": "dev_shared",
    "vip_list": [
      {
        "vip_ip": "10.0.0.1",
        "vip_port": 443,
        "vip_type": "https",
        "cert_name": "mycert.example.com",
        "pool_name": "app-pool"
      }
    ],
    "pool_list": [
      {
        "pool_name": "app-pool",
        "pool_lb_method": "round-robin",
        "pool_monitor": ["tcp"],
        "pool_members": [
          {
            "member_addr": "10.0.0.2",
            "member_service_port": 8080,
            "member_state": "enable"
          }
        ]
      }
    ]
  }
}
```

## **AS3 Example**
Refer to the [as3_declaration.j2](as3_deployment_role/templates/as3_declaration.j2) file

### **Notes for Usage**
> Ensure that you have installed the 'as3ninja' Python tool via pip into your Ansible Execution Environment to allow for validation of your AS3 declaration prior to applying to the F5.