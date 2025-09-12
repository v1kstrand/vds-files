here’s a tight “what we built + what you can reuse” brief you can keep for future one-page forms/projects.

# What we built (end-to-end)

* **Static form hosting**: GitHub Pages at `files.vdsai.se` (repo: `v1kstrand/vds-files`).
* **Submission endpoint**: `https://forms.vdsai.se/submit` → nginx reverse-proxies to a tiny Python relay on your OCI VM.
* **Relay → GitHub**: Python posts a `repository_dispatch` to `v1kstrand/vds-files`, which:

  * Stores each submission in `data/submissions/*.json`
  * Rebuilds `data/discovery_form.csv` on every submission (and nightly as a safety net).
* **Nice vanity URL**: `https://formular.vdsai.se` 301-redirects to the Swedish form page.
* **TLS**: certbot auto-manages certs for `forms.vdsai.se` and `formular.vdsai.se`.

---

# Reusable pieces (you can keep as-is)

1. **OCI/OS**

   * Same VM (Ubuntu 20.04) with systemd services for your GPU job **and** `formrelay`.
   * **Iptables**: explicit allows for ports 80/443 (and persisted, if you installed `iptables-persistent`).
   * **UFW**: inactive (no changes needed).

2. **VCN/Networking (public subnet)**

   * Route table: `0.0.0.0/0 → Internet Gateway`.
   * Security List ingress: TCP **80** and **443** from `0.0.0.0/0`.
   * Egress: allow all.
   * (No NSG attached; Security List path used.)

3. **Nginx**

   * Site: `forms.vdsai.se`

     * `location /submit` → proxy to `127.0.0.1:8080`
     * `location /healthz` → proxied (or you can return 200 directly)
   * Site: `formular.vdsai.se` → `return 301 https://files.vdsai.se/html/discovery_form/form-se.html;`
   * Global tweaks (optional): none required. Body size/rate-limit left out for simplicity.

4. **TLS**

   * Certbot (snap) installed; auto-renew timers in place.
   * Certificates for `forms.vdsai.se`, `formular.vdsai.se`.

5. **Python relay (at `/opt/formrelay/minimal_form_proxy.py`)**

   * Listens on `127.0.0.1:8080`.
   * Routes:

     * `GET/HEAD /healthz` → `200 ok`
     * `POST /submit` → validates `FORM_KEY`, normalizes fields (handles “Other/Annat/Övrigt”), fires `repository_dispatch`, redirects to “thanks”.
   * **GitHub payload** ≤10 top-level props (extras nested under `debug`).

6. **Systemd unit**

   * `/etc/systemd/system/formrelay.service` runs the relay as user `formrelay`.
   * Env file `/etc/formrelay.env` holds:

     * `GH_REPO_PAT=…` (Classic PAT; scope `public_repo` for public repo or `repo` if private)
     * `FORM_KEY=…` (simple tripwire, also present as hidden input in the form)

7. **GitHub repo workflows (`v1kstrand/vds-files`)**

   * `.github/workflows/store-form.json.yml`: on `repository_dispatch (form_submit)`, **save JSON + rebuild CSV** in one commit; has concurrency + rebase-push to avoid races.
   * `.github/workflows/nightly-merge.yml`: rebuild CSV nightly + on manual trigger (redundant safety).

---

# What changes per new project (minimal list)

* **Form page (HTML)**: new one-pager under `html/<new_form>/index.html`

  * `<form action="https://forms.vdsai.se/submit" method="post">`
  * Hidden inputs:

    * `name="form_key"` → same value as on server (or rotate if you want)
    * `name="redirect_to"` → where to send users after submit (your new thanks page)
  * Field **names must match** the relay:

    * `email`, `company_website` (optional)
    * `primary_goal` (+ `primary_goal_other`)
    * `role` (+ `role_other`)
    * `time` (alias of availability; + `time_other`)
    * `data_sources` (checkbox group; each uses **the same** name) (+ `data_sources_other`)
    * `outcome`
    * Make sure radios/checkboxes use value `"Other"` for the “Other/Annat” option.
  * **No JS submit interception**—only tiny show/hide for \*\_other inputs is OK.

* **Thanks page**: add or reuse one at `files.vdsai.se` and set `redirect_to` accordingly.

* **CSV columns**: already general (`submitted_at,email,goal,role,availability,data_sources,outcome,company_website`). If you add new fields you want in CSV, update the `fields = [...]` list in both workflows.

* **GitHub PAT**: reuse the same token **if you keep using the same repo**.

  * New repo? Update `OWNER/REPO` in the relay and issue a PAT with write perms for that repo.

* **Domains (optional)**:

  * Want a vanity “landing” URL? Add a new nginx server + certbot for e.g. `newname.vdsai.se` → `return 301 https://files.vdsai.se/html/<new_form>/index.html;`
  * Form **submission** keeps using `https://forms.vdsai.se/submit`.

---

# Quick clone-and-go checklist for a new one-pager

1. **Create the HTML page** in `v1kstrand/vds-files` under `html/<new_form>/index.html` (follow names above).
2. **Set hidden inputs**: correct `FORM_KEY` + `redirect_to`.
3. **Commit** → Page is live at `https://files.vdsai.se/html/<new_form>/index.html`.
4. **(Optional)** add vanity redirect domain in nginx + certbot.
5. **Submit a test** → check GitHub **Actions** (run succeeds) → new JSON in `data/submissions/` → CSV updated.
6. **Health**:

   ```bash
   curl -I https://forms.vdsai.se/healthz
   journalctl -u formrelay -n 30 --no-pager
   tail -n 50 /var/log/nginx/error.log
   ```

---

# Gotchas we already solved (remember these)

* **GitHub API 422** if `client_payload` has >10 top-level keys → we now keep essentials only.
* **Token perms**: Classic PAT with `public_repo` (or `repo`) works reliably for `repository_dispatch`.
* **Hairpin tests** from VM to its own public IP may fail—use `127.0.0.1` or test from your laptop.
* **iptables default REJECT** on Oracle images: we added explicit `ACCEPT` for 80/443.
* **HEAD /healthz** supported, so monitors and `curl -I` work.
* **Race conditions** on multiple submissions fixed with workflow `concurrency` + rebase-push.

---

# Tear-down (when the project ends)

* Disable relay: `sudo systemctl disable --now formrelay`
* Remove nginx sites (forms/formular) and reload nginx
* `certbot delete --cert-name <domain>` (optional)
* Revoke PAT (optional)
* Remove Security List 80/443 rules if the VM no longer hosts web traffic.

---

If you want, I can turn this into a one-page “runbook” you can drop into the repo (`docs/forms-runbook.md`).
