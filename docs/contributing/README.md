# Contributing

How these runbooks are written and kept up to date.

## Adding or editing a runbook

1. Copy the [runbook template](runbook-template.md) into the right phase folder under `docs/`, for example `docs/setup/always-on-ag.md`.
2. Write the steps. Keep one action per step and give every step an **Expected result**.
3. Add a row for it to the table in that phase's `README.md` and to the table in the main `README.md`.
4. Commit to `main`.

You can do all of this in the browser: open a file on GitHub and click the pencil icon to edit it, or use **Add file** > **Create new file** inside a phase folder.

## Adding screenshots

1. Save the screenshot as a PNG in `docs/images/<runbook-name>/`, named after the step, for example `docs/images/linked-server/step-04.png`. On GitHub you can do this with **Add file** > **Upload files** inside that folder.
2. Each runbook already has a commented-out image line under every step. Remove the `<!--` and `-->` around it so it reads:

    ```markdown
    ![Step 4: Create the linked server in SSMS](../images/linked-server/step-04.png)
    ```

3. Crop to the relevant window, and blur or remove real server names, IP addresses, logins and anything from your employer before uploading. This repo is public.

## Status labels

| Label | Meaning |
|---|---|
| Draft | Written, not yet run end to end |
| Reviewed | Read and checked by a second DBA |
| Tested in lab | Every step run on a lab build, with screenshots |

## Sources

Every runbook ends with a **Sources** section. Link to the original Microsoft Learn,
MSSQLTips or SQLShack article and write the steps in your own words. Do not paste
articles in.
