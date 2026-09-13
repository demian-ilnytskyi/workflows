# Firebase Functions deploy — setup

Used by [firebase_deploy_functions.yml](firebase_deploy_functions.yml). Authenticates
to Firebase/GCP via a service-account (or `authorized_user` ADC) key
(`GOOGLE_APPLICATION_CREDENTIALS`), not the deprecated `firebase login:ci` token.

## 1. Create the GCP service account

In the target Firebase/GCP project (Google Cloud Console → IAM & Admin →
Service Accounts):

1. Create a service account, e.g. `firebase-cd@<project-id>.iam.gserviceaccount.com`.
2. Grant it these roles:
   - `Firebase Admin` (`roles/firebase.admin`)
   - `Cloud Functions Admin` (`roles/cloudfunctions.admin`)
   - `Service Account User` (`roles/iam.serviceAccountUser`)
   - `Cloud Build Editor` (`roles/cloudbuild.builds.editor`)
   - `Artifact Registry Administrator` (`roles/artifactregistry.admin`)
   - `Secret Manager Admin` (`roles/secretmanager.admin`) — only needed if you
     pass `SECRETS_MAP` to `firebase_deploy_functions.yml`
   - `Project IAM Admin` (`roles/resourcemanager.projectIamAdmin`) — required
     the first time functions deploy grants IAM roles to GCP service agents;
     without it deploy fails with "We failed to modify the IAM policy for the
     project."
   - `Service Usage Admin` (`roles/serviceusage.serviceUsageAdmin`) — lets the
     account enable new APIs itself (Cloud Functions, Cloud Build, Eventarc,
     Pub/Sub, Storage, etc. get enabled on-demand by `firebase deploy`).
     Without it, deploy fails partway through with "Permissions denied
     enabling <api>.googleapis.com."
   - `Compute Viewer` (`roles/compute.viewer`) — `firebase deploy` looks up
     the project's default compute service account via
     `compute.projects.get` to grant it Eventarc/Run IAM bindings on a
     first-ever functions deploy. Without this role (even with the Compute
     Engine API enabled) that lookup 403s with "Required
     'compute.projects.get' permission", which gets misreported as the
     unrelated "We failed to modify the IAM policy for the project" error —
     see the Compute Engine API note below for how to tell them apart.
3. Keys tab → Add Key → JSON. Downloads a `.json` key file.
4. **Billing must already be linked** to the project (Cloud Billing →
   linked account). `roles/serviceusage.serviceUsageAdmin` cannot enable
   `cloudbilling.googleapis.com` itself if billing was never linked — do
   that once as project Owner via Cloud Console before the first deploy.
5. **Enable the Compute Engine API** (`compute.googleapis.com`) once, even
   though nothing here runs a VM: `firebase deploy` looks up the project's
   default compute service account (`<project-number>-compute@developer.gserviceaccount.com`)
   via the Compute Engine API (`compute.projects.get`) to grant it the
   Eventarc/Run IAM bindings a first-ever functions deploy needs. This needs
   **both** the API enabled and the `Compute Viewer` role from step 2 above —
   `firebase deploy --debug` is the only way to see which one is missing,
   since both failure modes get misreported as the unrelated "We failed to
   modify the IAM policy for the project" error otherwise:
   - API disabled → `Compute Engine API has not been used in project ... or
     it is disabled`
   - Role missing → `Required 'compute.projects.get' permission for
     'projects/<project-number>'`

   Enable the API once:
   ```
   gcloud services enable compute.googleapis.com --project <project-id>
   ```
   or via Console: `https://console.developers.google.com/apis/api/compute.googleapis.com/overview?project=<project-id>`.
   Wait a minute or two for it to propagate before retrying.
