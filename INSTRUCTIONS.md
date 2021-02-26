# How this all works?

The main script [landscapemembers.py](landscapemembers.py) is designed to pull a list of all current members of the given project from LF SFDC, using that as the primary source of data for the landscape. Data pulled includes...

- Member name
- Website
- Crunchbase URL
- Twitter handle
- Logo URL

The pull will look for active members based on the Purchase History, and respect the 'Don't show logo on website' item if selected.

With the data for things like logos and crunchbase entries, there is often more accurate data in other [landscapes](https://landscapes.dev). The script will attempt to look for this, and overlay data from other landscapes if the data in SFDC is empty. This also pulls in useful data such as `stock_ticker`, which often has to be set to `null` as the Crunchbase data is inaccurate.

# How to use

You can use this script in a few different ways, but generally one of the below works best

## Recommended - Automatic build with GitHub Actions

Easiest option to do a build is leveraging GitHub Actions, which requires nothing for you to setup on your local machine. You can have it run on demand, or 
set the `cron` option in the `update_members.yml` action to have it run on a schedule ( this is how it's done for the LF Landscape, which runs nightly at 9:00pm-ish EST ).

If it's all setup, goto the landscape repo under [Actions](actions). You should see a job called 'Update members' on the right side under 'Workflows'. Click that, then on the next screen click 'Run workflow'

There is some prerequiste setup in GitHub.

1) Add [secrets](https://docs.github.com/en/actions/reference/encrypted-secrets) for `SF_USERNAME`, `SF_PASSWORD`, `SF_TOKEN`, and `PAT`. The first three are part of the [configuration options for the tool](https://github.com/jmertic/landscape-tools/blob/master/README.md#environment-variables), and `PAT` is a [GitHub Personal Authorization Token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) set for the `repo` scope.
2) [Add two new labels](https://docs.github.com/en/github/managing-your-work-on-github/managing-labels#creating-a-label) - `automerge` and `automated-build`. These are for this workflow to all work and shouldn't be used for anything else.

Actions are stored in the `.github/workflows` directory. There are three to create if you want to do the entire workflow, including having it autocommit if the build all works successfully.

First one is `update_members.yml`

```yaml
name: Update members

on:
  workflow_dispatch:
  # you can add a cron option here as well if you want it fully hands off

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout omp-landscape
        uses: actions/checkout@v2
        with:
          path: master
      - name: Checkout landscape-tools
        uses: actions/checkout@v2
        with:
          repository: jmertic/landscape-tools
          path: landscape-tools
      - name: Set up Python 3.x
        uses: actions/setup-python@v2
        with:
          python-version: '3.x'
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r landscape-tools/requirements.txt
      - name: Run build
        working-directory: ./main
        env:
          SF_USERNAME: ${{ secrets.SF_USERNAME }}
          SF_PASSWORD: ${{ secrets.SF_PASSWORD }}
          SF_TOKEN: ${{ secrets.SF_TOKEN }}
        run: |
          ../landscape-tools/landscapemembers.py
      - name: Save missing.csv file
        uses: actions/upload-artifact@v2
        with:
          name: missing-members 
          path: ./main/missing.csv
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v3
        with:
          token: ${{ secrets.PAT }}
          branch-suffix: timestamp
          path: ./main
          title: Update members
          labels: automated-build
          commit-message: Update members
```

Next is `marksuccessfulbuild.yml`, which looks for when the Netlify preview build is done and labels the pull request to be merged.

```yaml
name: "Set Issue Label on successful build"
on:
  workflow_dispatch:
  issue_comment:
    types: [created]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: Amwam/issue-comment-action@v1.3.1
        if: ${{ github.event.issue.pull_request && contains(github.event.issue.labels.*.name, 'automated-build') }}
        with:
          keywords: '["Deploy preview for *omp-landscape* ready"]'
          labels: '["automerge"]'
          github-token: "${{ secrets.PAT }}"
```

Finally, `automerge.yaml` will merge the pull request moments after the issue us labeled `automerge`

```yaml
name: Autocommit pull requests

on:
  workflow_dispatch:
  pull_request:
    types: [labeled]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Automerge Pull Request if possible
        uses: "pascalgn/automerge-action@v0.13.0"
        env:
          GITHUB_TOKEN: "${{ secrets.PAT }}"
          MERGE_LABELS: "automerge"
          MERGE_RETRY_SLEEP: 300000
          MERGE_METHOD: "squash"
```



## Optional - Running a full build locally

To run things locally, you'll want to have both this repo and the given landscape repo checked out. The following steps should do that...

```zsh
git clone https://github.com/jmertic/landscape-tools.git
git clone [landscape repo url] landscape # if you have multiple landscapes pick a new name here ;-)
git clone https://github.com/cncf/landscapeapp.git
cd landscapeapp
npm install -g yarn@latest
yarn
cd ../landscape-tools
chmod +x *.py
pip install -r requirements.txt
./downloadcrunchbasedata.sh
```

You'll want to add the [API keys](https://github.com/cncf/landscapeapp#api-keys) and [.bash_profile or .zshrc](https://github.com/cncf/landscapeapp#installing-locally) bits here as well.

You will also want to create the `config.yaml` file in the `landscape` directory with the [proper config settings for your landscape](https://github.com/jmertic/landscape-tools#configuration)

Now to run a full build, do the following...

```zsh
cd landscape # or whatever the directory name you chose above for the landscape clone itself
../landscape-tools/landscapemembers.py
yf
yo
```

`../landscape-tools/landscapemembers.py` will take about a minute, and will update the `landscape.yml` file in your local clone, as well as adding the logos under `hosted_logos/`.

`yf` likely will take a few minutes on the first run, but should go faster after that. This will update `processed_landscape.yml`

`yo` will run and you can preview the landscape. Once you are happy with thing, commit the changes and push to Github.

```zsh
git add .
git commit -m 'Updated member logos' # or whatever you want to set here
git push
```

# Troubleshooting errors

There are a few common issues that pop up that require intervention.

- Bad logo, either the 'SVG embeds a PNG' error or 'Convert text to glyph'. Tips on how to resolve these errors and get a proper logo are at https://github.com/cncf/landscapeapp#images, and then refer to the internal instructions for updating logos.
- Bad crunchbase entry, for this just add the right one in SF for the member under 'Crunchbase URL'