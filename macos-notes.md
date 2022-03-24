# MacOS on Apple Silicon notes
## Migrating to zsh
Most of these customisation came from the [Moving to zsh](https://scriptingosx.com/2019/06/moving-to-zsh/) series.

### .profile
To retain backward compatibility with `bash`, the following settings and aliases were retained in the `.profile`:
```sh
PS1='\u@\h:\w\$ '
PS1='\[\e[01;37m\]\u@\[\e[01;32m\]\h\[\e[01;37m\]:\w\\$ \[\e[0m\]'
PROMPT_COMMAND='echo -ne "\033]0;${PWD##*/}\007"'
LANGUAGE=en_SG.UTF-8
LC_ALL=en_SG.UTF-8
LANG=en_SG.UTF-8
IGNOREEOF=1

PATH=/opt/homebrew/bin:$PATH

alias ls='ls --color'
alias killds="find . -name .DS_Store -print -exec rm {} \;"
alias cleardisk='rm -rf .Trashes/ .fseventsd/ .Spotlight-V100; cd; diskutil eject disk1'
```

### .zshrc
Customization specific to `zsh` are placed in the .zshrc file. The `.profile` content is included with `source`:
```sh
source ~/.profile
```


## QMK scripts
[QMK Toolbox](https://github.com/qmk/qmk_toolbox) do not have full Apple Silicon support yet. Internal flashing function relies on command line tools from brew and they using x86 binaries. To avoid using Rosetta 2, install `dfu-programmer` from [Homebrew](https://brew.sh/) and use the following alias function that is compatible with both `zsh` and `bash`:
```sh
dfu-flash() {
  if [[ ! -f $1 || -z $1 ]]
  then
    echo "Usage: dfu-flash <firmware.hex> [left|right]"
    return 1
  fi
  until [[ $(ioreg -p IOUSB | grep ATm32U4DFU) == *"DFU"* ]]
  do
    echo "Waiting for ATm32U4DFU bootloader..."; sleep 3
  done
  dfu-programmer atmega32u4 erase --force
  if [[ $2 == "left" ]]
  then
    echo -e "\nFlashing left EEPROM" && echo -e ':0F000000000000000000000000000000000001F0\n:00000001FF' | \
    dfu-programmer atmega32u4 flash --force --suppress-validation --eeprom STDIN
  elif [[ $2 == "right" ]]
  then
    echo -e "\nFlashing right EEPROM" && echo -e ':0F000000000000000000000000000000000000F1\n:00000001FF' | \
    dfu-programmer atmega32u4 flash --force --suppress-validation --eeprom STDIN
  fi
  echo -e "\nFlashing $1" && dfu-programmer atmega32u4 flash --force $1
  dfu-programmer atmega32u4 reset
}
```
