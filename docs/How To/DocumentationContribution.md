# Documentation Contribution

## Update the Site or Add Documentation

The documentation is hosted in GitHub at [https://github.com/redhat-openinfra-lab/openinfra-docs](https://github.com/redhat-openinfra-lab/openinfra-docs).

As the site requires **pip** to install **mkdocs** and **mkdocs-material**, we recommend creating a Virtual Environment to work on the page documentation.  This page will explain how to do this.

### Configure virtual environment

1. Install **python3** and **pip**

2. Create the virtual environment
```
export VENV=${HOME}/virtual_environments/mkdocs
python3 -m venv ${VENV}
```

3. Activate the Virtual Environment (You will want to do this step everytime you want to work on the doumentation)

4. Use **pip** to install **mkdocs** and **mkdocs-material**
```
python3 -m pip install --upgrade mkdocs mkdocs-material
```

### Clone Github Repository

Clone the [site](https://github.com/redhat-openinfra-lab/openinfra-docs/) to your local repository
```
git clone git@github.com:redhat-openinfra-lab/openinfra-docs.git
```

### Working locally

Once you have a copy of the repository, **mkdocs** can serve the documentation so you can view your changes real-time.  Remember, you will need to activate the virtual environment using Step 3 above before trying to start the **mkdocs** server. If you are working on the documentation on your desktop, you can simply start the server using:
```
cd openinfra-docs
mkdocs serve
```

This will start a server listening on port 8000 on your loopback device.  If you are using something other than your desktop, such as a Virtual Machine, you will need to open a port on your firewall and utilize the `-a` switch when starting the **mkdocs serve** command to specify an IP:PORT to start on.  Example:
```
cd openinfra-docs
mkdocs serve -a 172.20.135.10:8000
```

The site is configured to automatically publish changes to the gh-pages sites when a `push` to the master branch is executed.  Check the status of the pages build and deployment under the *Actions* menu.