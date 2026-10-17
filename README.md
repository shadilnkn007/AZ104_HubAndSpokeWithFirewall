# Azure Hub-and-Spoke Architecture with Azure Firewall Standard

**Student lab \| Azure portal \| Region: Central India**

## Lab objective

Build a hub-and-spoke network in Azure with centralized traffic
inspection using Azure Firewall. You will create three virtual networks,
peer the hub with two spokes, route spoke traffic through the firewall,
configure a DNAT rule for RDP, and test outbound web access.

> **Important feature note --- read before starting:** Azure Firewall
> **web categories** (including the **Business** category) are a
> **Premium** feature. This lab uses **Azure Firewall Standard**, as
> requested, and therefore uses an **application rule with explicit
> FQDNs** for the web-access exercise. If your instructor specifically
> requires the built-in **Business web category**, deploy **Azure
> Firewall Premium** instead and select the Business category in the
> application rule. Do not expect the Business category to be available
> on Standard.

## Architecture and IP plan

All resources are deployed in **Central India**.

  -----------------------------------------------------------------------
  Resource                Name (suggested)        Address / configuration
  ----------------------- ----------------------- -----------------------
  Resource group          `rg-hubspoke-lab`       Central India

  Hub VNet                `vnet-hub-ci`           `10.0.0.0/16`

  Azure Firewall subnet   `AzureFirewallSubnet`   `10.0.0.0/26`

  Hub management/test     `snet-hub-test`         `10.0.1.0/24`
  subnet (optional)                               

  Spoke 1 VNet            `vnet-spoke1-ci`        `10.10.0.0/16`

  Spoke 1 VM subnet       `snet-vm`               `10.10.1.0/24`

  Spoke 2 VNet            `vnet-spoke2-ci`        `10.20.0.0/16`

  Spoke 2 VM subnet       `snet-vm`               `10.20.1.0/24`

  VM in Spoke 1           `vm-spoke1`             Windows Server 2022; no
                                                  public IP

  VM in Spoke 2           `vm-spoke2`             Windows Server 2022; no
                                                  public IP

  Route tables            `rt-spoke1-to-fw`,      `0.0.0.0/0` → firewall
                          `rt-spoke2-to-fw`       private IP

  Firewall policy         `afwp-hub-ci`           Standard-compatible
                                                  policy

  Azure Firewall          `afw-hub-ci`            Standard; public IP
                                                  required for DNAT
  -----------------------------------------------------------------------

**Why is `AzureFirewallSubnet` /26?** Azure Firewall requires a subnet
with this exact name and a sufficiently large address range; `/26` is
the recommended minimum for this lab. Do not place VMs or other
resources in this subnet.

### Traffic flows to demonstrate

-   **Outbound:** Spoke VM → route table → Azure Firewall private IP →
    Internet (if allowed by firewall policy).
-   **Inbound RDP:** Your client → firewall public IP, TCP `50001` →
    DNAT → `vm-spoke1` private IP, TCP `3389`.
-   **Spoke-to-spoke:** The default route sends traffic to the firewall.
    Communication is permitted only if the firewall policy has a
    matching network rule and the return path is correct. This lab does
    not create direct peering between the spokes.

## Prerequisites and cost warning

-   An Azure subscription with permission to create VNets, peerings,
    VMs, public IPs, route tables, and Azure Firewall.
-   Choose a unique VM administrator username and a strong password. Do
    not commit credentials to GitHub.
-   Azure Firewall is a billable resource and can cost significantly
    more than a small VM. **Delete the lab resource group when
    finished.**
-   For RDP, use a trusted client public IP as the DNAT source when
    possible. Avoid allowing RDP from `*` in a production environment.

## Part 1 --- Create the resource group

