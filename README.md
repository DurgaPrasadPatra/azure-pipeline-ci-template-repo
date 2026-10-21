# azure-pipeline-ci-template-repo
reusable template ci pipeline for war based project
# Azure DevOps template: line-by-line explanation and improved version

Your template performs two main operations:

1. ArchiveFiles\@2 — packages files from the pipeline's working directory into a ZIP archive.
2. AzureWebApp\@1 — deploys that ZIP package to an Azure App Service web app.

This is a basic CI/CD deployment template. We can improve it by making the packaging more predictable, validating the artifact, and making the deployment easier to troubleshoot.

## 1. Your original template, explained

### Step 1 — Define template parameters

```
parameters:
- name: azureserviceconnection
  type: string

- name: applicationname
  type: string
```

- `parameters:` defines inputs that the main pipeline passes to the template.
- `azureserviceconnection` receives the name of the Azure Resource Manager service connection.
- `applicationname` receives the Azure App Service name.
- `type: string` means both inputs are strings.

Your main pipeline passes these values:

```
parameters:
  azureserviceconnection: $(connection)
  applicationname: $(appname)
```

The template references them with `${{ parameters.azureserviceconnection }}` and `${{ parameters.applicationname }}`.

### Step 2 — Archive the application files

```
- task: ArchiveFiles@2
  inputs:
    rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
    includeRootFolder: false
    archiveType: 'zip'
    archiveFile: '$(System.DefaultWorkingDirectory)/$(Build.BuildId).zip'
    replaceExistingArchive: true
```

| Setting                        | What it does                                                                                            |
| ------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `ArchiveFiles@2`               | Runs version 2 of Azure DevOps' built-in archive task.                                                  |
| `rootFolderOrFile`             | Specifies the folder or file to package. Here it uses the pipeline's default working directory.         |
| `includeRootFolder: false`     | Puts the contents of the selected folder at the ZIP root instead of nesting them under the folder name. |
| `archiveType: 'zip'`           | Creates a ZIP archive.                                                                                  |
| `archiveFile`                  | Saves the archive using the build ID as the filename.                                                   |
| `replaceExistingArchive: true` | Replaces an existing archive at that location.                                                          |

Potential problem: `$(System.DefaultWorkingDirectory)` can contain the checked-out source repository and potentially other working files. Archiving the entire directory may include unnecessary files or create an unsuitable package structure.

For a static website, you typically want `index.html`, JavaScript, CSS, and related assets at the ZIP root. For a Java application, you might instead package a WAR or JAR produced by the build.

### Step 3 — Deploy to Azure App Service

```
- task: AzureWebApp@1
  inputs:
    azureSubscription: '${{ parameters.azureserviceconnection }}'
    appType: 'webApp'
    appName: '${{ parameters.applicationname }}'
    package: '$(System.DefaultWorkingDirectory)/*.zip'
    deploymentMethod: 'auto'
```

| Setting                    | What it does                                                         |
| -------------------------- | -------------------------------------------------------------------- |
| `AzureWebApp@1`            | Uses version 1 of the Azure App Service deployment task.             |
| `azureSubscription`        | Identifies the configured Azure Resource Manager service connection. |
| `appType: 'webApp'`        | Targets a standard Azure Web App.                                    |
| `appName`                  | Identifies the target App Service.                                   |
| `package`                  | Specifies the ZIP package to deploy.                                 |
| `deploymentMethod: 'auto'` | Lets the task select an appropriate deployment method.               |

The `azureSubscription` field expects a service connection name, not the subscription ID itself.

The wildcard `*.zip` is less explicit than using the exact ZIP file path. A predictable artifact path helps prevent ambiguity if multiple ZIP files exist.

## `pipeline-template.yaml`

Since your application repository contains files such as `index.html`, `projects.html`, `projects.js`, and `script.js`, I'll assume this is a static website rather than a Java or .NET application.

This version copies only website-related files, excludes Git metadata and unrelated files, creates a predictable ZIP package, validates it, and deploys that exact package.

