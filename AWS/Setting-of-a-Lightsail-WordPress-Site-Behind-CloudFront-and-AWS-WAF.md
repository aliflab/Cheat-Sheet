# Guide: Putting a Lightsail WordPress Site Behind CloudFront and AWS WAF

Oct 20, 2026 · @ALIF

## Overview

Following this guide puts a WordPress site on AWS Lightsail behind a CloudFront distribution protected by the shared AWS WAF protection pack, with HTTPS, cached static files and no redirect loops. Every step was tested end to end on a live site in October 2026.

- **Time needed:** about 2 to 3 hours, most of it waiting for certificate validation and CloudFront deployments.
- **Downtime:** none if you follow the order. The site only switches to CloudFront in Step 6, after it has been tested.
- **Placeholders used throughout:**

| Placeholder | Meaning | Replace with |
| --- | --- | --- |
| `yoursite.com` | The site's main domain | Your real domain |
| `www.yoursite.com` | The www version | Your real www name |
| `yoursite.net` | Any extra domain that should reach the same site | Your extra domain, if any |
| `origin.yoursite.com` | New DNS name CloudFront uses to reach the server | `origin.` + your main domain |
| `203.0.113.10` | The Lightsail static IP | Your instance's static IP |
| `E1EXAMPLE123` | The new CloudFront distribution ID | The ID CloudFront gives you |
| `d111111abcdef8.cloudfront.net` | The new distribution's domain name | The name CloudFront gives you |

Work through Steps 1 to 8 in order. Steps 2 and 3 can run in parallel while you wait for the certificate.

## How it works

AWS WAF cannot attach to a Lightsail instance, so each site gets a CloudFront distribution in front of it and the shared WAF pack protects the distribution.

&#91;embedded content: request path · 5 parts, WAF at the CloudFront edge\]

Public hostnames resolve to CloudFront, where the WAF checks every request. Cached images, CSS and JavaScript are served from the edge; everything else travels over HTTP to `origin.yoursite.com`, the only name pointing at the server.

## Before you start

Gather access and site details first; most problems later come from a missing piece here.

**Access you need**

- [ ] AWS console access with rights to Lightsail, Route 53, Certificate Manager (ACM), CloudFront and WAF & Shield.
- [ ] SSH access to the Lightsail server with `sudo` (Lightsail console → instance → **Connect using SSH**, or your own SSH client).
- [ ] A WordPress admin login for the site, to test after the switch.
- [ ] Windows PowerShell (or any terminal) on your PC, for the `curl.exe` and `nslookup` tests.

**Things that must already exist**

- [ ] The Lightsail instance is running and has a **static IP** attached (Lightsail → **Networking** tab). A normal public IP changes on restart and would break the origin record.
- [ ] Every domain for the site has a **hosted zone in Route 53** in the same AWS account.
- [ ] The shared WAF protection pack **Lightsail\_Instances** exists (WAF & Shield → Protection packs). Do not create a new one.

**Site details worksheet** (fill this in before Step 1)

| Item | How to find it | Your value |
| --- | --- | --- |
| Main domain | The address visitors use |  |
| All hostnames to serve | Main domain + www, plus any extra domains that should show the same site |  |
| Lightsail static IP | Lightsail → instance → Networking |  |
| Current main-domain A record | Route 53 → hosted zone → the domain's A record value |  |
| Web server type | Run `apache2 -v` (Debian/Ubuntu) or check for `/opt/bitnami` (Bitnami image) |  |
| Existing CloudFront distributions using these domains | CloudFront → Distributions → search the domain name |  |
| Extra domains that already redirect elsewhere | Run `curl.exe -sI https://yoursite.net` and check for `301` + `Server: AmazonS3` |  |

Write down the **current main-domain A record value** (for example `203.0.113.10`). It is your rollback if anything goes wrong in Step 6.

## Step 1: Create the origin DNS record

Create `origin.yoursite.com` pointing straight at the server; CloudFront will fetch the site through this name. Nothing changes for visitors yet.

1. Open **Route 53 → Hosted zones → yoursite.com**.
2. Click **Create record**.
3. Fill in:
   - **Record name:** `origin`
   - **Record type:** A
   - **Alias:** off
   - **Value:** `203.0.113.10` (the static IP)
   - **TTL:** 300
   - **Routing policy:** Simple
4. Click **Create records**. Route 53 shows a green banner; the record is live within about a minute.
5. Check it from PowerShell:

```
nslookup origin.yoursite.com
```

The answer must show `203.0.113.10`.

If `origin.yoursite.com` already exists, check it points at the static IP and move on.

## Step 2: Prepare the server so it trusts CloudFront