1.  Sign in to the [Azure portal](https://portal.azure.com).
2.  Search for **Resource groups** → **Create**.
3.  Select your subscription.
4.  Resource group: `rg-hubspoke-lab`.
5.  Region: **Central India**.
6.  Select **Review + create** → **Create**.

**Explanation:** A resource group gives the lab one place to manage and
delete the resources.

## Part 2 --- Create the hub VNet and firewall subnet

1.  Search **Virtual networks** → **Create**.
2.  Basics:
    -   Resource group: `rg-hubspoke-lab`
    -   Name: `vnet-hub-ci`
    -   Region: **Central India**
3.  IP addresses:
    -   IPv4 address space: `10.0.0.0/16`
    -   Create subnet named exactly `AzureFirewallSubnet`, address range
        `10.0.0.0/26`.
    -   Optionally create `snet-hub-test` with `10.0.1.0/24` if you want
        a future test subnet. No VM is required in the hub for this lab.
4.  Review + create → Create.

**Explanation:** The hub is the shared network where central inspection
is provided. The firewall must be deployed into `AzureFirewallSubnet`.

## Part 3 --- Create the two spoke VNets

Repeat the VNet creation process for each spoke.

### Spoke 1

-   Name: `vnet-spoke1-ci`
-   Region: Central India
-   Address space: `10.10.0.0/16`
-   Subnet name: `snet-vm`
-   Subnet range: `10.10.1.0/24`

### Spoke 2

-   Name: `vnet-spoke2-ci`
-   Region: Central India
-   Address space: `10.20.0.0/16`
-   Subnet name: `snet-vm`
-   Subnet range: `10.20.1.0/24`

**Explanation:** Each spoke contains workloads and has a separate,
non-overlapping address range. VNet peering is not transitive, so the
spokes will not communicate directly just because each is peered with
the hub.

## Part 4 --- Create an Azure Firewall Policy

1.  Search for **Firewall policies** → **Create**.
2.  Resource group: `rg-hubspoke-lab`.
3.  Name: `afwp-hub-ci`.
4.  Region: Central India.
5.  Policy tier: **Standard** (if the portal asks for a tier).
6.  Create the policy.

**Explanation:** A Firewall Policy centrally stores rule collection
groups and rules. The firewall will use this policy instead of managing
rules directly on the firewall resource.

## Part 5 --- Deploy Azure Firewall Standard

1.  Search **Firewalls** → **Create**.
2.  Basics:
    -   Resource group: `rg-hubspoke-lab`
    -   Firewall name: `afw-hub-ci`
    -   Region: Central India
    -   Firewall SKU/tier: **Standard**
    -   Firewall management: **Use a Firewall Policy to manage this
        firewall**
    -   Firewall policy: `afwp-hub-ci`
3.  Choose existing virtual network: `vnet-hub-ci`.
4.  Select the `AzureFirewallSubnet` created earlier.
5.  Create a new **Standard** public IP, for example `pip-afw-hub-ci`.
6.  Review + create → Create. Deployment can take several minutes.
7.  After deployment, open `afw-hub-ci` → **Overview** and record:
    -   **Private IP address**: `FIREWALL_PRIVATE_IP`
    -   **Public IP address**: `FIREWALL_PUBLIC_IP`

Replace these placeholders with the actual values in your own lab notes.
Do not assume the private IP will be the same for every deployment.

**Explanation:** The private IP is the next hop used by the spoke route
tables. The public IP is the destination for the inbound DNAT test.

## Part 6 --- Create Windows Server 2022 VMs without public IPs

Create one VM in each spoke. Repeat these steps, changing the values for
Spoke 2.

### VM in Spoke 1

1.  Search **Virtual machines** → **Create** → **Azure virtual
    machine**.
2.  Basics:
    -   Resource group: `rg-hubspoke-lab`
    -   VM name: `vm-spoke1`
    -   Region: Central India
    -   Image: **Windows Server 2022 Datacenter** (or the current
        Windows Server 2022 Datacenter image available in your
        subscription)
    -   Size: choose a small lab-appropriate size available in your
        subscription.
    -   Configure a local administrator username and strong password.
3.  Networking:
    -   Virtual network: `vnet-spoke1-ci`
    -   Subnet: `snet-vm`
    -   Public IP: **None**
    -   NIC network security group: **Basic** or **Advanced**, as
        offered by the portal.
4.  For a simple student lab, allow inbound **RDP (TCP 3389)** in the
    VM's NSG. This rule only opens the VM's private interface; it does
    **not** give the VM a public IP. If the portal does not offer an RDP
    preset, add an inbound security rule after creation:
    -   Source: **IP Addresses** and use your trusted client IP if
        appropriate for the DNAT test; for a controlled lab exercise you
        may use `Any`, but remove or narrow this rule after testing.
    -   Source port ranges: `*`
    -   Destination: `Any`
    -   Service: `RDP` or port `3389`
    -   Protocol: TCP
    -   Action: Allow
    -   Priority: e.g. `300`
    -   Name: `Allow-RDP-Lab`
5.  Review + create → Create.

### VM in Spoke 2

Repeat using: - VM name: `vm-spoke2` - VNet: `vnet-spoke2-ci` - Subnet:
`snet-vm` - Public IP: **None**

6.  Open each VM → **Networking** and confirm **Public IP address: --- /
    None**.
7.  Record each VM's private IP address:
    -   `vm-spoke1`: `SPOKE1_VM_PRIVATE_IP`
    -   `vm-spoke2`: `SPOKE2_VM_PRIVATE_IP`

**Explanation:** A private-only VM is not directly reachable from the
Internet. In this lab, inbound RDP is published through the firewall
public IP and DNAT rule.

> **Security note:** An NSG rule allowing RDP from Any is not
> recommended for production. DNAT is the intended Internet entry point
> in this exercise; where possible, restrict both the DNAT source and
> NSG source to trusted addresses and remove temporary broad rules.

## Part 7 --- Configure hub-to-spoke and spoke-to-hub peerings

Create **four** peering links. For each peering, open the source VNet →
**Peerings** → **+ Add**.

### Peering 1: Hub → Spoke 1

On `vnet-hub-ci`: - Peering link name: `peer-hub-to-spoke1` - Remote
virtual network: `vnet-spoke1-ci` - Allow `vnet-hub-ci` to access
`vnet-spoke1-ci`: **Enabled** - Allow forwarded traffic from the remote
virtual network: **Enabled**

### Peering 2: Spoke 1 → Hub

On `vnet-spoke1-ci`: - Peering link name: `peer-spoke1-to-hub` - Remote
virtual network: `vnet-hub-ci` - Allow `vnet-spoke1-ci` to access
`vnet-hub-ci`: **Enabled** - Allow forwarded traffic from the remote
virtual network: **Enabled**

### Peering 3: Hub → Spoke 2

On `vnet-hub-ci`: - Peering link name: `peer-hub-to-spoke2` - Remote
virtual network: `vnet-spoke2-ci` - Allow access: **Enabled** - Allow
forwarded traffic from the remote virtual network: **Enabled**

### Peering 4: Spoke 2 → Hub

On `vnet-spoke2-ci`: - Peering link name: `peer-spoke2-to-hub` - Remote
virtual network: `vnet-hub-ci` - Allow access: **Enabled** - Allow
forwarded traffic from the remote virtual network: **Enabled**

Check each peering shows **Connected**.

**Explanation:** Peering connects the hub to each spoke. "Allow
forwarded traffic" permits traffic that has been forwarded by an
appliance such as Azure Firewall to cross the peering. Peering does not
create a direct Spoke 1--Spoke 2 connection.

## Part 8 --- Create route tables and force spoke traffic to the firewall

Create one route table per spoke so each workload subnet uses the
firewall as its next hop.

### Route table for Spoke 1

1.  Search **Route tables** → **Create**.
2.  Resource group: `rg-hubspoke-lab`; region: Central India.
3.  Name: `rt-spoke1-to-fw`.
4.  Leave **Propagate gateway routes** disabled unless your instructor
    has a specific reason to enable it.
5.  Create the route table.
6.  Open it → **Routes** → **+ Add**:
    -   Route name: `default-to-firewall`
    -   Destination type: IP Addresses
    -   Destination IP addresses/CIDR ranges: `0.0.0.0/0`
    -   Next hop type: **Virtual appliance**
    -   Next hop address: the actual private IP of `afw-hub-ci`
7.  Save.
8.  Open **Subnets** → **Associate**:
    -   Virtual network: `vnet-spoke1-ci`
    -   Subnet: `snet-vm`
9.  Save.

### Route table for Spoke 2

Repeat with: - Route table: `rt-spoke2-to-fw` - Route:
`default-to-firewall` - Destination: `0.0.0.0/0` - Next hop type:
Virtual appliance - Next hop address: the same `FIREWALL_PRIVATE_IP` -
Associate to `vnet-spoke2-ci` → `snet-vm`

**Explanation:** `0.0.0.0/0` is the default route. It sends traffic
without a more-specific matching route to Azure Firewall for inspection.
The firewall policy must allow the traffic; a route alone does not grant
access.

> **Routing note:** Azure's more-specific routes can take precedence
> over a default route. For a lab focused on inspection of all
> spoke-to-spoke traffic, verify the effective routes on both VMs. If a
> direct peering route is selected for a spoke destination, add explicit
> UDRs for the other spoke prefix on both spoke route tables, with the
> firewall private IP as the next hop. Do not create direct peering
> between the spokes for this lab.

## Part 9 --- Add a DNAT rule for RDP on TCP 50001

This rule allows you to connect to `vm-spoke1` through the firewall
without assigning a public IP to the VM.

1.  Open **Firewall policies** → `afwp-hub-ci`.
2.  Open **Rule collection groups**. Create or open a rule collection
    group for NAT rules.
3.  Add a **NAT rule collection**:
    -   Name: `nat-rdp-spoke1`
    -   Priority: `100` (or another unused priority)
    -   Action: **DNAT**
4.  Add a rule:
    -   Name: `dnat-rdp-spoke1`
    -   Source type: IP address
    -   Source: your trusted public client IP in CIDR notation
        (recommended); use `*` only if necessary for a temporary lab and
        permitted by your instructor.
    -   Protocol: TCP
    -   Destination address: the firewall's **public IP** (or the
        relevant firewall public IP object, depending on the portal UI)
    -   Destination port: `50001`
    -   Translated address: the private IP of `vm-spoke1`
    -   Translated port: `3389`
