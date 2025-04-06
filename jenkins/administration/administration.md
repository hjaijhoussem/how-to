# Jenkins Administration

1. [Shared library](#shared-library)
    1. [Folder structure](#folder-structure)
2. [Recovery and Rollback](#recovery-and-rollback)
    1. [Rollback Job and system configuration](#rollback-job-and-system-configuration)


## Shared library
## Folder structure
The Jenkins shared library follows a specific directory structure that helps organize your code effectively:


## Recovery and Rollback

### Rollback Job and system configuration
- `Job Configuration History plugin`: This plugin allows you to **view the history of job and system configurations**, and **revert** to a previous configuration.
doc: [Job Configuration History plugin](https://plugins.jenkins.io/jobConfigHistory/) 

How to use:
1. Install the plugin through Jenkins Plugin Manager
2. After installing, `Job Config History` will be available in the Jenkins Dashboard and job configuration page:

<p align="center">
    <img src="images/image-10.png" width="350" height="300" alt="Job Configuration History Plugin"/>
</p>

3. Click on `Job Config History` to access the history view
4. Use the filters in the left sidebar to narrow down your search:
   - `Show system configs only`: View only system-wide configuration changes
   - `Show job configs only`: View changes made to job configurations
   - `Show created jobs only`: View newly created jobs
   - `Show deleted jobs only`: View removed jobs
   - `Show all configs`: Display all configuration changes
   <p align="center">
    <img src="images/image-12.png" width="350" height="300" alt="Job Configuration History Plugin"/>
   </p>

5. To view a specific configuration change, click on `Show Diffs` button (additions in green, deletions in red)
![alt text](images/image-15.png)

6. To restore a previous configuration:
   - Navigate to the desired version
   - Click the `>>` button or the `Restore this configuration` in the `diff view`	
   - Confirm the restoration when prompted
   ![alt text](images/image-13.png)
   ![alt text](images/image-14.png)