The server must stop redirecting requests that come from CloudFront, or the site loops with **ERR\_TOO\_MANY\_REDIRECTS**. Do this before the switch; it does not affect current visitors.

**Why:** CloudFront reaches the server over plain HTTP. A server set up with Let's Encrypt (Certbot) redirects every HTTP request to HTTPS, and WordPress does the same when it thinks the visitor is on HTTP. CloudFront then fetches the redirect, the browser follows it back to CloudFront, and the cycle repeats. The fix is to let both Apache and WordPress read the **`CloudFront-Forwarded-Proto`** header, which CloudFront sends when the origin request policy in Step 4 is used.

Do not rely on an `X-Forwarded-Proto` custom origin header for this. In testing, CloudFront did not deliver it to the server even when configured.

### 2a. Find Apache's HTTP-to-HTTPS redirect

1. Connect to the server by SSH.
2. List the enabled site files and search them. Use capital `-R`: the files in `sites-enabled` are shortcuts, and lowercase `-r` skips them.

```bash
ls -la /etc/apache2/sites-enabled/
sudo grep -RniE "rewrite|redirect|DocumentRoot" /etc/apache2/sites-enabled/ /etc/apache2/conf-enabled/
```

On a Bitnami image, search `/opt/bitnami/apache/conf/` instead.

3. Check the site's `.htaccess`, using the `DocumentRoot` path from the output (usually `/var/www/html`):

```bash
sudo grep -niE "rewrite|redirect|https" /var/www/html/.htaccess
```

4. Identify the line that matters. It sits in the **port 80** site (`<VirtualHost *:80>`, usually `000-default.conf`) and looks like this Certbot block:

```apache
RewriteEngine on
RewriteCond %{SERVER_NAME} =www.yoursite.com [OR]
RewriteCond %{SERVER_NAME} =yoursite.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
```

You can ignore a `Redirect permanent` line in the HTTPS site (`000-default-le-ssl.conf`, port 443), the standard WordPress rules in `.htaccess`, and commented lines starting with `#`.

### 2b. Make the redirect skip CloudFront requests

1. Back up the real file (the shortcut points into `sites-available`), then open it:

```bash
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/000-default.conf.bak
sudo nano /etc/apache2/sites-available/000-default.conf
```

2. Add **one line** straight after `RewriteEngine on`, so the block reads:

```apache
RewriteEngine on
RewriteCond %{HTTP:CloudFront-Forwarded-Proto} !https
RewriteCond %{SERVER_NAME} =www.yoursite.com [OR]
RewriteCond %{SERVER_NAME} =yoursite.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
```

Make sure the original lines are not still there underneath; a leftover copy keeps the loop going.

3. If the port 80 site has a single `Redirect permanent / https://yoursite.com/` line instead, replace that line with:

```apache
RewriteEngine on
RewriteCond %{HTTP:CloudFront-Forwarded-Proto} !https
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [END,NE,R=permanent]
```

Then enable the rewrite module if needed: `sudo a2enmod rewrite`.

4. Save in nano with **Ctrl+O**, **Enter**, **Ctrl+X**.
5. Check the edit landed cleanly:

```bash
sudo grep -n "SERVER_NAME\|Forwarded" /etc/apache2/sites-available/000-default.conf
```

You should see one `CloudFront-Forwarded-Proto` line, the `SERVER_NAME` lines and one `RewriteRule`, with plain domain names.

6. Test the configuration and reload:

```bash
sudo apachectl configtest
sudo systemctl reload apache2
```

Only reload after **Syntax OK**. A message `AH00558: Could not reliably determine the server's fully qualified domain name` is harmless. If the test fails, restore the backup with `sudo cp …000-default.conf.bak …000-default.conf`.

Visitors who reach the server directly over HTTP are still redirected to HTTPS; only requests marked as HTTPS by CloudFront skip the redirect.

### 2c. Make WordPress trust CloudFront

1. Back up `wp-config.php` and check what is already there:

```bash
sudo cp /var/www/html/wp-config.php /var/www/html/wp-config.php.bak
sudo grep -nE "HTTPS|FORWARDED|wp-settings" /var/www/html/wp-config.php
```

2. Open it with `sudo nano /var/www/html/wp-config.php` and add this **above** the line `require_once ABSPATH . 'wp-settings.php';` (if a block already checks only `HTTP_X_FORWARDED_PROTO`, replace it with this one):

```php
if ((isset($_SERVER['HTTP_CLOUDFRONT_FORWARDED_PROTO']) && $_SERVER['HTTP_CLOUDFRONT_FORWARDED_PROTO'] === 'https')
    || (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false)) {
    $_SERVER['HTTPS'] = 'on';
}
```

3. Check for PHP errors:

