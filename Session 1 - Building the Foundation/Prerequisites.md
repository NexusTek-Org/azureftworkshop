# 🚀 Session 1 — Preparation Guide

You will build a grounded AI agent on **Microsoft Foundry** and then secure, evaluate and monitor it — on **your own Azure subscription**, at your own pace.

Work through this page before you start Lab 1. Every item here exists because it actually stopped someone: this session was run end to end against a live subscription, and these are the things that broke.

> ⏰ **Two items need lead time.** Model quota and Azure AI Search capacity can take **24–48 hours** to sort out. Check those first. Everything else takes minutes.

---

## ✅ What you need personally

| | Requirement |
|---|---|
| 💻 | **A laptop** with a modern browser — Edge, Chrome, Firefox or Safari |
| 🔑 | **An Azure account you can sign in to**, with an **MFA method already registered** |
| 📦 | **The lab data** — clone this repo, or download it as a ZIP from GitHub and extract it |

**Session 1 is entirely portal-based.** No Python, no VS Code, no SDK, no local dev environment. A browser is genuinely all you need.

> ⚠️ **"I can sign in" is not the same as "MFA is registered."** If your account has no authentication method enrolled, Entra interrupts with *"Let's keep your account secure"* and — once the tenant grace period has lapsed — there is **no Skip for now**.
>
> This appears **after** a correct password, and only on some token scopes, so the Foundry home page can load while every project link bounces you back to sign-in. It looks exactly like a broken lab. Visit [aka.ms/mfasetup](https://aka.ms/mfasetup) and complete enrolment first.

### Optional but handy

- **[Azure Storage Explorer](https://azure.microsoft.com/products/storage/storage-explorer)** — Lab 1 Exercise 3 uploads 20 files to a blob container. The portal handles it fine; this is faster if you already have it.
- **[Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)** — a few optional verification commands on this page use it.

---

## ✅ Azure subscription

This is **bring-your-own subscription**. Nothing is provided for you.

- **Not a free trial** — Azure AI Search is not available on free subscriptions.
- Use a **dedicated resource group** you can delete afterwards. Everything you build goes in one place, so clean-up is a single command.
- Expect to spend **well under $1** if you finish and delete the same day. See [what this costs](#-what-this-costs).

### Roles — read this one carefully

You need these on the subscription, or on the resource group you will work in:

| Role | Scope | Why |
|---|---|---|
| **Owner** | Resource group | Carlos said so |
| **Contributor** | Resource group | Create the project, models, storage and search |
| **User Access Administrator** | Resource group | **Mandatory** — you create four role assignments in Lab 1 |
| **Storage Blob Data Contributor** | Storage account | **Mandatory to upload the catalogue** — you grant this to yourself in Lab 1 Exercise 3 |

> ⚠️ **User Access Administrator is not optional.** Grounding does not work without four role assignments — three between services, one for yourself. If you cannot create role assignments you cannot finish Lab 1, and the failure appears **late**, after you have spent twenty minutes building things that look fine.
>
> If you are using a corporate subscription where you are only Contributor, ask for User Access Administrator on a single resource group. That is usually an easier request than it sounds.

> ⚠️ **Owner does not let you upload a blob.** Owner and Contributor are *control-plane* roles — they let you create a storage account but grant **no access to the data inside it**. Reading and writing blob content needs a `Storage Blob Data *` role.
>
> The portal usually hides this by falling back to the storage account key. But many tenants **disable shared-key auth by policy**, sometimes automatically within minutes of an account being created. When that happens, a subscription **Owner** gets *"You do not have the required permissions"* on the very first upload. Test it below.

### Resource providers

Register these on your subscription (**Subscription → Settings → Resource providers**):

| Provider | Needed for |
|---|---|
| `Microsoft.CognitiveServices` | Foundry project and model deployments |
| `Microsoft.Search` | Knowledge base indexing |
| `Microsoft.Storage` | The catalogue container |
| `Microsoft.OperationalInsights` | Log Analytics *(Lab 2)* |
| `Microsoft.Insights` | Application Insights *(Lab 2)* |

---

## ✅ Model quota — check this first

You need quota **in the region you plan to use**, for a chat model *and* an embedding model:

| Model | Minimum TPM |
|---|---|
| GPT-class chat model | **30K** |
| Text embedding model | **30K** |

- Check yours → [Manage Azure AI Foundry quota](https://learn.microsoft.com/azure/ai-foundry/how-to/quota)
- Request an increase → [Quota increase request](https://aka.ms/oai/quotaincrease)

> ⏳ Quota increases typically take **24 hours**. This is the item most likely to delay you, so do it first.

---

## ✅ Azure AI Search capacity — the one people miss

**Try creating an Azure AI Search resource in your chosen region, then delete it.**

Capacity in popular regions is regularly exhausted, and you only find out at creation time:

> *"This region is at capacity. Azure AI Search isn't accepting new resources in the selected region."*

When this session was built, **East US 2 and East US were both full**; West US 2 worked.

- The search resource **does not need to match your project's region**. Cross-region adds a little latency and is fine here.
- Pick a **fallback region** now so you are not hunting for one mid-lab.

---

## ✅ Anonymous blob access — check if your tenant allows it

[Lab 1 Exercise 4](./Lab%201%20-%20Build%20your%20first%20Agent%20on%20Foundry/Exercise%204%20-%20Build%20and%20test%20your%20agent.md#task-44-add-a-tool-for-live-data) publishes a small mock inventory API as **anonymously readable blobs**, so your agent has a real tool to call without you having to host an API. Many enterprise tenants **deny this by Azure Policy**.

Test it on any throwaway storage account:

```bash
az storage account update --name <anytestaccount> --resource-group <rg> --allow-blob-public-access true
```

- **Succeeds** → nothing to do.
- **Fails with a policy error** → the exercise documents a **SAS token** fallback that needs no policy change. Good to know now rather than mid-exercise.

> This is a deliberate shortcut, flagged as such in the exercise. Real inventory APIs sit behind API-key or managed-identity auth — both of which the Foundry OpenAPI tool supports.

---

## ✅ Network access

If you are on a corporate network, VPN or proxy, make sure these are reachable:

| Endpoint | Used for |
|---|---|
| `portal.azure.com` | Azure portal |
| `ai.azure.com` | Microsoft Foundry portal |
| `*.services.ai.azure.com` | Project and model endpoints |
| `*.openai.azure.com` | Model inference |
| `*.search.windows.net` | Knowledge base retrieval |
| `*.blob.core.windows.net` | Catalogue storage |
| `login.microsoftonline.com` | Sign-in |

> 🔍 Test from the **network you will actually be on**. A blocked `*.search.windows.net` looks exactly like a permissions failure, and you will spend a long time debugging the wrong thing.

---

## 💰 What this costs

You are paying for this, so it is worth knowing where the money goes. Small, but not zero:

| Resource | Billing | Rough cost |
|---|---|---|
| Chat + embedding deployments | Per token | Cents — a measured 5-turn conversation cost **$0.003** |
| Storage account (LRS) | Per GB | Negligible |
| **Azure AI Search** | Depends on SKU — see below | **The main line item** |
| Application Insights / Log Analytics *(Lab 2)* | Per GB ingested | Negligible at this volume |

A full end-to-end run of both labs, deleted the same day, costs **well under a dollar**.

> ⚠️ **Check which Search SKU you get.** Foundry may provision **serverless** ($0.24/hour while active, $0.20/GB-month storage), which is cheap when idle. A provisioned **Standard S1** bills **$0.34/hour regardless of use** — about **$248/month** — and keeps billing after you have finished. Verify with:
>
> ```bash
> az search service show --resource-group <rg> --name <search> --query sku
> ```
>
> Either way, **delete the resource group when you are done** — see [clean-up](./README.md#-clean-up).

---

## 🔎 Verify your access in five minutes

Run these before Lab 1. If all six pass, you are ready to start.

1. Sign in to the [Azure portal](https://portal.azure.com) and confirm the directory in the top-right is the one holding your subscription.
2. Open your resource group → **Access control (IAM)** → **View my access**. Confirm **Contributor** *and* **User Access Administrator**.
3. Open the [Foundry portal](https://ai.azure.com). It should load without bouncing you back to sign-in.
4. Start creating an **Azure AI Search** resource in your chosen region. If you see a capacity warning, note it and **Cancel** — do not create it.
5. Confirm you have the [lab data package](./lab-data/contoso-outdoors-catalogue/README.md) locally.
6. **Test a blob upload.** Create a throwaway storage account and container, and upload any small file through the portal. If you get *"You do not have the required permissions"*, your tenant has disabled shared-key auth — grant yourself **Storage Blob Data Contributor** on the account and try again.

    > 🔑 Step 6 is the single most likely thing to stop someone who otherwise has every role they were told to get. Worth the two minutes.

---

## ❓ If something is blocked

Almost everything here has a workaround:

| Blocker | What to do |
|---|---|
| Cannot get User Access Administrator | Ask for it on a **single resource group** rather than the subscription — a much smaller request |
| No quota in your preferred region | Move the whole session to a region that has it — nothing here is region-specific |
| AI Search at capacity | Use a different region for search only; it does not need to match your project |
| Blob upload denied despite being Owner | Grant yourself **Storage Blob Data Contributor** — Owner is control-plane only |
| Anonymous blob access denied by policy | Use the SAS-token fallback in Lab 1 Exercise 4 |
| Corporate proxy blocks endpoints | Use a personal network, or run from an Azure VM in the same subscription |
| Free-trial subscription | Azure AI Search is unavailable — you need a pay-as-you-go subscription |

---

**Ready?** Start with [Session 1 — Building the Foundation](./README.md).
