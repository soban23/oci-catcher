# OCI Free ARM Server Catcher

Oracle Cloud gives you a free ARM server (Ampere A1), but creating one usually fails with
**"Out of host capacity"**. This repo uses GitHub Actions to keep trying for you, every
90 seconds, around the clock, until it works. Your computer can be off.

It costs nothing: GitHub Actions is free for public repos, and failed attempts aren't billed by Oracle.

> **Free limits (as of late 2026):** 2 ARM OCPUs and 12 GB RAM total, 200 GB of disk total.
> Older guides say 4 OCPU / 24 GB. That's outdated. Go over the limit and Oracle may
> disable your instance (or bill you, on a paid account).

---

## What you need

- An Oracle Cloud account (free tier is fine)
- A GitHub account
- About 15 minutes

Everything below happens in your browser.

---

## Step 1: Create a network (once)

1. In the Oracle console, search **Virtual cloud networks** in the top search bar.
2. Make sure the compartment on the left is your **root** compartment (your account name).
3. Click **Actions → Start VCN Wizard → Create VCN with Internet Connectivity → Start VCN Wizard**.
4. Type any name (e.g. `free-vcn`), leave the rest as is, click **Next → Create**.

This is free and opens port 22 so you can SSH in later.

## Step 2: Open Cloud Shell

Click the **Developer tools** icon in the top bar (small monitor icon, left of the bell) → **Cloud Shell**.
A terminal opens at the bottom. You'll paste commands here.

## Step 3: Make an SSH key (to log in to your server later)

Paste this in Cloud Shell and press Enter:

```bash
ssh-keygen -t ed25519 -f ~/oci_key -N "" && cat ~/oci_key.pub
```

Then download the private key to your PC: Cloud Shell **Menu → Download** → type `oci_key` → Download.
Keep it safe. You need it to log in, and it can't be recovered.

> Already have a key? Upload your `.pub` file via **Menu → Upload** and use that instead.

## Step 4: Collect your values

Paste this in Cloud Shell:

```bash
echo "REGION:      $(oci iam region-subscription list --query 'data[?"is-home-region"]|[0]."region-name"' --raw-output)"
echo "OCI_TENANCY: $OCI_TENANCY"
echo "OCI_AD:      $(oci iam availability-domain list -c $OCI_TENANCY --query 'join(`,`, data[].name)' --raw-output)"
echo "OCI_IMAGE:   $(oci compute image list -c $OCI_TENANCY --operating-system 'Canonical Ubuntu' --operating-system-version '24.04' --shape VM.Standard.A1.Flex --sort-by TIMECREATED --query 'data[0].id' --raw-output)"
echo "SUBNETS:"; oci network subnet list -c $OCI_TENANCY --query 'data[].["display-name", id]' --output table
```

Copy the output into a notepad. For the subnet, use the ID of the one with **public** in its name.

> Want a different OS? Replace `'Canonical Ubuntu'` / `'24.04'` with e.g. `'Oracle Linux'` / `'9'`.

## Step 5: Make an API key (lets GitHub talk to Oracle)

1. Click your profile icon (top-right) → **My profile** → **Tokens and keys** → **Add API key**.
2. Choose **Generate API key pair** → **Download private key** → **Add**.
3. A config preview appears. Copy the `user=` and `fingerprint=` values.

This key has full access to your Oracle account. Only put it in GitHub Secrets, and delete it when you're done (see "When it works").

## Step 6: Set up this repo

1. Click **Use this template** or **Fork** (or create a new **public** repo and copy in
   `.github/workflows/oci-a1.yml`).
2. Go to **Settings → Secrets and variables → Actions → New repository secret** and add these 8:

