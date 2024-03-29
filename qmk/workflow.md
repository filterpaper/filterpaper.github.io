# QMK with GitHub Workflow

Setting up a local [QMK build environment](https://docs.qmk.fm/#/newbs_getting_started) can be onerous and requires substantial amount of disk space. This is a setup guide for a workflow build environment that runs entirely on GitHub. It uses a standalone [userspace](userspace.md) repository that will compile QMK firmware in a container. An account on [GitHub](https://github.com/) is the only prerequisite.


# Create a Keymap JSON

* Start by visiting the [QMK Configurator](https://config.qmk.fm/#/) site.
* Select your keyboard from the drop-down list (and choose a layout if required).
* Use your GitHub username for the `Keymap Name` field, e.g.:

![workflow1](workflow1.png)

* Customise the key map according to your preference.
* Select download next to `KEYMAP.JSON` to save the json file locally.
* Rename the file to your keyboard name, e.g. `cradio.json`, and note its location.


# Create a Repository

* Login to your GitHub account.
* Select the `Repositories` tab, and click on `New` on the right.
* Use `qmk_keymap` (or anything unique) as the repository name.
* Leave the other settings as default and select `Create repository`:

![workflow2](workflow2.png)

## Upload the Keymap JSON

* In the `Quick setup` page that follows, select `uploading an existing file`.
* Locate the json file (e.g. `cradio.json`) from the previous step.
* Drag that file into the browser page to upload.
* Write a meaningful commit message and select `Commit changes`:

![workflow3](workflow3.png)


# Create the Workflow

Back in the `qmk_keymap` repository page, press the period (`.`) key. The [github.dev](https://docs.github.com/en/codespaces/the-githubdev-web-based-editor) web-based VSCode editor will be loaded. This interface is where you can edit and commit code directly to GitHub.

* In the left `Explorer` section, click on the `New Folder` icon to create a folder named `.github/workflows` (note the `.` prefix). Press enter to complete the action:

![workflow4](workflow4.png)

* Click on the new `.github/workflows` folder and select the `New File` icon. Create a new file named `build.yml`:

![workflow5](workflow5.png)

* With the `build.yml` file selected, paste the following workflow content into the editor window on the right:
{% raw %}
```yml
name: Build QMK firmware
on: [push, workflow_dispatch]

jobs:
  build:
    runs-on: ubuntu-latest
    container: qmkfm/base_container
    strategy:
      fail-fast: false
      matrix:
# List of keymap json files to build
        file:
        - cradio.json
# End of json file list

    steps:

    - name: Checkout QMK
      uses: actions/checkout@v2
      with:
        repository: qmk/qmk_firmware
# Uncomment the following for develop branch
#        ref: develop
        fetch-depth: 1
        persist-credentials: false
        submodules: recursive

    - name: Checkout userspace
      uses: actions/checkout@v2
      with:
        path: users/${{ github.actor }}
        fetch-depth: 1
        persist-credentials: false

    - name: Build firmware
      run: qmk compile "users/${{ github.actor }}/${{ matrix.file }}"

    - name: Archive firmware
      uses: actions/upload-artifact@v2
      with:
        name: ${{ matrix.file }}_${{ github.actor }}
        retention-days: 5
        path: |
          *.hex
          *.bin
          *.uf2
      continue-on-error: true
```
{% endraw %}
(Do note that the `build.yml` workflow file requires proper indentation on every line.)

## Customising the Workflow

* Matrix `file:` section is a list of keymap files to be built.
* Change `cradio.json` keymap to your json file name.
* Additional files (with `-` prefix) can be appended here for multiple keyboards.

## Committing the Workflow

* Select `Source Control` on the left column and enter a meaningful commit message.
* Click on the `Commit` checkmark above to commit the file directly to your repository:

![workflow6](workflow6.png)

Committing a change to the repository will automatically trigger the workflow to build every json file listed in the `file:` section.

# Review GitHub Actions

* Return to your [GitHub](https://github.com/) page and the `qmk_keymap` repository.
* Select the `Actions` tab to display the `Build QMK Firmware` workflow.
* Select that workflow to display its run from the last commit.
* Successfully compiled firmware will be under the `Artifacts` section:

![workflow7](workflow7.png)

Download and flash the firmware file into your keyboard using [QMK Toolbox](https://docs.qmk.fm/#/newbs_flashing?id=flashing-your-keyboard-with-qmk-toolbox). If there are build errors, review the job log for details.


# Customising QMK

You can proceed to customise the QMK firmware using the [Userspace guide](https://docs.qmk.fm/#/feature_userspace) with the [github.dev](https://docs.github.com/en/codespaces/the-githubdev-web-based-editor) editor:

* Create a `config.h` file for QMK variables and definitions.
* Create a `rules.mk` to enable and disable QMK features.
* Create a `source.c` file for your customised code.
  * Add `SRC += source.c` to `rules.mk` to build this source.
* Commits will automatically trigger firmware build actions.

Additional keymaps for other keyboards must be retained in json format and appended to the `file:` matrix list in `build.yml`.


# References

* [QMK Firmware Documentation](https://docs.qmk.fm/#/)
* [GitHub Actions guide](https://docs.github.com/en/actions/learn-github-actions)
