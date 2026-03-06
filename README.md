Directions:

Save it as desktop_doctor.py.

In VS Code, open that file.

Press the Play button. It will run scan automatically.

For actual cleanup, run in the terminal:

python3 desktop_doctor.py clean --yes

For a JSON report:

python3 desktop_doctor.py report

What the Play button now does:

no arguments needed

defaults to safe scan mode

shows you what is taking space without deleting anything

What clean mode removes:

Trash

thumbnail cache

pip cache

/tmp items owned by your user

apt cache

old journal logs

What it does not touch:

documents

pictures

code repos

project folders

installed packages
