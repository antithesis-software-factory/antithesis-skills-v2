# Customer setup

* Import the repository under test as a Replit App (Import in the sidebar). Run triage inside this App, not in a Replit chat. Triage needs the code under test and the Antithesis secrets, and only an App has them.
* Add these secrets to the App (the App's Secrets pane):
  * ANTITHESIS_API_KEY
  * ANTITHESIS_TENANT

  To share them across Apps, add them once as Account secrets (your account settings → Account secrets → New secret), then link them into each App.
* Add the skills to your Replit Workspace: Settings → Customization → Skills → Add.
  * If Add offers "Add from GitHub", use this URL and add the `antithesis` skill:

     ```
     https://github.com/antithesis-software-factory/antithesis-skills-v2/tree/master/plugins/antithesis/skills
     ```

  * Otherwise, choose "Upload a skill" and upload the skill as a zip, with `SKILL.md` at the top level of the zip:

     ```bash
     git clone https://github.com/antithesis-software-factory/antithesis-skills-v2
     cd antithesis-skills-v2/plugins/antithesis/skills
     (cd antithesis && zip -r ../antithesis.zip .)
     ```

  Re-import or re-upload to pick up a new release. To use the skills in one App only, copy `antithesis` into the App's `.agents/skills/` directory instead.

# Usage

In the App, ask Agent to triage a run, or invoke the skill directly:

```
/antithesis triage <run id>
```

There is no automated triage yet. Replit has no way to start an Agent session from a webhook, so triage runs only when someone asks for it.