```bash
sudo php -l /var/www/html/wp-config.php
```

It must say **No syntax errors detected**; otherwise restore `wp-config.php.bak`. No Apache reload is needed.

Leave the WordPress Address and Site Address as `https://yoursite.com`. Do **not** change them to `origin.yoursite.com`; that would point every link on the site at the origin name.

### 2d. Test the server as CloudFront will reach it

Run from PowerShell (type `curl.exe`, not `curl`):

```
curl.exe -sI http://origin.yoursite.com -H "Host: yoursite.com" -H "CloudFront-Forwarded-Proto: https"
```

| Result | Meaning |
| --- | --- |
| `HTTP/1.1 200 OK`, `charset=UTF-8` | Ready. Continue. |
| `301`, `charset=iso-8859-1`, no `X-Redirect-By` | Apache still redirects. Recheck 2a and 2b. |
| `301` with `X-Redirect-By: WordPress` | WordPress still redirects. Recheck 2c, especially the snippet's position. |

## Step 3: Request the SSL certificate in us-east-1

Request one certificate that lists every hostname the distribution will serve, in **US East (N. Virginia)**. CloudFront only uses certificates from that region; one requested in Sydney never appears in CloudFront's list.

1. Open **Certificate Manager**. In the top-right region selector, choose **United States (N. Virginia) us-east-1**.
2. Click **Request** → **Request a public certificate** → **Next**.
3. Under **Fully qualified domain name**, enter `yoursite.com`. Click **Add another name to this certificate** for each other hostname, for example `www.yoursite.com`.
   - Include every name visitors use. A name missing here cannot be added to the distribution later without a new certificate.
   - Leave out extra domains that already redirect elsewhere (see Step 6c).
4. Leave the defaults: **Allow export:** Disable export; **Validation method:** DNS validation; **Key algorithm:** RSA 2048.
5. Click **Request**. The certificate opens with status **Pending validation**.
6. In the **Domains** table:
   - Rows already showing **Success** need nothing; a matching validation record already exists from an earlier certificate.
   - For rows showing **Pending validation**, click **Create records in Route 53**, tick those domains, then **Create records**. ACM creates each record with the correct name in the correct hosted zone.
7. If you create a record by hand instead, copy the **CNAME name** and **CNAME value**, then in **Route 53 → that domain's hosted zone → Create record**:
   - **Record type:** CNAME; **Value:** the CNAME value.
   - **Record name:** the CNAME name with only the zone's domain removed from the end. For `_abc123.www.yoursite.com.` type `_abc123.www`.
   - Removing too much (dropping the `www` part) or too little (doubling the domain) leaves validation stuck.
8. Check a record is live from PowerShell:

```
nslookup -type=CNAME _abc123.www.yoursite.com
```

A `canonical name = …acm-validations.aws` answer means the record is correct; ACM then confirms it on its own schedule.

9. Refresh the certificate page every 10 minutes. It usually shows **Success** on every row and status **Issued** within 5 to 30 minutes.

Keep every `_…` validation CNAME in Route 53 permanently. ACM uses them to renew the certificate automatically each year; deleting them breaks the site's HTTPS at renewal.

## Step 4: Create the CloudFront distribution

The Create distribution wizard has six screens; the choices below give a pay-as-you-go distribution attached to the shared firewall, with nothing cached yet. Start at **CloudFront → Distributions → Create distribution**.

### Screen 1: Choose a plan

Select **Pay-as-you-go**. Do **not** choose the Free plan or another flat-rate plan:

- A plan creates its own web ACL; you cannot attach the shared **Lightsail\_Instances** pack, and the web ACL cannot be replaced later.
- The Free plan allows only 5 WAF rules and no SQL injection rule set.
- Each plan covers one apex domain only.

If the Security screen later says WAF is "included in your plan", go back to this screen and switch to Pay-as-you-go.

### Screen 2: Get started

Enter a distribution name (for example `yoursite.com`) and, when asked for the domain to serve, the main domain `yoursite.com`. Other hostnames are added in Step 5.

### Screen 3: Specify origin

1. **Origin type:** **Other** ("Refer to any AWS or non-AWS origin through its publicly resolvable URL"). Not Amazon S3, not VPC origin.
2. **Origin domain:** `origin.yoursite.com`. **Origin path:** leave blank.
3. **Origin settings:** choose **Customize origin settings**, then set:

| Setting | Value |
| --- | --- |
| Enable origin mutual TLS | Off |
| Add custom header | None needed |
| Origin Shield | Off |
| Protocol | **HTTP only** |
| HTTP port | **80** |
| Connection attempts | 3 |
| Connection timeout | 10 |
| Response timeout | 30 |
| Keep-alive timeout | 5 |
| Response completion timeout | Off |
| Origin IP address type | IPv4-only |

