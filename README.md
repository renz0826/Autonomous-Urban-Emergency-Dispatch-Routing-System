# Setup environment (collaborators)
IMPORTANT: It is highly recommended to create a python venv before installing dependencies as to keep your global python environment clean.

To create and activate venv:
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

After cloning, run the commands:
```bash
pip install -r requirements.txt # make sure your venv is enabled if opting to work in venv
nbstripout --install && git config filter.nbstripout.extrakeys "metadata.kernelspec metadata.language_info"
```

The last two commands make sure jupyter notebook metadata changes are not included in commits and cause issues.