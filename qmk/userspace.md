One of the least obvious way of building a QMK firmware is using the json that is directly exported from the configurator.
I favor the use of this method because:

* It eliminates additional step of converting the output with json2c if configurator is used as a convenient key map editor. 
* It is less tedious and error prone than maintaining key layout purely using a text editor.
* Easier to maintain files inside a single user space than to place them deep inside keyboard folders, and extend support to more than one keyboard.

# First time setup
Decide on the user space name. If you intend to contribute or push changes to qmk, user same name should be your github id. I'll use "newbie" as an example 

Create directory inside user newbie. 

Customise layout using web configuration for your keyboard , ensure that the key map name is the same as the user space. 

Download the json and it will be called newbie.json. Move that into user space 

qmk compile json file with absolute path. 

Firmware will be built with default settings. 

Start customising, with config and rules

Add some code file. 

Compile again and it will pick up those changes.

Practices with rules define and config define 

Add a new keyboard, more defines, distinct json 
