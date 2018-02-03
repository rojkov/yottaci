What is YottaCI?
================

It's a set of scripts and deployment descriptions that can be used to
set up a continuous integration system in Azure specialized for
Yocto/OpenEmbedded projects. The scripts are split in three
submodules:

  * `azure-deployment` - a script deploying YottaCI infra in Azure Cloud;
  * `azure-backend` - the actual code;
  * `azure-webhook` - webhooks serving requests from Github.

It is possible to configure YottaCI as a Github application available
publicly or privately. Look at https://github.com/apps/yottaci and
https://github.com/apps/yottaci-staging as examples. The latter
Github app is the staging setup used for pre-production testing
of YottaCI.

Deployement
===========

Run `azure-deployment/yottaci-deployment.sh` to deploy YottaCI to Azure Cloud.
The script requires your Azure subscription ID and YottaCI's resource group
name as command line parameters. Also you have to update
`azure-deployment/yottaci-template.json` with values for

  * `githubapp_production_issuer_id` - ID of the production Github app.
    It can be found at the application's settings page;
  * `githubapp_staging_issuer_id` - ID of the staging Github app;
  * `githubapp_production_pem` - Github app's private key used
    authenticate production YottaCI in Github;
  * `githubapp_staging_pem` - Github app's private key used
    authenticate staging YottaCI in Github;
  * `client_id` - your Azure client ID;
  * `client_secret` - your Azure client secret;
  * `tenant_id` - your Azure tenant ID;
  * `subscription_id` - your Azure subscription ID.

The expected result is successful creation of 3 Azure Function Apps:

  * `webooks` which contains only one function serving as a Github
    trigger to translate Github events to queue messages;
  * `webhooksstaging` which is the same as the first one, but
    used for preproduction testing of YottaCI;
  * `backend` which contains functions actually handling Github events.

The `backend` function app also has a `staging` slot which is used
to test new backend code. If everything is OK the code can be swapped
into the production slot. Then the old code from the production slot
gets placed to `staging` and can be swapped back again into production.

How to configure a project's repo
=================================

To configure YottaCI do builds for your git repository place a file
`.yottaci.yml` into the repository's root. The content should conform the
YAML format.

You may have multiple YAML documents in the file. Every document triggers
a separate build.

Here is the example::

	---
	bitbake_target: curl-native
	bitbake_ref: 1.34
	oecore_ref: pyro
	localconf: |
	  LICENSE_FLAGS_WHITELIST = "commercial"
	  SUPPORTED_RECIPES_CHECK = ""
	dependencies:
	  - url: git://git.openembedded.org/meta-openembedded
	    ref: pyro
	    layers:
	      - meta-oe
	---
	bitbake_target: curl-native
	configuration_name: latest
	bitbake_ref: master
	oecore_ref: master
	localconf: |
	  LICENSE_FLAGS_WHITELIST = "commercial"
	dependencies:
	  - url: git://git.openembedded.org/meta-openembedded
	    ref: master
	    layers:
	      - meta-oe

bitbake_target
    This is the only mandatory setting. Defines bitbake's target to build.

bitbake_ref
    Git reference for the bitbake tool repository.

oecore_ref
    Git reference for the Openembedded-Core repository.

localconf
    Custom additions to the local.conf file.

dependencies
    List of Git repositories containing meta-layers needed for your meta-layer
    to build successfully. If a repository contains several meta-layers in
    subfolders then those subfolders need to be listed in the option layers.
    Otherwise the repository's root is treated as a required meta-layer.


Known issues
============

1. It may happen that the SMB share where YottaCI stores bitbake
   downloads reports 0Mb free space due to internal maintenance probably.
   This makes bitbake report build failure immidiately.
2. A race condition may happen when multiple builds do git clones at
   the same time.
3. Sometime copying bitbake cache from a builder to the SMB share fails
   if the size of the cache gets bigger than 30G.