5.  Save and wait for policy deployment to complete.

**Explanation:** DNAT translates the destination of an incoming
connection. The client connects to the firewall's public IP on port
`50001`; the firewall forwards it to the VM's private IP on port `3389`.

### Test RDP

From a Windows client that is permitted by the DNAT source rule:

1.  Open **Remote Desktop Connection** (`mstsc.exe`).
2.  Computer: `FIREWALL_PUBLIC_IP:50001`
3.  Select **Connect** and sign in with the VM's local administrator
    credentials.

If the connection fails, check the DNAT source restriction, firewall
policy deployment, VM private IP, NSG inbound rule for TCP 3389, VM
power state, and Windows Firewall. Never add a public IP to the VM as a
workaround.

## Part 10 --- Configure outbound application access

### Option A --- Keep Azure Firewall Standard (used in this lab)

Because Standard does not support built-in web categories, use an
application rule with explicit FQDNs for a simple demonstration.

1.  Open `afwp-hub-ci` → **Rule collection groups**.
2.  Create or open a network/application rule collection group.
3.  Add an **Application rule collection**:
    -   Name: `app-allow-approved-web`
    -   Priority: `200` (or another unused priority)
    -   Action: **Allow**
4.  Add a rule:
    -   Name: `allow-approved-web-fqdns`
    -   Source type: IP address
    -   Source: `10.10.0.0/16`, `10.20.0.0/16` (add as separate entries
        if required by the portal)
    -   Protocol: `http:80`, `https:443`
    -   Target FQDNs: `www.microsoft.com`, `learn.microsoft.com`
