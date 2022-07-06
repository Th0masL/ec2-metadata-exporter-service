# ec2-metadata-exporter-service

This Python script is used to monitor and extract the Metadata and Tags from within an AWS EC2 Instance, and expose them to internal processes by exporting them to a local Environment Variables file.

The script keep running in the background to make sure that any changes to the EC2 Tags is being reflected in the local file.

You can then use this local Environment Variable file in other Linux services and scripts, allowing dynamic control of your services and script by simply updating the values of the EC2 Tag.

## Requirements

- You need to [allow the access to Tags in the EC2 Instance Metadata.](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html#allow-access-to-tags-in-IMDS)
- [ec2-metadata](https://github.com/adamchainz/ec2-metadata)
    - Install: `pip3 install ec2-metadata`
- psutil
    - Install: `pip3 install psutil`

## Installation

1. Create the folder `/etc/ec2-metadata-exporter`
2. Copy the config file `config.json.example` to `/etc/ec2-metadata-exporter/config.json`
3. Copy the script file `ec2-metadata-exporter` to `/usr/local/bin/ec2-metadata-exporter`
4. Copy the service file `ec2-metadata-exporter.service` to `/lib/systemd/system/ec2-metadata-exporter.service`
5. Reload the SYSTEMD Daemon with `systemctl daemon-reload`
6. Start the service with `service ec2-metadata-exporter start`
7. Verify that the service works correctly:
    - Check the logs with `journalctl -u ec2-metadata-exporter`
    - And verify the file that has been created (Default location: `/etc/aws-ec2-metadata.env`)
8. Enable to service on boot with `systemctl enable ec2-metadata-exporter`
9. Update your other services to start using this new Environment Variable file

## Overview

### Config file

The default location for the config file is `/etc/ec2-metadata-exporter/config.json`.

```json
{
    // Name to give to the variable that will contains the name of your AWS Account
    "aws_account_var_name": "ENV",
    // ID<>Name of your AWS Accounts, based on the Account ID
    "aws_accounts_id_name_map": "prod=123456789012;dev=234567890123",
    // Allow to override the name of the CONTINENT variable
    "continent_var_name": "CONTINENT",
    // Custom prefix for the variables CLOUD, CONTINENT and REGION
    "custom_prefix": "",
    // Allow to override the export location
    "export_to_path": "/etc/aws-ec2-metadata.env",
    // Define how often the Metadata/Tags are refreshed
    "refresh_time_seconds": 60,
    // Allow to override the name of the REGION variable
    "region_var_name": "REGION",
    // Enable/disable the export of some static metadata variables
    "static_metadata_to_export": [
        "AWS_ACCOUNT_ID",
        "AWS_ACCOUNT_NAME",
        "AWS_AMI_ID",
        "AWS_INSTANCE_ID",
        "AWS_INSTANCE_PROFILE_NAME",
        "AWS_INSTANCE_TYPE",
        "AWS_PRIVATE_IPV4",
        "AWS_PUBLIC_IPV4",
        "CLOUD",
        "CONTINENT",
        "REGION",
        "VCPU"
    ]
}
```

### Example of the file that got created

```bash
> cat /etc/aws-ec2-metadata.env
AWS_ACCOUNT_ID='234567890123'
AWS_AMI_ID='ami-466f9156da8a0d61a'
AWS_INSTANCE_ID='i-55a78429f6647ba02'
AWS_INSTANCE_TYPE='t3.small'
AWS_PRIVATE_IPV4='172.31.8.5'
CONTINENT='eu' # Generated from the Region
ENV='dev'
HOSTNAME='test-server'
SOMEVAR1='somevalue1'
SERVER_TYPE='webserver'
NAME='test-server'
REGION='eu-west-2'
VCPU='2'
```

### How to pass those Environment Variables to another Service

1. Let's imagine that your server is an HAProxy Server, and you want to inject the Environment Variables into the HAProxy process
2. Edit the HAProxy Service file in `/lib/systemd/system/haproxy.service`
3. Under the `[Service]` section:
    - Add the line `ExecStartPre=/usr/bin/test -f /etc/aws-ec2-metadata.env` to tell HAProxy that it cannot start as long as the file `/etc/aws-ec2-metadata.env` does not exist
    - Add the line `EnvironmentFile=/etc/aws-ec2-metadata.env` to tell HAProxy to load the Environment Variables from this file
    - Add or edit the line `Restart` and set the value to `always`, to ensure that SYSTEMD keeps retrying if the file `/etc/aws-ec2-metadata.env` is not present just yet
4. Save your changes and reload SYSTEMD Daemon with `systemctl daemon-reload`
5. The values of the AWS EC2 Metadata and Tags are now available inside the HAProxy Process


### Dynamic refresh/update of the EC2 Tags into the Environment Variables file

If you never update the EC2 Tags on running EC2 Instances, you don't really need to automatically refresh the list of EC2 Tags often at all, as the first Enviroment Variable that is generated when the service starts will already contains and export everything you want to export to the Environment Variable file.

But if you update the EC2 Tags often, you definitely want to refresh the list of EC2 Tags every couple of minutes, to detect possible changes.

And depending on the service you are injecting those Environment Variables into, you might have to do some additionnal SYSTEMD configuration if you want your service has to be notified and reload the new values of the Environment Variable file when then EC2 Tags were added/removed/edited.

In order to do that, you can create a 'watcher' service that will monitor the file `/etc/aws-ec2-metadata.env` and automatically trigger a reload or a restart of another service when the Environment Variable file has been updated.

See [this thread](https://superuser.com/questions/1171751/restart-systemd-service-automatically-whenever-a-directory-changes-any-file-ins) for more information about this approach.

### Overview of Syslog

- Startup logs

    ```log
    > service ec2-metadata-exporter start
    > journalctl -u ec2-metadata-exporter
    Jul 06 08:05:29 ip-172-31-8-5 systemd[1]: Started Metadata Exporter Service for AWS EC2 Instances.
    Jul 06 08:05:29 ip-172-31-8-5 ec2-metadata-exporter[5898]: Script starting, loading the configuration from /etc/ec2-metadata-exporter/config.json ...
    Jul 06 08:05:30 ip-172-31-8-5 ec2-metadata-exporter[5898]: Creating the Env Var file /etc/aws-ec2-metadata.env for the first time. Found a total of 13 tags. Check frequency: 60 seconds.
    <...>
    ```

- Shutdown logs

    ```log
    > service ec2-metadata-exporter stop
    > journalctl -u ec2-metadata-exporter
    Jul 06 08:07:47 ip-172-31-8-5 systemd[1]: Stopping Metadata Exporter Service for AWS EC2 Instances...
    Jul 06 08:07:48 ip-172-31-8-5 ec2-metadata-exporter[5898]: Received kill signal. Exiting gracefully.
    Jul 06 08:07:48 ip-172-31-8-5 systemd[1]: ec2-metadata-exporter.service: Succeeded.
    Jul 06 08:07:48 ip-172-31-8-5 systemd[1]: Stopped Metadata Exporter Service for AWS EC2 Instances.
    ```
