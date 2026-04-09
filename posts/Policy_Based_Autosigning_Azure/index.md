---
layout: post
title: "Policy-based Autosigning in Azure with the Azure CLI and Virtual Machine Tags"
date: 2016-06-30
author: Keiran
---

*Originally published on the Puppet blog, June 2016.*

When developing a Puppet-based solution that uses a Puppet master, one of the initial challenges you need to solve is how your agent certificates are going to be signed. To quote the Puppet documentation:

> Before Puppet agent nodes can retrieve their configuration catalogs, they need a signed certificate from the local Puppet certificate authority (CA). When using Puppet's built-in CA, agents will submit a certificate signing request (CSR) to the CA Puppet master and will retrieve a signed certificate once one is available.

You need to take care when it comes to certificate signing, as it provides one of the core layers of defence against rogue or unauthorized Puppet agents requesting catalogs from your master. This is an issue because catalogs often contain secrets or other information about your infrastructure that should be exposed only to authorized nodes.

There are a number of ways to sign incoming certificate requests:

* **Interactively** — The Puppet Enterprise console or the `puppet cert` command. In environments that have lower levels of automation, it's common for certificates to be signed manually by an operator who validates and approves the request.
* **Programmatically** — The Puppet CA API is excellent for automation in certain cases — for example, when you're using a provisioning tool/orchestrator where agent certificate requests can be polled and signed programmatically without human intervention.
* **Automatically** — Policy-based autosigning involves the execution of an external script to determine if a certificate request should be signed. This is ideal for dynamic environments such as those where instances are provisioned automatically via Resource Group Templates or CloudFormation, where the speed of certificate signing is paramount, or where there is limited infrastructure to run API polling jobs for new certificate requests.

---

## Policy-based Autosigning in Microsoft Azure Environments

In this use case, we will demonstrate how policy-based autosigning can be used to automatically sign Puppet certificates from virtual machines residing in Microsoft Azure after validating that the virtual machine instance meets all the environment's standards.

### Environment Overview

In this demo environment, we provision Windows and Linux instances via Microsoft Azure Resource Manager templates through a CI/CD pipeline. Provisioned virtual machine instances reside in resource groups that match their short (non-FQDN) name.

As every instance provisioned has its Standard Operating Environment (SOE) configured and managed via Puppet, we inject the Puppet agent using a custom script extension on launch, which results in a CSR to the Puppet master on first run. As this is a fully automated environment, we must have the CSR signed as soon as possible. However, before signing the certificate, we want to ensure a few things are in place to ensure there is no degradation to the security of the environment.

Specifically, we want to ensure that:

* The instance's certificate matches the defined hostname standard for the environment.
* The instance resides in our Azure subscription.
* The instance has been tagged correctly as per the tagging standards for the environments.

Fortunately, Puppet's policy-based autosigning functionality allows us to execute a script each time a CSR is received. That lets us evaluate our three requirements above, and sign the certificate if it is deemed suitable.

---

## Programmatically Querying Azure Resources

Azure comes with a rich API and easy-to-use command line tool that allows you to interact with all the resources within your Azure subscriptions. In this case, we'll leverage the CLI to query our subscription's resources directly and use the `--json` flag to return the response to us in JSON for parsing.

Installing and configuring the Azure CLI against your subscription is out of scope for this post. However, please ensure that:

* The Azure CLI is configured to be executed by the user who runs the Puppet server (i.e., `pe-puppet`), as policy-based autosigning scripts are executed by this user. In other words, ensure that `azure login` is run by this user.
* The Azure CLI user has read-only access to your subscription resources. We need only to be able to query the subscription resources, not change them. Follow the least-privilege model.
* The Azure CLI is configured to use ARM mode.

We could have used Azure's REST API or Azure's Ruby SDK, which would have eliminated the need for installing the Azure CLI. However, for the purpose of demonstration, using the Azure CLI is the path of least resistance.

---

## Using Policy-based Autosigning

Policy-based autosigning functions as follows:

* Define a script or binary to be executed by the Puppet CA on receipt of a CSR in the `[main]` section of the `puppet.conf` file.
* On receipt of the CSR, the script is executed, with the certificate name passed as the first (and only) positional argument. The content of the CSR is provided to the script's STDIN in PEM encoded format.
* The script applies logic based on the provided input, determines if the CSR should be signed, and exits accordingly:
  * Exit zero — Sign the certificate request.
  * Exit non-zero — Do not sign the certificate request.