6. **First-ever functions deploy on a project** hits several one-time
   issues, because the default compute service account
   (`<project-number>-compute@developer.gserviceaccount.com`, which is what
   Cloud Build actually runs as) may come with none of its usual roles on a
   freshly created project. Each missing role only surfaces after the
   previous one is fixed, so grant all of these up front:
   ```
   SA="<project-number>-compute@developer.gserviceaccount.com"
   for R in roles/logging.logWriter roles/artifactregistry.writer \
            roles/storage.objectViewer; do
     gcloud projects add-iam-policy-binding <project-id> \
       --member="serviceAccount:$SA" --role="$R"
   done
   ```
   The individual symptoms, in the order they appear:
   - Gen1 functions fail with "Access to bucket
     gcf-sources-<project-number>-<region> denied. You must grant Storage
     Object Viewer permission to
     <project-number>-compute@developer.gserviceaccount.com." Fix once,
     doesn't recur:
     ```
     gcloud projects add-iam-policy-binding <project-id> \
       --member="serviceAccount:<project-number>-compute@developer.gserviceaccount.com" \
       --role="roles/storage.objectViewer"
     ```
   - Every function fails with a generic "Build failed with status: FAILURE
     and message: An unexpected error occurred" / "Build error details not
     available", and **no build log exists anywhere** — nothing in Cloud
     Logging (`logName:"cloudbuild"` returns 0 entries), no logs bucket, and
     `gcloud builds describe` shows a null `logsBucket`. Cloud Functions
     builds run with `logging: CLOUD_LOGGING_ONLY`, so a build service
     account that can't write logs dies at the first real buildpack step
     with no output. The missing-logs symptom IS the diagnosis. Fix once:
     ```
     gcloud projects add-iam-policy-binding <project-id> \
       --member="serviceAccount:<project-number>-compute@developer.gserviceaccount.com" \
       --role="roles/logging.logWriter"
     ```
     Note this is deterministic — it never resolves by retrying, unlike the
     Eventarc case below.
   - Once logs are readable, the next failure shows up in them as
     `DENIED: Permission 'artifactregistry.repositories.downloadArtifacts'
     denied on .../repositories/gcf-artifacts` at build step 2, failing with
     `ERROR: failed to create image cache`. Needs
     `roles/artifactregistry.writer` (write, not just read — the build both
     reads and populates the layer cache).
   - **Any failed deploy leaves orphaned function records behind**, and the
     NEXT deploy then fails early with
     `Error: [<fn>(<region>)] Changing from an HTTPS function to a background
     triggered function is not allowed. Please delete your function and
     create a new one instead.` — masking whatever the real error was. The
     orphan is a husk: `gcloud functions describe <fn>` shows
     `state: FAILED`, no `eventTrigger` field, and `CloudRunServiceNotFound`
     / `EventarcTriggerNotFound` state messages. With no trigger recorded,
     the CLI reads it as HTTPS and refuses to convert it. Delete the husks
     before each retry, or you debug the wrong error:
     ```
     gcloud functions delete <fn> --region=<region> --quiet
     ```
     Tell-tale that this is what happened: the pipeline fails with NO new
     Cloud Build runs (`gcloud builds list` shows nothing newer than the
     previous attempt), because it never got past CLI validation.
   - Gen2 functions fail with an Eventarc "Permission denied while using the
     Eventarc Service Agent" error, alongside Firebase's own message:
     "Since this is your first time using 2nd gen functions, we need a
     little bit longer to finish setting everything up. Retry the
     deployment in a few minutes." No fix needed — just re-run the pipeline
     a few minutes later once the service-agent IAM grants have propagated.

## 2. Store the key in GitHub

1. Repo (or org) → Settings → Secrets and variables → Actions.
2. Add a repository secret named `GOOGLE_APPLICATION_CREDENTIALS_JSON`.
3. Paste the **entire contents** of the downloaded JSON key file as the
   value (single line is fine, JSON stays valid).
4. Add a repository **variable** named `FIREBASE_PROJECT_ID` set to the
   Firebase project ID — optional if `.firebaserc` already has the right
   `default` project, since the deploy action falls back to that.
5. If you need `firebase_deploy_functions.yml`'s `SECRETS_MAP` (to set
   `defineSecret(...)`-backed Functions secrets from repo secrets before
   deploy), add each named secret it references (e.g. `SUPABASE_URL`,
   `SUPABASE_SERVICE_KEY`) as its own repository secret, and pass
   `secrets: inherit` on the caller job — see
   `demian-ilnytskyi/inflalite`'s `.github/workflows/deploy_firebase.yaml`
   for a working example of the `ENV_VAR: SECRET_NAME` map format.

## 3. Rotate a leaked key

If a key value is ever pasted somewhere insecure (chat, ticket, log): go to
the service account's Keys tab in GCP Console, delete the exposed key, add a
new one, and update the GitHub secret with the new JSON.

## Alternative: org policy blocks key creation

If the target project's GCP org enforces
`iam.disableServiceAccountKeyCreation` ("Secure by Default"), step 3 above
(Keys → Add Key) is unavailable — Owner doesn't override it, and it requires
`roles/orgpolicy.policyAdmin` at the **organization** level to lift. Rather
than getting that exception, authenticate as a real Google user instead of a
service-account key:

1. On a machine with a browser, run `gcloud auth application-default login`
   as an account that has (or is granted) the required roles from step 2
   above on the target project.
2. `cat ~/.config/gcloud/application_default_credentials.json` — paste its
   **entire contents** as `GOOGLE_APPLICATION_CREDENTIALS_JSON` (same
   secret, same steps as "2. Store the key in GitHub" above). This JSON has
   `"type": "authorized_user"` instead of `"service_account"` — no
   service-account key is created or stored, so the policy never applies.
3. Everything downstream is unchanged: `write-google-credentials` detects
   the `authorized_user` type automatically and additionally exports
   `GOOGLE_CLOUD_QUOTA_PROJECT` (set to `FIREBASE_PROJECT_ID`), since this
   credential type has no project of its own to bill API usage against.

This credential expires/rotates with the underlying Google account (password
change, org offboarding, `gcloud auth revoke`) — treat it as at least as
sensitive as a service-account key, and prefer a machine/CI-dedicated Google
account over a personal one if you go this route.

## How it's used at runtime

[`write-google-credentials`](../actions/write-google-credentials/action.yml)
writes `GOOGLE_APPLICATION_CREDENTIALS_JSON` to a temp file on the runner and
exports `GOOGLE_APPLICATION_CREDENTIALS` (via `GITHUB_ENV`) pointing at that
path. The `firebase` CLI picks up Application Default Credentials from that
env var automatically — no `--token` flag anywhere, for either credential
type above.