5.  Save and wait for policy deployment.

This permits the specified website names only. Application rules are
Layer 7 rules, and Azure Firewall evaluates them after DNAT and network
rules.

### Option B --- If the lab specifically requires the "Business" web category

Use **Azure Firewall Premium** instead of Standard. Premium is a
different, higher-cost tier. In the application rule, select **Web
categories** as the target type and choose **Business** (the exact
portal labels may vary). Do not attempt to select this category on
Standard.

**Explanation:** Web categories classify sites by type. The built-in
category named **Business** is a Premium feature; explicit FQDN
filtering is the Standard-compatible alternative.

## Part 11 --- Verify outbound traffic and routing

1.  Confirm both spoke VMs are running and have no public IP addresses.

2.  On each VM, open a browser or PowerShell session and test:

    ``` powershell
    Test-NetConnection www.microsoft.com -Port 443
    ```

    A successful TCP test is useful, but also open the approved website
    in a browser to test the application rule.

3.  Test an unapproved website. It should not be allowed by this
    specific application rule unless another rule or permitted path
    allows it. A rule collection is not an automatic "deny all" if other
    allow rules exist, so check the full policy.

4.  On each VM, open **Networking** → **Effective routes** (or use
    **Network Watcher → Next hop**) and verify the default route points
    to the firewall private IP.

5.  In the firewall policy, review the rule collections and ensure the
    correct rules are enabled and priorities do not conflict.

6.  If diagnostic logging was configured, review Azure Firewall logs in
    the configured Log Analytics workspace. Logging is optional for this
    basic lab and may incur extra cost.

