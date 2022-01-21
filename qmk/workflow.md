# Building QMK with GitHub Actions
A workflow can be setup for GitHub Actions to automatically build QMK firmware. To use this feature, a GitHub account must be setup, [using a personal fork of QMK](https://docs.qmk.fm/#/getting_started_github).

# Setting up the workflow
The workflow should be setup in a personal or development branch, not the `master`. It should be placed in the same user keymap development branch. See [Git best practices for working with QMK](https://docs.qmk.fm/#/newbs_git_best_practices) for details.

Create a `build.yml` in the folder location `~/qmk_firmware/.github/workflows`, with the following content:

```yml

name: Build firmware for keyboards
on: [push, workflow_dispatch]

jobs:
  build:
    runs-on: ubuntu-latest
    container: qmkfm/qmk_cli
    strategy:
      fail-fast: false
      matrix:
        keyboard:
          - clueboard/66/rev4
        user:
          - ${{ github.actor }}

    steps:

    - name: Checkout QMK
      uses: actions/checkout@v2
      with:
        fetch-depth: 1
        persist-credentials: false
        submodules: true

    - name: Build firmware
      run: make ${{ matrix.keyboard }}:${{ matrix.user }}

    - name: Archive firmware
      uses: actions/upload-artifact@v2
      with:
        name: "${{ matrix.keyboard }}_${{ matrix.user }}_firmware"
        retention-days: 2
        path: |
          *.hex
          *.bin
      continue-on-error: true
```

## Customising the workflow

Workflow matrix in the example above is using the keyboard `clueboard/66/rev4`. Change this line to the keyboard name that you use. If you have more than one keyboard, they can be listed one after another with a `-` prefix, e.g.:

```yml
matrix:
  keyboard:
    - crkbd/rev1
    - ferris/sweep
    - planck/rev6
```

The example also uses your GitHub user name as the default for the keymap name with the parameter `${{ github.actor }}`. You can change that and add more user keymaps to build, e.g.:

```yml
matrix:
  keyboard:
    - crkbd/rev1
    - ferris/sweep
    - planck/rev6
  user:
    - my_keymap_name
    - default
```

# Submitting the workflow

Ensure that a user keymap has been created for every keyboard listed in the workflow. Commit them, along with the workflow file `~/qmk_firmware/.github/workflows/build.yml` in your branch and `git push` the commit to GitHub.

## Downloading the firmware files

Load the GitHub page of your QMK fork (e.g. https://github.com/git_username/qmk_firmware) and visit the "Actions" tab. On the left "Workflows" sidebar, you will find one labelled `Build firmware for keyboards` that was initiated by the `build.yml` file.
1. Select `Build firmware for keyboards` to display its run on the right table.
2. Select the latest workflow in the table to display job status.
3. Successfully compiled firmware can be downloaded from the "Artifacts" section.