---

## Tying it All Together with an Autosigning Script

Based on the two capabilities above, I've written some simple Ruby to do the following:

* Validate the certificate name passed to the script as `ARGV[0]` to ensure it meets the hostname standards.
* Query the Azure platform and ensure that the requesting instance exists in our subscription (`sourced-dev`).
* Query the Azure platform and ensure the requesting instance has been tagged correctly for the environment, specifically:
  * That the `service_tier` tag exists, and its value matches the regex `^(prod|nonprod|lab)$`
  * The `business_unit` tag exists, and its value matches the regex `^[a-z0-9]{4}$`
  * The `unique_instance_id` tag exists, and its value matches the regex `^[a-z0-9]{3}$`

This is implemented as follows:

**Step 1** — Create the `/opt/autosign/autosign.rb` file with the script contents below. Ensure that the permissions are `pe-puppet:pe-puppet` and `mode 750` for PE deployments.

```ruby
#!/usr/bin/ruby
# Example Puppet Autosign script for Azure
# Keiran Sweet, Sourced Group
#
# Note:
# For example purposes all output goes to STDERR so it appears in the puppetserver.log
#

require 'json'
azuresubscriptionname = 'sourced-dev'

#
# The tags that must be present on an instance alongside the regex to validate
# their values when set.
#
tagregex = {
  'service_tier'        => '^(prod|nonprod|lab)$',
  'business_unit'       => '^[a-z0-9]{4}$',
  'unique_instance_id'  => '^[a-z0-9]{3}$',
}

STDERR.puts ""
STDERR.puts "----------------------------------------------------------------------"
STDERR.puts "Commencing Azure validation for instance #{ARGV[0]}"

# Validate the certificate/hostname being passed to us first.
unless ARGV[0].match('^.*$')
  STDERR.puts " * The certificate / hostname passed to the autosign script DOES NOT match the standard - Exiting"
  exit(1)
else
  STDERR.puts " * The certificate / hostname passed to the autosign script does match the standard"
  servernamearray = ARGV[0].split('.')
  servername = servernamearray[0]
end

STDERR.puts " * Querying the Azure API"
jsonraw = %x(/bin/azure vm list -s #{azuresubscriptionname} #{servername} --json 2>>/dev/null)

if $? == 0
  puts " * The azure CLI command returned zero when querying #{servername} in #{azuresubscriptionname} (Instance found)"
else
  puts " * The Azure CLI returned with non-zero when querying #{servername} in #{azuresubscriptionname} (Instance not found)"
  STDERR.puts "Completed Azure Metadata validation for instance #{ARGV[0]} - NOT Signing CSR"
  STDERR.puts "----------------------------------------------------------------------"
  STDERR.puts ""
  exit(1)
end

json = JSON.parse(jsonraw)

# We now validate each of the tags to make sure that they match the regex defined for each
# tag in the tagregex hash.
tagregex.keys.each { |tagname|
  if json[0]['tags'].key?(tagname)
    if json[0]['tags'][tagname].match(tagregex[tagname])
      STDERR.puts "  * The #{tagname} MATCHES regex #{tagregex[tagname]} (Value: #{json[0]['tags'][tagname]})"
    else
      STDERR.puts "  * The #{tagname} DOES NOT MATCH regex #{tagregex[tagname]} (Value: #{json[0]['tags'][tagname]})"
      STDERR.puts "  * Aborting further tag evaluation"
      STDERR.puts "Completed Azure Metadata validation for instance #{ARGV[0]} - NOT Signing CSR"
      STDERR.puts "----------------------------------------------------------------------"
      STDERR.puts ""
      exit(1)
    end
  else
    STDERR.puts "  * The tag #{tagname} DOES NOT exist and is mandatory"
    STDERR.puts "Completed Azure Metadata validation for instance #{ARGV[0]} - NOT Signing CSR"
    STDERR.puts "----------------------------------------------------------------------"
    STDERR.puts ""
    exit(1)
  end
}
STDERR.puts "Completed Azure Metadata validation for instance #{ARGV[0]} - Signing CSR"
STDERR.puts "----------------------------------------------------------------------"
STDERR.puts ""
```

**Step 2** — Update the `puppet.conf` file's `[master]` section and reference our autosign script:

```ini
[master]
autosign = /opt/autosign/autosign.rb
```

**Step 3** — Restart your Puppet server process to ensure that your changes have taken effect:

```bash
systemctl restart pe-puppetserver
```

---

## Testing and Results

Now that our policy script is in place, we can launch an instance in Azure and have the instance bootstrap the Puppet agent. Based on where the instance resides and how it is configured, the CSR will be signed, or not signed.

**Test 1** — The instance does not exist in our subscription. Result: do not sign certificate (exit 1).

```text
Commencing Azure validation for instance pzzzzz9s99zp23s.nxj5ujpspjpudkyjyxa00eo44e.cx.internal.cloudapp.net
 * The certificate / hostname passed to the autosign script does match the standard
 * Querying the Azure API
 * The Azure CLI returned with non-zero when querying pzzzzz9s99zp23s in sourced-dev (Instance not found)
Completed Azure Metadata validation for instance pzzzzz9s99zp23s.nxj5ujpspjpudkyjyxa00eo44e.cx.internal.cloudapp.net - NOT Signing CSR
----------------------------------------------------------------------
```

**Test 2** — The instance resides in our subscription but is missing the mandatory tag `missing_tag`. Result: do not sign certificate (exit 1).

```text
Commencing Azure validation for instance pzzzzz9s99zp23s.nxj5ujpspjpudkyjyxa00eo44e.cx.internal.cloudapp.net
 * The certificate / hostname passed to the autosign script does match the standard
 * Querying the Azure API
 * The azure CLI command returned zero when querying pzzzzz9s99zp23s in sourced-dev (Instance found)
  * The service_tier MATCHES regex ^(prod|nonprod|lab)$ (Value: prod)
  * The business_unit MATCHES regex ^[a-z0-9]{4}$ (Value: zzzz)
  * The unique_instance_id MATCHES regex ^[a-z0-9]{3}$ (Value: 23s)
  * The tag missing_tag DOES NOT exist and is mandatory
Completed Azure Metadata validation for instance pzzzzz9s99zp23s.nxj5ujpspjpudkyjyxa00eo44e.cx.internal.cloudapp.net - NOT Signing CSR
----------------------------------------------------------------------
```

**Test 3** — The instance resides in our subscription and has all required tags, but tag values don't match the defined regex. Result: do not sign certificate (exit 1).

```text
Commencing Azure validation for instance pzzzzz9s99zp23s.nxj5ujpspjpudkyjyxa00eo44e.cx.internal.cloudapp.net
 * The certificate / hostname passed to the autosign script does match the standard
 * Querying the Azure API
 * The azure CLI command returned zero when querying pzzzzz9s99zp23s in sourced-dev (Instance found)
 * The service_tier DOES NOT MATCH regex ^(prod|nonprod|lab)$ (Value: keiran)
 * Aborting further tag evaluation
Completed Azure Metadata validation for instance pzzzzz9s99zp23s.nxj5ujpspjpudkyjyxa00eo44e.cx.internal.cloudapp.net - NOT Signing CSR
----------------------------------------------------------------------
```

**Test 4** — The instance meets all our requirements! Result: sign the CSR (exit 0).

```text
Commencing Azure validation for instance pzzzzz9s99zp23s.nxj5ujpspjpudkyjyxa00eo44e.cx.internal.cloudapp.net
 * The certificate / hostname passed to the autosign script does match the standard
 * Querying the Azure API
 * The azure CLI command returned zero when querying pzzzzz9s99zp23s in sourced-dev (Instance found)
 * The service_tier MATCHES regex ^(prod|nonprod|lab)$ (Value: prod)
 * The business_unit MATCHES regex ^[a-z0-9]{4}$ (Value: zzzz)
 * The unique_instance_id MATCHES regex ^[a-z0-9]{3}$ (Value: 23s)
Completed Azure Metadata validation for instance pzzzzz9s99zp23s.nxj5ujpspjpudkyjyxa00eo44e.cx.internal.cloudapp.net - Signing CSR
----------------------------------------------------------------------
```

---

## Summary

As you can see from the above, using policy-based autosigning provides a fast, flexible and secure way to automatically sign your Puppet certificate requests for instances that reside on the Microsoft Azure platform, or any platform that can programmatically deploy an instance.

*Keiran Sweet is a consultant with Sourced Group and is currently based in Toronto, Canada. He works with customers to automate more and integrate with next-generation technologies. He has been using Puppet for seven years and presented a talk, [Order in a World of Snowflakes](https://www.youtube.com/watch?v=d9T80hDDZNA), at PuppetConf 2015.*