### Optional spoke-to-spoke test

The route tables direct traffic toward the firewall, but the firewall
must explicitly permit the required private traffic.

1.  In `afwp-hub-ci`, add a **Network rule collection** with action
    **Allow** and an unused priority.
2.  Add a rule such as:
    -   Name: `allow-spoke1-to-spoke2-rdp` (only if testing RDP)
    -   Source: `10.10.0.0/16`
    -   Destination: `10.20.0.0/16`
    -   Protocol: TCP
    -   Destination port: `3389`
3.  For two-way initiation, create a corresponding rule for
    `10.20.0.0/16` → `10.10.0.0/16`, or use appropriately scoped rules
    based on the lab objective.
4.  If the route table's default route does not steer the destination
    through the firewall, add explicit routes for the other spoke prefix
    to each spoke route table:
    -   In Spoke 1 route table: `10.20.0.0/16` → Virtual appliance →
        firewall private IP.
    -   In Spoke 2 route table: `10.10.0.0/16` → Virtual appliance →
        firewall private IP.
5.  Verify the effective routes and NSGs on both VMs.

**Explanation:** VNet peering is non-transitive. The hub-and-spoke
firewall pattern works by routing traffic through the hub firewall, and
the firewall policy must permit the traffic.

## Troubleshooting checklist

-   **Firewall deployment fails:** Confirm `AzureFirewallSubnet` exists,
    is named exactly, and has a `/26` range; confirm the firewall and
    policy are in the same region.
-   **Outbound web access fails:** Check the route-table association,
    actual firewall private IP, policy rule source ranges, target FQDN
    spelling, HTTP/HTTPS protocols, and any other policy rules.
-   **RDP fails:** Check the firewall public IP, TCP `50001` DNAT
    destination port, translated VM private IP and port `3389`, source
    IP restriction, NSG, Windows Firewall, and VM status.
-   **Spoke traffic bypasses the firewall:** Inspect effective routes. A
    default route is less specific than a route for a VNet prefix; use
    explicit spoke-prefix UDRs if needed.
-   **Spoke-to-spoke traffic is blocked:** Check explicit UDRs, firewall
    network rules, peering state, forwarded-traffic settings, NSGs, and
    the return route.

## Part 12 --- Clean up

When you have finished and saved any screenshots required by your
instructor:

1.  Open **Resource groups**.
2.  Select `rg-hubspoke-lab`.
3.  Select **Delete resource group**.
4.  Type `rg-hubspoke-lab` to confirm, then delete.

Wait for deletion to finish. Deleting the resource group removes the
lab's firewall, public IP, policy, VNets, peerings, VMs, NSGs, and route
tables contained in it.

**Cost reminder:** Do not leave Azure Firewall running after the lab. It
is a billable service even when you are not actively testing.

## Completion checklist

-   [ ] Hub VNet `10.0.0.0/16` created in Central India
-   [ ] `AzureFirewallSubnet` created and Azure Firewall Standard
    deployed
-   [ ] Firewall Policy attached to the firewall
-   [ ] Spoke 1 `10.10.0.0/16` and Spoke 2 `10.20.0.0/16` created
-   [ ] One Windows Server 2022 VM deployed in each spoke, each without
    a public IP
-   [ ] Four hub/spoke peering links show Connected
-   [ ] Both spoke VM subnets have a route table with `0.0.0.0/0` →
    firewall private IP
-   [ ] DNAT rule maps firewall public IP TCP `50001` → Spoke 1 VM TCP
    `3389`
-   [ ] Standard-compatible application rule permits the selected FQDNs
    (or Premium Business category used instead)
-   [ ] Outbound access and effective routes verified
-   [ ] Resources deleted after the lab

## Further reading

-   [Hub-and-spoke network topology --- Microsoft
    Learn](https://learn.microsoft.com/en-us/azure/networking/design-guide/hub-spoke)
-   [Azure Firewall and traffic inspection --- Microsoft
    Learn](https://learn.microsoft.com/en-us/azure/networking/design-guide/azure-firewall)
-   [Azure Firewall web categories --- Microsoft
    Learn](https://learn.microsoft.com/en-us/azure/firewall/web-categories)
-   [Azure Firewall DNAT tutorial --- Microsoft
    Learn](https://learn.microsoft.com/en-in/azure/firewall/tutorial-firewall-dnat)