| Secret | Where it comes from | Looks like |
|---|---|---|
| `OCI_USER` | `user=` from Step 5 | `ocid1.user.oc1..aaaa...` |
| `OCI_FINGERPRINT` | `fingerprint=` from Step 5 | `12:ab:34:...` |
| `OCI_TENANCY` | Step 4 | `ocid1.tenancy.oc1..aaaa...` |
| `OCI_KEY` | Open the `.pem` from Step 5 in Notepad, copy **everything** | `-----BEGIN PRIVATE KEY-----` ... `-----END PRIVATE KEY-----` |
| `OCI_AD` | Step 4 (paste the whole line, commas included) | `abCd:AP-MUMBAI-1-AD-1` |
| `OCI_IMAGE` | Step 4 | `ocid1.image.oc1.ap-mumbai-1.aaaa...` |
| `OCI_SUBNET` | Step 4, the **public** subnet | `ocid1.subnet.oc1.ap-mumbai-1.aaaa...` |
| `SSH_PUBKEY` | The line printed in Step 3 (or your `.pub` file) | `ssh-ed25519 AAAA...` |

Paste each value exactly, with no extra spaces or quotes.

## Step 7: Adjust settings

Open `.github/workflows/oci-a1.yml`, click the pencil icon, and edit the **SETTINGS** block:

| Setting | What to put |
|---|---|
| `REGION` | Your home region from Step 4, e.g. `ap-mumbai-1`, `eu-frankfurt-1`, `us-ashburn-1` |
| `OCPUS` / `MEM` | `2` / `12` for the max free size. `1` / `6` gets through more often; you can resize later |
| `BOOT_GB` | Disk size, 47 to 200. All your volumes together must stay under 200 GB |
| `NAME` | Any instance name |
| `WAIT` | Seconds between tries. Don't go below 60 or Oracle rate-limits you |

Commit the change.

## Step 8: Run it

Go to **Actions** (click **Enable** if asked) → **oci-a1-catcher** → **Run workflow**.

Open the run → **launch** → **Try to launch until capacity is found**. You should see:

```
attempt 1 (...AD-1): no capacity, retrying in 90s
```

That means it's working. Close everything. It restarts itself every 6 hours and keeps going.

How long it takes depends on your region: anywhere from minutes to a few weeks.

---

## When it works

The log says **SUCCESS** and the run gets a green ✓. Then:

1. **Disable the workflow:** Actions → oci-a1-catcher → **⋯** → **Disable workflow**.
2. **Delete the API key:** Oracle → My profile → Tokens and keys → delete the key from Step 5.
3. **Find your IP:** Oracle → Compute → Instances → your instance → **Public IP address**.
4. **Log in** from your PC (in the folder with `oci_key`):
   ```bash
   ssh -i oci_key ubuntu@YOUR_IP
   ```
   (Use `opc@` instead of `ubuntu@` for Oracle Linux.)

The workflow also checks for an existing instance with the same `NAME` and won't create a second one.

---

## Troubleshooting

| Log message | Fix |
|---|---|
| `Missing secret: X` | You didn't add secret `X`, or misspelled its name. |
| `rate limited, waiting 5 min` | Normal now and then. If it keeps happening, make sure nothing else (Cloud Shell, another script) is also trying, or raise `WAIT`. |
| `NotAuthenticated` | `OCI_USER`, `OCI_FINGERPRINT`, `OCI_TENANCY` or `OCI_KEY` is wrong, or `REGION` isn't your home region. Recreate the API key and copy again. |
| `Invalid subnetId` / `NotAuthorizedOrNotFound` on subnet | Wrong subnet ID, or the network is in a different region or compartment. Redo Steps 1 and 4. |
| `LimitExceeded` | You already use your free A1 quota (another instance exists), or `OCPUS`/`MEM` is over the limit. |
| `InvalidParameter` on image | The image doesn't match the shape (ARM needs an aarch64 image). Redo the image command in Step 4. |
| Log stops updating | Refresh the page. GitHub's live log lags sometimes. |

---

## Good to know

- **Home region only.** Free resources only work in the region you picked at signup. It can't be changed.
- **Idle servers can be reclaimed.** Oracle may take back free instances that sit almost unused for a week. Run something real on it.
- **Public repo is fine.** Secrets stay encrypted and hidden from logs and forks. Private repos only get limited free minutes, and this would use them up in a few days.
- **Paid (Pay-As-You-Go) accounts** get capacity much faster, but anything above the free limits is billed. Set a budget alert first (Billing → Budgets).
