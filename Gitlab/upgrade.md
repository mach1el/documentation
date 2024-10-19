# GitLab CE Upgrade Guide

Upgrading GitLab CE (Community Edition) is a multi-step process that requires careful planning to ensure minimal downtime and avoid data loss. Below is a **step-by-step guide** to upgrade GitLab CE.

## Prerequisites

1. **Backup**: Ensure you have a full backup of your GitLab instance (including configuration files, database, and Git repositories).
2. **Test Environment**: Ideally, you should first test the upgrade on a staging environment before applying it to production.
3. **Admin Access**: Ensure you have root or sudo privileges on the server where GitLab CE is installed.
4. **Version Compatibility**: Always check the official GitLab upgrade path to ensure that you can safely upgrade from your current version to the desired version.

---

## Step 1: Backup GitLab

The most crucial step before any upgrade is to back up your GitLab instance.

Run the following command to create a backup:

```bash
sudo gitlab-backup create
```

You can find the backup file in /var/opt/gitlab/backups/ by default. The backup will include repositories, database, and uploads.

Additionally, backup your configuration files:

```bash
sudo cp /etc/gitlab/gitlab.rb /etc/gitlab/gitlab.rb.bak
sudo cp /etc/gitlab/gitlab-secrets.json /etc/gitlab/gitlab-secrets.json.bak
```

## Step 2: Check for GitLab CE Version Information

Before upgrading, check your current GitLab version:

```bash
sudo gitlab-rake gitlab:env:info
```
> Note the version, as GitLab may require incremental upgrades, particularly when upgrading across major versions.

## Step 3: Stop GitLab Services (Optional for Safety)

You may stop GitLab services to prevent users from interacting with the system during the upgrade:

```bash
sudo gitlab-ctl stop
```
> This will halt all GitLab processes, including Sidekiq, Unicorn, and PostgreSQL.

## Step 4: Check GitLab CE Upgrade Path

Refer to the official GitLab upgrade guide for any version-specific instructions:

  * [GitLab CE Upgrading Docs](https://docs.gitlab.com/ee/update/)

If you're upgrading across major versions (e.g., from 13.x to 14.x), you may need to upgrade through intermediate versions (e.g., from 13.x to 13.12.x first).

## Step 5: Update the GitLab CE Repository

Depending on your operating system, you may need to update your GitLab CE package repository.

### For Ubuntu or Debian:

  1. Open the GitLab package repository file:

  ```bash
  sudo nano /etc/apt/sources.list.d/gitlab_gitlab-ce.list
  ```

  2. Ensure the following entry exists (replace <version> with your desired version):
  
  ```bash
  deb https://packages.gitlab.com/gitlab/gitlab-ce/ubuntu focal main
  ```

  3. Update the system and repository:

  ```bash
  sudo apt-get update
  ```

### For CentOS or RHEL:

Ensure the GitLab package repository is properly configured by checking /etc/yum.repos.d/gitlab_gitlab-ce.repo.

* Update the repository:

```bash
sudo yum clean all
sudo yum makecache
```

## Step 6: Upgrade GitLab CE Package

### For Ubuntu/Debian:

1. Run the following command to upgrade GitLab CE:

```bash
sudo apt-get install gitlab-ce
```

2. You may also specify a version if needed:

```bash
sudo apt-get install gitlab-ce=<version>
```

### For CentOS/RHEL:

1. Upgrade GitLab CE using yum:

```bash
sudo yum install gitlab-ce
```

2. To upgrade to a specific version:

```bash
sudo yum install gitlab-ce-<version>
```

## Step 7: Reconfigure GitLab

After upgrading, you must reconfigure GitLab to apply the new configurations:

```bash
sudo gitlab-ctl reconfigure
```
> This command will apply any necessary changes, update configurations, and restart the GitLab services.

## Step 8: Check GitLab Status

Once the reconfiguration is complete, check if all GitLab services are running correctly:

```bash
sudo gitlab-ctl status
```

You can also use the following command to verify the environment information and confirm the upgraded version:

```bash
sudo gitlab-rake gitlab:env:info
```

## Step 9: Run Database Migrations

GitLab upgrades may require database migrations, especially if you're upgrading across major versions. You can manually run the migrations using the following command:

```bash
sudo gitlab-rake db:migrate
```

## Step 10: Clean Up Unnecessary Files

Once the upgrade is successful, you can clean up old files and repositories:

```bash
sudo gitlab-ctl cleanse
```
> This will clean up obsolete and temporary files left behind during the upgrade process.

## Step 11: Verify GitLab Functionality

Finally, verify that GitLab is running properly:

* Access the GitLab web interface and ensure everything is working (e.g., repositories, CI/CD pipelines, etc.).
* Check logs for any errors or warnings:

  ```bash
  sudo gitlab-ctl tail
  ```

## Step 12: Re-enable Services (If Stopped)

If you stopped GitLab services before the upgrade, you can start them again:

```bash
sudo gitlab-ctl start
```

# Notes on Upgrading Across Major Versions

* If you're upgrading across major versions (e.g., from 12.x to 14.x), consult the official GitLab documentation for any breaking changes or special upgrade paths.

* GitLab typically requires intermediate upgrades for major versions, so you might need to upgrade to the latest minor version of the current major version before jumping to the next major version.

# Rollback (In Case of Failure)

If something goes wrong during the upgrade, you can roll back using your backups:

1. Restore from Backup:
   * If the upgrade fails and you need to roll back, follow the GitLab backup restore procedure.

    ```bash
    sudo gitlab-backup restore BACKUP=your-backup-timestamp
    ```

2. Restore Configurations:

    * If necessary, restore your configuration files from the backup:

    ```bash
    sudo cp /etc/gitlab/gitlab.rb.bak /etc/gitlab/gitlab.rb
    sudo cp /etc/gitlab/gitlab-secrets.json.bak /etc/gitlab/gitlab-secrets.json
    ```

3. Reconfigure GitLab:

    * Reconfigure GitLab after restoring:

    ```bash
    sudo gitlab-ctl reconfigure
    ```