4. **Cache settings:** choose **Customize cache settings**, then set:

| Setting | Value |
| --- | --- |
| Viewer protocol policy | **Redirect HTTP to HTTPS** |
| Allowed HTTP methods | **GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE** |
| Cache HTTP methods: OPTIONS | Unticked |
| Allow gRPC requests over HTTP/2 | Unticked |
| Cache policy | **CachingDisabled** |
| Origin request policy | **AllViewerAndCloudFrontHeaders-2022-06** |
| Response headers policy | Leave empty |

The origin request policy is the one setting that prevents the redirect loop: it makes CloudFront send `CloudFront-Forwarded-Proto` to the server. Plain **AllViewer** does not. Ignore the blue tip about query strings and do not click **Update my cache policy**.

5. Click **Next**.

### Screen 4: Enable security

1. Select **Enable security protections**.
2. Switch on **Use existing WAF configuration**.
3. In **Choose a web ACL**, select **Lightsail\_Instances**.
4. Leave monitor mode, rate limiting, SQL protections and Layer 7 DDoS unticked. Those build a separate new web ACL with its own cost; the shared pack already covers them.
5. Click **Next**.

If Lightsail\_Instances is not listed, select **Do not enable security protections** and attach the distribution after creation (Step 5e).

### Screen 5: Get TLS certificate

Select the certificate from Step 3. Its **Covered domains** must list every hostname. Do not choose **Create a new certificate** unless Step 3 was skipped. Click **Next**.

### Screen 6: Review and create

Check these before clicking **Create distribution**:

- [ ] Billing: **Pay-as-you-go**
- [ ] Origin: Custom origin `origin.yoursite.com`, Origin Shield No
- [ ] Cache: Redirect HTTP to HTTPS, all seven methods, CachingDisabled, AllViewerAndCloudFrontHeaders-2022-06
- [ ] Security: Lightsail\_Instances (WAFv2), Use existing WAF configuration: Yes
- [ ] TLS certificate: the us-east-1 certificate covering all hostnames

After creation, note the **distribution ID** (for example `E1EXAMPLE123`) and **distribution domain name** (for example `d111111abcdef8.cloudfront.net`). **Last modified** shows **Deploying** for several minutes; you can keep working in Step 5 meanwhile.

## Step 5: Finish the distribution

Add the remaining hostnames, the WordPress behaviours and static-file caching, then confirm the firewall is attached. All of this happens on the new distribution, **E1EXAMPLE123**.

### 5a. Add every hostname and the certificate

1. Open **CloudFront → Distributions → E1EXAMPLE123 → General tab → Settings → Edit**.
2. Under **Alternate domain name (CNAME)**, click **Add item** for each hostname not yet listed (for example `www.yoursite.com`).
3. Under **Custom SSL certificate**, select the Step 3 certificate. Check its ID, not just its name: older certificates for the same domain can appear with the same name.
   - The link under the dropdown shows only the certificate's main domain. That is normal; check the full list in ACM if unsure.
4. Leave **Legacy clients support** unticked and **Security policy** on **TLSv1.2\_2021 (recommended)**.
5. Click **Save changes**.

If saving fails:

- **"The certificate … doesn't cover the alternate domain name"**: the wrong certificate is selected, or the certificate is missing a name. Request a new one (Step 3) and select it.
- **"One or more of the CNAMEs you provided are already associated with a different resource"**: another distribution holds that hostname. Search **Distributions** for the domain and check that distribution's **Origins** column:
  - Origin is the old server or an old copy of the site: open it → **General → Edit** → **Remove** the hostname → **Save changes**, or **Disable** it. Then save the new distribution again.
  - Origin is an S3 website bucket (`…s3-website-…amazonaws.com`): it is a redirect for that domain. Leave it alone and leave that domain off the new distribution (see Step 6c).

### 5b. Check the origin

1. Open the **Origins** tab, select `origin.yoursite.com-…` (CloudFront adds a random suffix to the name), click **Edit**.
2. Confirm **Protocol: HTTP only** and **HTTP port: 80**. Click **Cancel** if nothing needs changing.

### 5c. Get the static-file cache policy

One cache policy serves every site in the account. Open **CloudFront → Policies → Cache** and look under **Custom policies**:

- If **WP-Cache-With-Query** exists, open it and confirm the values in the table below. If they match, use it.
- Otherwise click **Create cache policy**, enter the values below, and click **Create**.

| Setting | Value |
| --- | --- |
| Name | `WP-Static-Files` (or reuse WP-Cache-With-Query) |
| Minimum / Default / Maximum TTL | 1 / 86400 / 31536000 seconds |
| Headers | None |
| Cookies | None |
| Query strings | **All** |
| Gzip / Brotli | Both ticked |

