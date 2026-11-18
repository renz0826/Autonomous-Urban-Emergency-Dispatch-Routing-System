# Setup environment (collaborators)
IMPORTANT: It is highly recommended to create a python venv before installing dependencies as to keep your global python environment clean.

#### Create and activate venv
1. Windows:
```bash
python -m venv .venv 
.venv\Scripts\Activate
```
2. macOS / Linux
```bash
python3 -m venv .venv
source .venv/bin/activate
```

#### Setup nbstripout to clean `.ipynb` before commits
After cloning, run the commands:
```bash
pip install -r requirements.txt # make sure your venv is enabled if opting to work in venv
nbstripout --install && git config filter.nbstripout.extrakeys "metadata.kernelspec metadata.language_info"
```

The last two commands make sure jupyter notebook metadata changes are not included in commits and cause issues.

#### Verify that nbstripout is installed and configured
```bash
nbstripout --version
nbstripout --status
```
Expected output:
```bash
0.9.1
nbstripout is installed in repository 'path/to/repository'

Filter:
  clean = "path/to/python.exe" -m nbstripout
  smudge = cat
  diff= "path/to/python.exe" -m nbstripout -t
  extrakeys= metadata.kernelspec metadata.language_info

Attributes:
  *.ipynb: filter: nbstripout

Diff Attributes:
  *.ipynb: diff: ipynb
```