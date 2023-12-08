# Miscellaneous Notes

## Keys to the Kingdom

For security purposes, all administrative and root logins are kept in BitWarden for safe keeping.  This is an enterprise approved password management application.  BitWarden is integrated with Red Hat Rover Groups for access management.

### Red Hat Rover Groups

You can find Rover Groups in the <a href="https://rover.redhat.com/apps/ target="_blank"> Rover </a>application.  In the Search bar, enter groups.  

Access <a href="https://rover.redhat.com/groups/" target="_blank"> Rover Groups </a>; the main page will display your group membership.  The `na-ssa-infrastructure` group is the group used for BitWarden membership.  Current owners of the group are Ken, Chris, and Peter.  They can help with any membership additions and deletions.

### Join the Group E-Mail

Once a user is added to the group, they will recieve an e-mail to accept the invitation.  Until they join, they won't be in the group.


## BitWarden

<a href="https://vault.bitwarden.com/#/login" target="_blank">BitWarden </a> can be access via your favorite browser.  If necessary create an account using your @redhat.com email address.  

The *Na Ssa Infrastructure* Collection has all of the lab's passwords, keys, and secrets.  You can create your own Folders to organize the secrets.  Keep in mind, the *Folders* are only available to you.  They are not shared folders with the *na-ssa-infrastructure* group, just the collection is shared.  Feel free to organize however you want. 


## Weblinks & ISOs

The rhel8 repo server (172.20.129.19) has an alias created for it for [www.openinfra.lab](http://172.20.129.19/) and has a number of useful links on it.

Additionally, if you host your ISO files on this server in `/var/www/html`, they will be auto-populated on this page to make copying of the links to them useful for pasting into IMM or any other UI where you are prompted for the URL to an ISO.

## [Gitea](http://172.20.129.19/)

This git installation is running as an Operator within the [Openshift Baremetal Cluster](https://console-openshift-console.apps.ocp-bm.openinfra.lab/).  In order to customize the deployment to enable SSH access, the default configuration needed to be modified.  This was accomplished by:

1. Under *Workloads->ConfigMaps* in the **gitea** namespace, examine the config ini that was being used.  In our case it was the **cloud-infra-git-config** ConfigMap.

2. Create a new ConfigMap with the contents of the previous config and any changes that are neeeded.  We called this **nassa-gitea-static-config**.

3. Under *Operators->Installed Operators* select the **Gitea Operator** and go to the *Gitea* tab.

4. Select **cloud-infra-git**, go to YAML and added the following line to the spec:

```
  giteaConfigMapName: nassa-gitea-static-config
```