Query strings must be **All**: WordPress adds `?ver=` to CSS and JavaScript files, so updated files reach visitors immediately.

### 5d. Create the five behaviours

On the **Behaviors** tab, click **Create behavior** once per row of the table below. Every behaviour uses these shared settings:

- **Origin and origin groups:** `origin.yoursite.com-…` (the only option)
- **Compress objects automatically:** Yes
- **Viewer protocol policy:** Redirect HTTP to HTTPS
- **Restrict viewer access:** No
- **Cache key and origin requests:** Cache policy and origin request policy (recommended)
- **Origin request policy:** **AllViewerAndCloudFrontHeaders-2022-06**
- **Response headers policy** and **Function associations:** leave empty

| Path pattern | Allowed HTTP methods | Cache policy | Purpose |
| --- | --- | --- | --- |
| `/wp-admin/*` | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE | CachingDisabled | Admin area, always live |
| `/wp-login.php` | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE | CachingDisabled | Login page |
| `/wp-admin/admin-ajax.php` | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE | CachingDisabled | Admin and plugin requests |
| `/wp-content/*` | GET, HEAD | **WP-Cache-With-Query** (or WP-Static-Files) | Images, theme CSS and JS, uploads |
| `/wp-includes/*` | GET, HEAD | **WP-Cache-With-Query** (or WP-Static-Files) | WordPress core CSS and JS |

The admin behaviours need all seven methods; with GET and HEAD only, logging in and saving fail. Custom cache policies sit under **Custom** in the dropdown, below the managed ones.

When finished, the Behaviors tab shows six rows with **Default (\*)** last. Check two columns:

- **Cache policy name:** WP-Cache-With-Query on exactly the two static rows; CachingDisabled on the other four.
- **Origin request policy name:** Managed-AllViewerAndCloudFrontHeaders-2022-06 on **every** row. Any row on plain AllViewer brings the redirect loop back for that path.

### 5e. Confirm the firewall is attached

1. Open the distribution's **Security** tab. It should show **Lightsail\_Instances (WAF v2)** as enabled.
2. If it does not, open **WAF & Shield → Protection packs (web ACLs)**, set **Region scope** to **CloudFront (Global) and Regional**, click **Lightsail\_Instances → Manage resources → Add global resources**, select the distribution and save.

Wait until **Last modified** shows a date instead of **Deploying** before Step 6.

## Step 6: Switch DNS to CloudFront

The switch is one record edit per domain, and it can be undone in a minute. Test through CloudFront first so the switch holds no surprises.

### 6a. Test through CloudFront before switching

This sends a request to CloudFront as if DNS were already switched, without affecting visitors.

1. Find one of CloudFront's addresses for the new distribution:

```
nslookup d111111abcdef8.cloudfront.net
```

2. Copy one IPv4 address from the answer (for example `198.51.100.7`) and run:

```
curl.exe -sI --resolve yoursite.com:443:198.51.100.7 https://yoursite.com/
```

3. You want `HTTP/1.1 200 OK` with `Via: … (CloudFront)`. A `301` back to the same URL means a loop: return to Step 2 and check the origin request policy in Step 5d. A `403` means the hostname is missing from the distribution (Step 5a).

### 6b. Point the main domain and www at CloudFront

1. Open **Route 53 → Hosted zones → yoursite.com**.
2. Select the `yoursite.com` record of type **A** → **Edit record**.
3. Switch the **Alias** toggle **on** (just under Record type). The Value box is replaced by **Route traffic to**.
4. Choose **Alias to CloudFront distribution**, then pick `d111111abcdef8.cloudfront.net` from the list.
5. Leave Routing policy **Simple** and click **Save**. The record now shows **Alias: Yes**.

Typing the cloudfront.net name into the Value box with Alias off fails with "Value is not a valid IPv4 address". Turning Alias on is the fix.

6. Handle `www.yoursite.com` according to what it is now:

| Current www record | Action |
| --- | --- |
| CNAME → `yoursite.com` | Leave it. It follows the main domain to CloudFront. |
| A record → an IP address | Edit it like steps 2 to 5 (Alias on, same distribution). |
| None | **Create record**: name `www`, type A, Alias on, same distribution. |
| CNAME → anything else | Change its value to `yoursite.com`. |

www works only if `www.yoursite.com` is on the certificate and in the distribution's alternate domain names (Step 5a). If it is not, www visitors get an error as soon as the main domain switches.

### 6c. Extra domains

First check what each extra domain does today:

```
curl.exe -sI https://yoursite.net
```

