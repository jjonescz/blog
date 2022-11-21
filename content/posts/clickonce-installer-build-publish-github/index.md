---
title: Build and publish ClickOnce installer on GitHub
slug: clickonce-installer-build-publish-github
date: 2022-11-21T16:47:00+01:00
summary: How to automatically detect form changes and prevent users from losing
  entered data or submitting empty forms.
tags:
  - ClickOnce
  - WPF
  - Windows
  - .NET
categories:
  - Programming
# cover:
#   image: cover.png
#   alt: Prevent user from losing changes in a form
#   relative: true
ShowToc: false
---

This post summarizes how one can build a [ClickOnce package](https://learn.microsoft.com/en-us/visualstudio/deployment/quickstart-deploy-using-clickonce-folder?view=vs-2022)
automatically in GitHub Actions and then automatically publish it to GitHub Pages.

Assuming you have a WPF application in a GitHub repository.

1. Create branch `gh-pages`,
   [enable GitHub Pages](https://docs.github.com/en/pages/quickstart)
   and set it to publish from `gh-pages` branch.

2. In your `gh-pages` branch, add `.gitattributes` with the following content:

   ```gitattributes
   * -text
   ```

   to prevent Git from messing with line endings in your release files.

   Also add an empty [`.nojekyll` file](https://github.blog/2009-12-29-bypassing-jekyll-on-github-pages/)
   to prevent GitHub from processing your release files before publishing them.

3. In your `main` branch, create `./release.ps1` script as below
   (don't forget to change the two properties at the beginning marked with "👈").
   Optionally, you can embed this code directly into the GitHub Action below,
   but then you won't be able to run the release locally which might be useful.

   ```ps1
   [CmdletBinding(PositionalBinding=$false)]
   param (
       [switch]$OnlyBuild=$false
   )

   $appName = "WpfApplication" # 👈 Replace with your app name.
   $projDir = "src/$appName" # 👈 Replace with your project directory.

   Set-StrictMode -version 2.0
   $ErrorActionPreference = "Stop"

   Write-Output "Working directory: $pwd"

   # Find MSBuild.
   $msBuildPath = & "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" `
       -latest -requires Microsoft.Component.MSBuild -find MSBuild\**\Bin\MSBuild.exe `
       -prerelease | select-object -first 1
   Write-Output "MSBuild: $((Get-Command $msBuildPath).Path)"

   # Load current Git tag.
   $tag = $(git describe --tags)
   Write-Output "Tag: $tag"

   # Parse tag into a three-number version.
   $version = $tag.Split('-')[0].TrimStart('v')
   $version = "$version.0"
   Write-Output "Version: $version"

   # Clean output directory.
   $publishDir = "bin/publish"
   $outDir = "$projDir/$publishDir"
   if (Test-Path $outDir) {
       Remove-Item -Path $outDir -Recurse
   }

   # Publish the application.
   Push-Location $projDir
   try {
       Write-Output "Restoring:"
       dotnet restore -r win-x64
       Write-Output "Publishing:"
       $msBuildVerbosityArg = "/v:m"
       if ($env:CI) {
           $msBuildVerbosityArg = ""
       }
       & $msBuildPath /target:publish /p:PublishProfile=ClickOnceProfile `
           /p:ApplicationVersion=$version /p:Configuration=Release `
           /p:PublishDir=$publishDir /p:PublishUrl=$publishDir `
           $msBuildVerbosityArg

       # Measure publish size.
       $publishSize = (Get-ChildItem -Path "$publishDir/Application Files" -Recurse |
           Measure-Object -Property Length -Sum).Sum / 1Mb
       Write-Output ("Published size: {0:N2} MB" -f $publishSize)
   }
   finally {
       Pop-Location
   }

   if ($OnlyBuild) {
       Write-Output "Build finished."
       exit
   }

   # Clone `gh-pages` branch.
   $ghPagesDir = "gh-pages"
   if (-Not (Test-Path $ghPagesDir)) {
       git clone $(git config --get remote.origin.url) -b gh-pages `
           --depth 1 --single-branch $ghPagesDir
   }

   Push-Location $ghPagesDir
   try {
       # Remove previous application files.
       Write-Output "Removing previous files..."
       if (Test-Path "Application Files") {
           Remove-Item -Path "Application Files" -Recurse
       }
       if (Test-Path "$appName.application") {
           Remove-Item -Path "$appName.application"
       }

       # Copy new application files.
       Write-Output "Copying new files..."
       Copy-Item -Path "../$outDir/Application Files","../$outDir/$appName.application" `
           -Destination . -Recurse

       # Stage and commit.
       Write-Output "Staging..."
       git add -A
       Write-Output "Committing..."
       git commit -m "Update to v$version"

       # Push.
       git push
   } finally {
       Pop-Location
   }
   ```

4. Create a GitHub Action workflow file `.github/workflows/release.yml`:

   ```yaml
   name: Release

   on:
     push:
       tags: [v*]

   jobs:
     release:
       runs-on: windows-latest

       steps:
         - name: Checkout code
           uses: actions/checkout@v3

         - name: Setup Git
           run: |
             git config --global url."https://user:${{ secrets.GITHUB_TOKEN }}@github".insteadOf https://github
             git config --global user.name github-actions
             git config --global user.email github-actions@github.com

         - name: Run release script
           shell: pwsh
           run: ./release.ps1
   ```

5. This will run whenever you [publish a release](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)
   using the GitHub UI or manually push a tag
   (which is what publishing the release does under the hood):

   ```sh
   git tag v1.0.0
   git push origin v1.0.0
   ```

   When it runs, it builds your app, creates a ClickOnce package,
   and pushes it to the `gh-pages` branch, so it's available for download at:

   ```url
   https://{user}.github.io/{repo}/{app}.application
   ```

   (Replace `{user}` by your GitHub username, `{repo}` by the repository name
   and `{app}` by `$appName` from the `release.ps1` script above.)
