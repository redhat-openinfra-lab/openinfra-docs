# Welcome to the Red Hat NA-SSA Lab's Documentation Site.

This site is created using MKDocs.  For full documentation visit [mkdocs.org](https://www.mkdocs.org).

## Pre-Requisites

Local client must have python and git.  After installing python, use pip to install the mkdocs and mkdocs-material packages.  It is not required, but using VS Code makes updating the documents easy and integrates with GitHub nicely.


## Update the Site or Add Documentation

Clone the site to your local repository.  Once you have a copy of the repository, update the documentation locally.  Preview your changes using *mkdocs serve*.  This will initialize a local webserver as 127.0.0.1:8000.


## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        ...       # Other markdown pages.
        /images   # images that are used in docs pages


## Helpful commands

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

