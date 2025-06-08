### Crossplane Shared API Gateway Solution Requirements

Here are the solution requirements for the Crossplane service we have designed.

***

## 1. Platform and Tooling

* **Kubernetes Cluster:** A conformant Kubernetes cluster (e.g., K3s, AKS).
* **Crossplane:** Core Crossplane installed via Helm in the `crossplane-system` namespace.
* **Crossplane Provider:** The official Upbound Azure Family provider (`upbound/provider-family-azure`) must be installed and configured with a default `ProviderConfig`.
* **Azure Credentials:** An Azure Service Principal with a dedicated custom role must be created, with its credentials stored in a Kubernetes secret (`azure-secret`) in the `crossplane-system` namespace.

---

## 2. Shared Gateway Infrastructure (`XIngress` XR)

This is a cluster-scoped composite resource that defines the foundational platform.

* **Azure Resources:**
    * **Resource Group:** A dedicated resource group to contain all shared components.
    * **VNet & Subnet:** A VNet with a dedicated subnet for API Management. The VNet's IP address space is configurable.
    * **API Management (APIM):** A single APIM instance deployed in **Internal** VNet integration mode. The SKU must be either **Premium** or **Developer** to support this.
    * **Front Door:** An **Azure Front Door Premium** profile to act as the single, global ingress point.
    * **WAF:** An Azure WAF Policy for Front Door, configured with the **OWASP 3.2** rule set in **Prevention** mode.
    * **Private Link:** The Front Door profile must be connected to the internal APIM instance via an Azure Private Link connection.
* **Configuration:**
    * The platform administrator must be able to define a unique instance name, an existing DNS Zone, Azure location, globally unique names for APIM and Front Door, and the VNet address prefix.
    * All other resource names and the APIM subnet IP range will be derived or hardcoded in the `Composition`.

---

## 3. Team Workspace / "Landing Zone" (`APITeamWorkspaceClaim`)

This is a namespace-scoped claim for application teams to request their "landing zone" on the shared platform.

* **Azure Resources / Configuration:**
    * **APIM Product:** A dedicated `Product` is created within the shared APIM instance to act as a logical workspace for the team.
    * **Front Door Custom Domain & Route:** A unique custom domain is configured on the shared Front Door. A single, catch-all (`/*`) routing rule forwards all traffic for this domain to the shared APIM instance.
    * **DNS:** A `CNAME` record is created in a pre-existing, specified public Azure DNS Zone to point the custom domain to the Front Door endpoint.
    * **RBAC:** An Azure `RoleAssignment` is created to grant a specified team Azure AD Group the "API Management Service Contributor" role, scoped to the shared APIM instance.
* **Self-Service Interface:**
    * A team must be able to request a workspace by specifying their application identifier, desired DNS prefix, APIM Product name, and their team's Azure AD Group ID.

---

## 4. API Deployment (`APIClaim`)

This is a secondary, namespace-scoped claim for application teams to deploy their APIs into their existing landing zone.

* **Azure Resources / Configuration:**
    * **APIM API:** An `API` is created within the shared APIM instance, defined by an OpenAPI Specification (YAML format) provided via a public URL.
    * **APIM Product Association:** The new `API` is automatically associated with the team's existing APIM `Product`.
    * **APIM Policy:** The claim must support attaching a custom APIM policy to the API, provided either inline or as a URL to an XML file.
    * **Backend Authentication:** The `Composition` will configure the APIM instance's System-Assigned Managed Identity to be used for authenticating to the backend service.
* **Self-Service Interface:**
    * A team must be able to deploy an API by specifying the target `Product` name, the API's base path, the URL to their OpenAPI spec, their backend service URL, and an optional policy.



## 5. Target Deployment Environment Prerequisites

* **Azure Subscription:** An active Azure subscription is required.
* **Service Principal:** A dedicated Azure Service Principal must be created with a custom role granting it the necessary permissions to manage all required resources.
* **Azure DNS Zone:**
    * A public Azure DNS Zone **must exist** before deploying the `XIngress`. This is the *only* pre-provisioned Azure resource; all other Azure resources will be created by Crossplane.
    * The domain for this zone (e.g., `myplatform.cloud`) must be properly delegated from its domain registrar to the Azure DNS name servers.
* **Network Connectivity:** The self-hosted K3s cluster must have outbound internet access to connect to the Azure APIs and the Upbound registry (`xpkg.upbound.io`).