- **Answer is `301` to `https://yoursite.com/` with `Server: AmazonS3`:** it is an S3 redirect behind its own distribution. Leave its records, bucket and distribution unchanged; its visitors land on `yoursite.com` and go through CloudFront from there.
- **It should show the same site directly:** add `yoursite.net` and `www.yoursite.net` to the certificate and the alternate domain names, then in **Route 53 → yoursite.net** make both the apex and www **A records with Alias on** to the same distribution. WordPress will then serve the site under that domain too.

### 6d. Records to leave alone

Do not change MX, TXT (SPF, DMARC, site verification), `_domainkey`, `selector1/2`, `autodiscover`, NS, SOA or any `_…` certificate validation CNAME. They run email and domain verification.

### Rollback

If the site breaks after 6b, edit the main domain's A record, switch **Alias off**, set the value back to the IP you recorded in Before you start (for example `203.0.113.10`), and save. Visitors go straight to the server again within a few minutes while you fix the problem.

## Step 7: Test and verify

The setup is done when every hostname answers through CloudFront, admin works and static files are cached. Wait a few minutes after Step 6 for DNS to update.

### 7a. Every hostname from the command line

```
curl.exe -sI https://yoursite.com
curl.exe -sI https://www.yoursite.com
curl.exe -sI https://yoursite.net
```

| Result | Meaning |
| --- | --- |
| `200 OK` with `Via: … (CloudFront)` and `X-Cache: … from cloudfront` | Working through CloudFront and the firewall. |
| `301` to `https://yoursite.com/` from another hostname, with `X-Redirect-By: WordPress` or `Server: AmazonS3` | Working: visitors are sent to the main address. |
| `301` whose `Location` is the **same** URL | Redirect loop. See Troubleshooting. |
| `403` "The request could not be satisfied" | Hostname missing from the distribution, or DNS not updated yet. |
| Certificate or SSL error | Wrong certificate attached (Step 5a). |
| No `Via: … CloudFront` header | DNS still points at the server. Check Step 6b. |

### 7b. In a browser

- [ ] Open each hostname in a **new** InPrivate or incognito window and click through several pages.
- [ ] Log in at `https://yoursite.com/wp-admin`, edit a page, save, and check the change appears on the live site.
- [ ] Submit any contact or booking form on the site and confirm it arrives.

A browser that saw a redirect loop can remember it for up to an hour. Always test in a fresh private window.

### 7c. Static files are cached

1. In the browser press **F12 → Network**, then reload the page twice.
2. Click any `.css`, `.js` or image file under `/wp-content/`.
3. Under **Response Headers**, look for `X-Cache: Hit from cloudfront`. `Miss` on the first load is normal; reload once more.

Pages themselves always show `Miss`; they are deliberately not cached.

### 7d. Firewall and certificate

- [ ] Distribution **Security** tab shows **Lightsail\_Instances (WAF v2)** with request charts filling in.
- [ ] **WAF & Shield → Lightsail\_Instances → Manage resources** lists the new distribution.
- [ ] In **ACM (us-east-1)**, the new certificate shows **In use: Yes**.

## Step 8: Clean up

Remove what the old setup left behind, but only after Step 7 passes and has run for a day.

**Old distributions for this site** (not S3 redirect distributions):

1. In **CloudFront → Distributions**, search the domain and select the **old** distribution. Check its ID twice; never select `E1EXAMPLE123`.
2. Click **Disable** → **Disable**. Wait until it shows **Disabled** and **Last modified** shows a date (often 10 to 15 minutes).
3. Select it again → **Delete** → **Delete**.

Distributions whose origin is an S3 website bucket are live redirects. Keep them.

**Old certificates:** in **ACM (us-east-1)**, a certificate this site no longer uses shows **In use: No**. Select it → **Delete**.

**Server backups:** once the site has run cleanly for a few days, remove the backup files:

```bash
sudo rm /etc/apache2/sites-available/000-default.conf.bak
sudo rm /var/www/html/wp-config.php.bak
```

**Keep:**

- Every `_…` certificate validation CNAME in Route 53, for automatic renewal.
- The `origin.yoursite.com` record.
- The `CloudFront-Forwarded-Proto` lines in Apache and `wp-config.php`. Removing them brings the redirect loop back.

## Troubleshooting

Each problem below came up during a real setup; the fix shown is the one that worked.

