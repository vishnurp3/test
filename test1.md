Good question – you’re very close to being able to hit this from Postman and then from Spring Boot.

I’ll break it into two parts:

1. **Create a read-only token with minimal privileges**
2. **Use Postman to call GitLab and read that SQL file**

I’ll also map the URL from your screenshot into the GitLab API format.

---

## 1. Create a minimal-privilege access token in GitLab

> Note: if personal / project access tokens are disabled in your NatWest GitLab, you’ll need to ask a GitLab admin or tools team to enable them or create one for you.

### 1.1 Steps to create a Personal Access Token

In your NatWest GitLab UI:

1. Log in normally via SSO in the browser.
2. Click your **avatar** (top right) → **Edit profile** (or **Preferences**, depending on version).
3. In the left sidebar, go to **“Access Tokens” / “Personal access tokens”**. ([GitLab Docs][1])
4. Click **“Add new token”**.
5. Fill:

   * **Name**: something like `read-repo-for-sql`.
   * **Expiration date**: keep it short (e.g., 30 days or whatever your policy allows).
6. **Scopes (this is the important part):**

   * ✅ **Check only**: `read_repository`
   * ❌ **Do NOT check**: `write_repository`, `api`, or anything else that gives write access.
   * If the instance requires it, you *may* also need `read_api`, which is still read-only. Start with `read_repository` and add `read_api` only if you get permission errors. ([about.gitlab.com][2])
7. Click **Create personal access token**.
8. Copy the token **once** and store it somewhere safe (password manager). You won’t see it again.

If PATs are disabled for enterprise users (quite common in banks), you may need a **Project Access Token** instead (scoped only to that project and still can be `read_repository` only). ([GitLab Docs][3])

Ask your admin something like:

> “For project `natwestgroup/DigitalX/Technology/RiskFinanceLegalAndAuditSolutions/RiskModels/basel`, I need a read-only access token (scope `read_repository`, maybe `read_api`) so a service can download a SQL file. No write permissions.”

---

## 2. Map your existing browser URL to the GitLab API

From your screenshot, the browser URL is basically:

```text
https://natwest.gitlab-dedicated.com/natwestgroup/DigitalX/Technology/RiskFinanceLegalAndAuditSolutions/RiskModels/basel/-/raw/feature/demo_cdf_model_migration/db/Code/CDF/Data/CDF_MODEL_VERSION.sql?ref_type=heads
```

Break that down:

* **Host**:
  `https://natwest.gitlab-dedicated.com`

* **Project path** (everything between host and `/-/`):
  `natwestgroup/DigitalX/Technology/RiskFinanceLegalAndAuditSolutions/RiskModels/basel`

* **Branch** (after `/raw/`):
  `feature/demo_cdf_model_migration`

* **File path in repo** (after the branch):
  `db/Code/CDF/Data/CDF_MODEL_VERSION.sql`

The official GitLab API for raw files is: ([GitLab Docs][4])

```text
GET /api/v4/projects/:id/repository/files/:file_path/raw?ref=:branch
```

So your API URL becomes:

```text
https://natwest.gitlab-dedicated.com/api/v4/projects/\
natwestgroup%2FDigitalX%2FTechnology%2FRiskFinanceLegalAndAuditSolutions%2FRiskModels%2Fbasel/\
repository/files/db%2FCode%2FCDF%2FData%2FCDF_MODEL_VERSION.sql/raw?ref=feature%2Fdemo_cdf_model_migration
```

Notes:

* `:id` can be the **URL-encoded project path**, so `/` becomes `%2F`.
* `file_path` is also URL-encoded (`/` → `%2F`).
* `ref` is the branch name URL-encoded (`feature/demo_cdf_model_migration` → `feature%2Fdemo_cdf_model_migration`).

---

## 3. Test from Postman

Once you have the token, testing in Postman is straightforward.

### 3.1 Quick sanity check – is the API reachable?

In Postman:

1. **Method**: `GET`
2. **URL**:
   `https://natwest.gitlab-dedicated.com/api/v4/version`
3. **Headers**:

   * `PRIVATE-TOKEN: <your-token>`

If everything is OK, you’ll get some JSON like:

```json
{
  "version": "XX.YY.Z",
  "revision": "..."
}
```

If this fails (401 / 403), either:

* Token scopes are wrong,
* Token is invalid or expired,
* Or your admins have disabled PATs / API access.

---

### 3.2 Get the SQL file content with Postman

Create a new request:

1. **Method**: `GET`

2. **URL** (all on one line):

   ```text
   https://natwest.gitlab-dedicated.com/api/v4/projects/natwestgroup%2FDigitalX%2FTechnology%2FRiskFinanceLegalAndAuditSolutions%2FRiskModels%2Fbasel/repository/files/db%2FCode%2FCDF%2FData%2FCDF_MODEL_VERSION.sql/raw?ref=feature%2Fdemo_cdf_model_migration
   ```

3. **Headers**:

   * Key: `PRIVATE-TOKEN`
     Value: `<your-token-here>`

4. Hit **Send**.

If your token and URL are correct, the **Body** tab should show the content of `CDF_MODEL_VERSION.sql` (the MERGE script you showed).

If you want to download it as a file, use Postman’s **“Save Response → Save to a file”** option.

> Tip: you can also use the browser “raw” URL directly in Postman:
>
> ```text
> https://natwest.gitlab-dedicated.com/natwestgroup/DigitalX/Technology/...
>   /basel/-/raw/feature/demo_cdf_model_migration/db/Code/CDF/Data/CDF_MODEL_VERSION.sql?ref_type=heads
> ```
>
> with the same `PRIVATE-TOKEN` header. But the **API URL** is more standard and documented.

---

## 4. Keeping it safe (no write / delete)

* Use **only `read_repository` (and optionally `read_api`)** on the token – this prevents commits, pushes, file creation, etc. ([about.gitlab.com][2])

* Store the token in:

  * Postman **environment variable**, not hard-coded in requests.
  * For Spring Boot, put it in an environment variable or secret manager, and read it via `application.yml` like:

    ```yaml
    gitlab:
      token: ${GITLAB_TOKEN}
    ```

* If you’re extra cautious, ask your admin for a **Project Access Token** scoped only to this one project, again with `read_repository` only. ([GitLab Docs][3])

---

If you want, next step I can:

* Turn this Postman call into the exact Spring Boot `WebClient` or `RestTemplate` code that does the same GET and then appends your `UNION ALL SELECT` rows.

[1]: https://docs.gitlab.com/user/profile/personal_access_tokens/?utm_source=chatgpt.com "Personal access tokens"
[2]: https://gitlab.com/gitlab-org/gitlab/-/blob/v17.6.2-ee/doc/user/profile/personal_access_tokens.md?utm_source=chatgpt.com "doc/user/profile/personal_access_tokens.md"
[3]: https://docs.gitlab.com/user/project/settings/project_access_tokens/?utm_source=chatgpt.com "Project access tokens"
[4]: https://docs.gitlab.com/api/repository_files/?utm_source=chatgpt.com "Repository files API | GitLab Docs"
