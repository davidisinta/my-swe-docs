# Namecheap → GitHub Pages: 10-Step Hero Process

A clean, happy-path checklist for connecting a **new Namecheap domain** to a **new GitHub Pages site**. Steps are ordered to avoid the most common failure (`NotServedByPagesError`) — the key trick is verifying the domain at the **account level before** you set the custom domain on the repo.

Assumptions: apex domain like `yourdomain.com`, GitHub username `youruser`, repo `your-site`.

---

## Step 1 — Register the domain on Namecheap
Buy `yourdomain.com`. After purchase, open **Domain List → Manage** and confirm under **Nameservers** that it's set to **Namecheap BasicDNS**. This is what makes the Advanced DNS host records you'll add later actually authoritative. Don't switch it to anything else.

## Step 2 — Create the repo and push your site
Create a new GitHub repo (`your-site`) and push your static files to the **`main`** branch, with `index.html` at the **root**. Public repos work on any plan; private repos need GitHub **Pro/Team/Enterprise** for Pages to be served.

## Step 3 — Enable GitHub Pages
In the repo: **Settings → Pages**. Set the source to **Deploy from a branch**, branch **`main`**, folder **`/ (root)`**, and Save. Wait for the first **"pages build and deployment"** run (Actions tab) to go green. Confirm the default `youruser.github.io/your-site/` URL loads before adding a domain.

## Step 4 — Verify the domain at the ACCOUNT level (do this first!)
Go to your **profile → Settings → Pages** (the *account* page, not the repo) → **Add a domain** → enter `yourdomain.com`. GitHub shows a TXT record:
- **Host:** `_github-pages-challenge-youruser`
- **Value:** a token string

Leave this tab open — you'll publish that record in Step 5 and click **Verify** in Step 7. Doing verification *before* binding the domain to the repo prevents the stale-association problem that causes `NotServedByPagesError`.

## Step 5 — Add all DNS records in Namecheap
In Namecheap: **Domain List → Manage → Advanced DNS → Host Records**. Add these and **Save All Changes**:

| Type | Host | Value |
|------|------|-------|
| A Record | `@` | `185.199.108.153` |
| A Record | `@` | `185.199.109.153` |
| A Record | `@` | `185.199.110.153` |
| A Record | `@` | `185.199.111.153` |
| CNAME Record | `www` | `youruser.github.io.` |
| TXT Record | `_github-pages-challenge-youruser` | *(the token from Step 4)* |

**Gotchas:** Host is just the prefix (no `.yourdomain.com` — Namecheap appends it). No quotes around the TXT value. TTL "Automatic" ≈ 30 min; set it to 1 min if you want faster propagation.

## Step 6 — Confirm DNS is resolving
Give it a few minutes, then check from a terminal:
```bash
dig +short yourdomain.com          # → the four 185.199.x IPs
dig +short www.yourdomain.com      # → resolves to GitHub
dig +short TXT _github-pages-challenge-youruser.yourdomain.com
```
Don't move on until the apex shows all four IPs and the TXT token appears.

## Step 7 — Verify ownership on GitHub
Back on the account-level Pages tab, click **Verify**. If it says "couldn't find the TXT record," that's just propagation — wait ~10–30 min and retry. Once green, `yourdomain.com` is locked to your account.

## Step 8 — Set the custom domain on the repo
Now go to the **repo's** Settings → Pages → **Custom domain**, enter `yourdomain.com`, and Save. This auto-creates a **`CNAME` file** in your repo root containing `yourdomain.com` — confirm it appears in the Code tab. The DNS check should pass within minutes (it has a verified domain + correct DNS to work with).

## Step 9 — Wait for HTTPS, then enforce it
After the DNS check passes, GitHub provisions a **Let's Encrypt TLS certificate** automatically. This can take a few minutes up to ~24 hours. When the **Enforce HTTPS** checkbox stops being greyed out, tick it so the site is served only over `https://`.

> If you set CAA records on the domain, make sure one allows `letsencrypt.org`, or the cert can't be issued.

## Step 10 — Final verification & maintenance
Open `https://yourdomain.com` in an **incognito** window (avoids cached failures). Check both apex and `www` redirect correctly. Then keep these in mind:

- **If your site has a build step**, the `CNAME` file must survive into the *published output*, not just the source — a build can wipe it from the root.
- **Don't set a `baseurl`/`base`/`homepage` to `/repo-name/`** — once served from the domain root, those paths 404. Use `/`.
- **Renaming the repo, or flipping visibility**, can unlink the custom domain (GitHub's takeover protection). Account-level verification from Step 4 is your insurance against this.
- **Don't delete the four A records or the www CNAME** when adding other records — TXT/A/CNAME coexist fine.

---

### Quick troubleshooting reference

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `NotServedByPagesError`, DNS is correct | Stale/unverified domain↔repo association | Account-level verify (Step 4/7), then re-add custom domain |
| "DNS check unsuccessful" right after adding records | Propagation not finished | Wait for TTL (~30 min), click Check again |
| Enforce HTTPS greyed out | Cert not provisioned yet | Wait (up to 24h) after DNS check passes |
| Site loads blank / unstyled | Build base path points to `/repo-name/` | Set base path to `/`, redeploy |
| Pages won't serve at all | Private repo on Free plan | Make repo public, or upgrade to Pro |

**The single most important ordering insight:** verify the domain to your account *before* binding it to the repo. Most "correct DNS but still broken" cases come from skipping that step.