```

# ============================================================
# TEMPLATE: pipeline-template.yaml
# PURPOSE: Package and deploy a static website to Azure App Service
# ASSUMPTION: The self-hosted agent is running Linux
# ============================================================

# 1. Parameters supplied by the main azure-pipelines.yml
parameters:
  - name: azureserviceconnection
    type: string

  - name: applicationname
    type: string

# 2. Pipeline execution steps
steps:

  # ----------------------------------------------------------
  # STEP 1: Prepare a clean staging directory for website files
  # ----------------------------------------------------------
  - task: Bash@3
    displayName: 'Prepare website staging directory'
    inputs:
      targetType: 'inline'
      script: |
        set -euo pipefail

        # Remove files left in the staging directory, if any.
        rm -rf "$(Build.ArtifactStagingDirectory)/site"

        # Create a directory to hold deployment files.
        mkdir -p "$(Build.ArtifactStagingDirectory)/site"

        echo "Website staging directory prepared."

  # ----------------------------------------------------------
  # STEP 2: Copy only website files; do not package .git
  # ----------------------------------------------------------
  - task: CopyFiles@2
    displayName: 'Copy website files'
    inputs:
      SourceFolder: '$(Build.SourcesDirectory)'
      Contents: |
        **/*.html
        **/*.css
        **/*.js
        **/*.json
        **/*.png
        **/*.jpg
        **/*.jpeg
        **/*.svg
        **/*.ico
        **/*.webp
        **/*.gif
        **/*.woff
        **/*.woff2
      TargetFolder: '$(Build.ArtifactStagingDirectory)/site'
      CleanTargetFolder: true
      OverWrite: true

  # ----------------------------------------------------------
  # STEP 3: Create a ZIP containing the website files
  # ----------------------------------------------------------
  - task: ArchiveFiles@2
    displayName: 'Create website deployment ZIP'
    inputs:
      rootFolderOrFile: '$(Build.ArtifactStagingDirectory)/site'
      includeRootFolder: false
      archiveType: 'zip'
      archiveFile: '$(Build.ArtifactStagingDirectory)/website.zip'
      replaceExistingArchive: true

  # ----------------------------------------------------------
  # STEP 4: Verify that the ZIP exists and is valid
  # ----------------------------------------------------------
  - task: Bash@3
    displayName: 'Validate deployment package'
    inputs:
      targetType: 'inline'
      script: |
        set -euo pipefail

        PACKAGE="$(Build.ArtifactStagingDirectory)/website.zip"

        # Fail if the archive was not created or is empty.
        test -s "$PACKAGE"

        # Check the archive structure and contents.
        unzip -t "$PACKAGE"

        # Display the packaged files for troubleshooting.
        echo "Files included in deployment package:"
        unzip -l "$PACKAGE"

  # ----------------------------------------------------------
  # STEP 5: Deploy the exact ZIP to Azure App Service
  # ----------------------------------------------------------
  - task: AzureWebApp@1
    displayName: 'Deploy website to Azure App Service'
    inputs:
      azureSubscription: '${{ parameters.azureserviceconnection }}'
      appType: 'webApp'
      appName: '${{ parameters.applicationname }}'
      package: '$(Build.ArtifactStagingDirectory)/website.zip'
      deploymentMethod: 'auto'

```

### Why this version is better

| Improvement                 | Benefit                                                               |
| --------------------------- | --------------------------------------------------------------------- |
| Selective file copying      | Avoids deploying `.git` history and unnecessary pipeline files.       |
| Dedicated staging directory | Separates deployment files from the source checkout.                  |
| Predictable ZIP filename    | Prevents wildcard ambiguity when selecting the package.               |
| Package validation          | Detects missing or corrupted archives before deployment.              |
| Descriptive task names      | Makes pipeline logs easier to understand and troubleshoot.            |
| Fail-fast Bash settings     | Stops the script when a command fails instead of silently continuing. |

Important: The file patterns must match your actual website. If your site uses fonts, PDFs, XML files, or other assets, add the corresponding patterns. If you have a folder such as `assets/` containing files with other extensions, make sure those assets are included too.

The `unzip` command must be installed on your Linux agent. Also, if the repository contains multiple applications or unrelated HTML files, narrow the source and copy patterns to the intended application.

## 3. How the pipeline executes now

1\. Checkout source code

Azure DevOps retrieves the application repository.

2\. Stage website files

Copy HTML, CSS, JavaScript, and selected assets.

3\. Archive and validate

Create `website.zip` and inspect its contents.

4\. Deploy to Azure App Service

Upload the validated ZIP using the configured service connection.

5\. Verify the website

Open the deployed URL and confirm the homepage and linked assets load correctly.

## 4. What should you verify after deployment?

1. Confirm that the pipeline run succeeds.
2. Open the Azure App Service in the Azure portal and check its overview page for the application URL.
3. Open the URL in your browser.
4. If the website fails to load, check App Service logs, deployment logs, application configuration, and the expected location of `index.html`.

A successful deployment task only confirms that the deployment operation completed; it does not guarantee that every page or asset works correctly.

One architectural note: For a pure HTML/CSS/JavaScript website, Azure Static Web Apps or Azure Storage static website hosting may be simpler alternatives to App Service. If you're deliberately practising Azure App Service deployment, however, the template above is a reasonable starting point.
