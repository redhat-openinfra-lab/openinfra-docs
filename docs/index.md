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


## Lab VPN Access

* Connect to the *Red Hat* VPN
* Access the <a href="https://10.9.57.124/" target="_blank">Fortigate Administration Page</a>.
* You will need to login as the `user_mgr` account. These credentials are in the <a href="https://vault.bitwarden.com/#/login" target="_blank">BitWarden Vault</a> under *Fortigate User Management FW*.
* After logging in, navigate to the *User & Device -> User Definition* on the left:

    ![VPN-User](images/vpn-user.png)

* Click *Create New* 
> NOTE: If you are changing your password, click *Edit User* and change it there.
    - User Type: Local User
    - Login Credentials: Input user name and password, use your Kerberos name for consistency.
    - Contact Information: Input your Red Hat e-mail address
    - Extra Information: Select the following

    ![VPN-Wizard](images/vpn-wizard.png)

  > NOTE> : If you accidentally create a user and you want to delete it, first edit the user, remove the group membership from it, disable group membership and then you will be able to delete it.

Once you have the new credentials configured, log out in the top right corner.  Test your new credentials by connecting to the <a href="https://209.132.179.151:20443" target="_blank"> Fortinet VPN Portal</a>


Connecting validated the credentials.  Download the Fortinet VPN Client from their <a href="https://www.fortinet.com/support/product-downloads" target="_blank">Product Downloads</a> page or in the portal, from the *Download FortiClient* dropdown list.

   ![VPN-Download](images/vpn-download.png)