| Symptom | Cause | Fix |
| --- | --- | --- |
| Browser: **ERR\_TOO\_MANY\_REDIRECTS**; curl: `301` to the same URL, `Server: Apache`, `charset=iso-8859-1`, no `X-Redirect-By` | Apache's HTTP-to-HTTPS redirect also catches CloudFront's requests | Step 2a–2b: add `RewriteCond %{HTTP:CloudFront-Forwarded-Proto} !https`, reload Apache |
| Same loop, but curl shows `X-Redirect-By: WordPress` | WordPress thinks the request is HTTP | Step 2c: the `wp-config.php` snippet, placed above `require_once ABSPATH . 'wp-settings.php';` |
| Loop continues although the server tests fine directly | CloudFront is not sending `CloudFront-Forwarded-Proto` | Every behaviour's origin request policy must be **AllViewerAndCloudFrontHeaders-2022-06**, not AllViewer. Wait for **Deploying** to finish |
| `X-Forwarded-Proto` custom header set on the origin, but the server never receives it | CloudFront does not reliably forward this header | Use `CloudFront-Forwarded-Proto` instead (Step 2). The custom header can stay; it is harmless |
| Security screen says WAF is "included in your plan"; existing web ACL cannot be chosen | A flat-rate plan (for example Free) was picked on wizard screen 1 | Go back to screen 1 and choose **Pay-as-you-go** |
| Saving alternate domains: "certificate … doesn't cover the alternate domain name" | Selected certificate lacks a hostname, or an older certificate is selected | Select the certificate whose ID matches the one covering all names; request a new one if needed |
| New certificate missing from CloudFront's dropdown | Requested outside us-east-1, or not yet **Issued** | Request it in **N. Virginia**; wait for **Issued**, then reopen the Edit page |
| "One or more of the CNAMEs … already associated with a different resource" | Another distribution holds the hostname | Remove the name from the old distribution or disable it (Step 5a). Leave S3 redirect distributions alone |
| ACM row stuck on **Pending validation** | Validation record name wrong (part dropped or domain doubled) | Delete the record and use **Create records in Route 53**; confirm with `nslookup -type=CNAME` |
| Route 53: "Value is not a valid IPv4 address" | cloudfront.net name typed with Alias off | Turn **Alias** on, choose **Alias to CloudFront distribution** |
| `403` "The request could not be satisfied" | Hostname not in the distribution's alternate domain names | Add it (Step 5a); the certificate must cover it |
| wp-admin login fails or saving does nothing | Admin behaviour allows only GET, HEAD | Set all seven methods on `/wp-admin/*`, `/wp-login.php`, `/wp-admin/admin-ajax.php` |
| A form or plugin feature fails with `403` | It posts to a file under `/wp-content/` while that path allows GET, HEAD only | Set all seven methods on `/wp-content/*`; caching is unaffected |
| An image or file replaced under the same name still shows the old version | It is cached at CloudFront for up to 24 hours | **Invalidations → Create invalidation**, path such as `/wp-content/uploads/*` |
| A legitimate page or upload is blocked | A managed WAF sub-rule matches it | Set only that sub-rule to **Count** (see Monitoring), watch, then decide |
| CloudFront **504** or site down | Server stopped answering, often under attack or out of resources | Check the instance is running → Lightsail **Metrics** (CPU, burst capacity) → restart the instance |

To see which layer redirects, compare these two:

```
curl.exe -sI https://yoursite.com
curl.exe -sI http://origin.yoursite.com -H "Host: yoursite.com" -H "CloudFront-Forwarded-Proto: https"
```

If the second returns `200` but the first loops, the problem is in CloudFront's settings; if both loop, it is on the server. To watch what actually reaches the server, run `sudo tcpdump -A -s0 -i ens5 'tcp dst port 80' | grep -iE "host:|forwarded-proto:"`, load the site, and press **Ctrl+C** to stop.

## Monitoring and maintenance

The firewall shows web traffic: what reached each site and what it blocked. It does not monitor WordPress itself, so pair it with an uptime monitor and Lightsail's metrics.

**Traffic and blocks: distribution Security tab**

1. Open **CloudFront → Distributions → E1EXAMPLE123 → Security**.
2. Set **Date range** (for example last 24 hours) and **Granularity** (1 hour).
3. Read **Security trends**: all, allowed and blocked requests, the hourly chart, **Top attack types** and **Top countries**.

The figures cover the whole shared **Lightsail\_Instances** pack, so every site in it appears together. A normal day shows low overnight traffic, a business-hours peak and a few percent blocked. Investigate a sudden jump in blocked share or an overnight spike in allowed traffic.

The **Sampled bot requests** panel splits traffic into verified bots (search engines, monitors), unverified bots (scrapers, scanners) and non-bots. **Enable bot protection** there turns on AWS WAF Bot Control, a paid add-on with monthly and per-request charges; it is not part of this setup.

**Why a request was blocked: sampled requests**

Open **WAF & Shield → Protection packs (web ACLs)** (Region scope: CloudFront (Global) and Regional) → **Lightsail\_Instances → View dashboard, logs and sampled requests**. Sampled requests list recent requests with IP, country, URL, the matching rule and the action. Use it when someone reports being blocked. Full logs are in the **CloudWatch log group** linked from the pack.

**A rule blocks something legitimate: Count mode**

1. **Lightsail\_Instances → Manage rules →** the rule set.
2. Under **Rule overrides**, set only the problem sub-rule to **Count**, then **Save rule**. It stops blocking but still records matches.
3. Watch its matches, then set it back to **Block** or leave it in Count.

Leave the **Override rule group** checkbox unticked; it switches the whole rule set to Count. To always allow a trusted service (an uptime monitor or SEO tool), add an IP allow rule and move it to the top of the rule order.

**Site slow or down**

1. Check the Lightsail instance is running.
2. Open its **Metrics** tab: CPU utilization and burst capacity. Burst capacity at zero slows the server sharply.
3. Restart the instance; this clears a jammed web server in most cases. Attack traffic exhausting the server is the usual cause.

CloudFront adds only about 30 to 60 ms to each page. If pages are slow, measure the server directly:

```
curl.exe -s -o NUL -w "ttfb %{time_starttransfer}s total %{time_total}s\n" https://yoursite.com/
curl.exe -s -o NUL -w "ttfb %{time_starttransfer}s total %{time_total}s\n" http://origin.yoursite.com/ -H "Host: yoursite.com" -H "CloudFront-Forwarded-Proto: https"
```

Similar numbers mean the delay is WordPress building the page; a page caching plugin such as WP Super Cache or W3 Total Cache is the next step there.

**Routine**

| When | Check |
| --- | --- |
| Weekly | Security tab of each distribution for spikes |
| Someone reports being blocked | Sampled requests in the protection pack |
| Site slow or down | Lightsail Metrics, then restart |
| After replacing a file under the same name | Create an invalidation |
| Yearly | Certificate status in ACM shows renewed (automatic while validation records exist) |

## Quick reference

Every value a finished site should have, for checking an existing setup at a glance.

**Route 53 records**

| Record | Type | Alias | Points to |
| --- | --- | --- | --- |
| `yoursite.com` | A | Yes | `d111111abcdef8.cloudfront.net` |
| `www.yoursite.com` | CNAME (or A alias) | No (Yes if A) | `yoursite.com` (or the distribution) |
| `origin.yoursite.com` | A | No | `203.0.113.10`, TTL 300 |
| `_….yoursite.com`, `_….www.yoursite.com` | CNAME | No | `_….acm-validations.aws.` (keep) |

**Certificate:** ACM, **us-east-1**, public, DNS validation, RSA 2048, export disabled, every hostname listed.

**Distribution: general**

| Setting | Value |
| --- | --- |
| Billing | Pay-as-you-go |
| Alternate domain names | Every hostname on the certificate |
| Custom SSL certificate | The us-east-1 certificate |
| Security policy | TLSv1.2\_2021 |
| Legacy clients support | Off |
| HTTP versions | HTTP/2 on |
| Web ACL | Lightsail\_Instances |

**Distribution: origin**

| Setting | Value |
| --- | --- |
| Origin domain | `origin.yoursite.com` (custom origin) |
| Protocol / port | HTTP only / 80 |
| Origin Shield | Off |
| Timeouts | 3 attempts, 10 s connect, 30 s response, 5 s keep-alive |
| IP address type | IPv4-only |

**Distribution: behaviours** (all: Redirect HTTP to HTTPS, compress Yes, origin request policy **AllViewerAndCloudFrontHeaders-2022-06**)

| Path pattern | Methods | Cache policy |
| --- | --- | --- |
| `/wp-admin/*` | All 7 | CachingDisabled |
| `/wp-login.php` | All 7 | CachingDisabled |
| `/wp-admin/admin-ajax.php` | All 7 | CachingDisabled |
| `/wp-content/*` | GET, HEAD | WP-Cache-With-Query |
| `/wp-includes/*` | GET, HEAD | WP-Cache-With-Query |
| `Default (*)` | All 7 | CachingDisabled |

**Cache policy WP-Cache-With-Query:** TTL 1 / 86400 / 31536000 s; headers none; cookies none; query strings all; Gzip and Brotli on.

**Server**

| File | Required content |
| --- | --- |
| Port 80 Apache site (for example `000-default.conf`) | `RewriteCond %{HTTP:CloudFront-Forwarded-Proto} !https` before the HTTPS redirect rule |
| `wp-config.php` | The `CloudFront-Forwarded-Proto` snippet above `require_once ABSPATH . 'wp-settings.php';` |
| WordPress Address / Site Address | `https://yoursite.com` (unchanged) |
