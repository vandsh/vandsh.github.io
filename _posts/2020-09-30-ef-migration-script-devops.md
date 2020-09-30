---
layout: post
title: "EF Migrations Database Script deployment Azure DevOps"
category: Azure
tags: Azure EF DevOps
---

Working on a recent project that has a small database to store data for post processing, we turned to [ef for dotnet core](https://docs.microsoft.com/en-us/ef/core/).  We used a `code-first` approach and leveraged `migrations`. The only last piece we had to figure out is how to have these changes make it to our servers via `DevOps`.

Thankfully there is a nice little `cli` for ef; [`dotnet-ef`](https://docs.microsoft.com/en-us/ef/core/miscellaneous/cli/dotnet).  The cli made it easy to include new migrations/features into the project and also help us facilitate getting those changes out into the wild.  Below are a few of the commands and snippets we used to accomplish this:

## Basic usage

Below covers just some of the basic cli commands to get up and running.

### Installing dotnet-ef

Pretty basic, but just getting it installed for use later...

```powershell
dotnet tool install --global dotnet-ef
```

### Adding a migration

Adding a new migration; could be a new "feature" or a set of significant changes

```powershell
dotnet ef migrations add FeatureName
```

### Updating database

This command will actually update the database that is part of the current project, given the current migration files.

```powershell
dotnet ef database update --project .\src\YourProject.csproj
```

## DevOps usage

Assuming you are now at a point where you have your migrations in source and are ready to deploy, below should help guide you thru getting those changes out onto servers.

### Create a migration script

After all your standard dotnet core build tasks, add a `powershell` task like so:

```yaml
- task: PowerShell@2
  inputs:
    targetType: 'inline'
    script: |
      dotnet tool install --global dotnet-ef
      dotnet ef migrations script --idempotent --project $(Build.SourcesDirectory)\src\YourProject.csproj --output $(Build.ArtifactStagingDirectory)/db.sql
 
``` 

What the above does is install the `dotnet-ef` cli within the context of the hosted agent and then use it to create a single script to encompass all the migration changes. As you can see we specify the `--project`, the `--output` folder so it gets picked up by the `publishBuildArtifacts` task and a `--idempotent` flag. The `--idempotent` flag encompasses all changes with a check to the `__EFMigrationsHistory` to see what version of migration it is on, thus allowing you to both create from scratch and iterate over an existing database with the same script:

```sql
IF NOT EXISTS(SELECT * FROM [__EFMigrationsHistory] WHERE [MigrationId] = N'20200923211638_InitialCreate')
```

Now that we have a single script that can handle all things database schema, let's make sure the release is capable of executing this.

### Deploy database changes in Release

Assuming you are at a point where all your web assets are already being deployed successfully, let's find the `SQL Server database deploy` task. Search for it by name in the task search and add it to your release.  This task is able to deploy `dacpac` files as well as `sql` scripts, we will be using the latter. After adding the task, populate it using the following:

 - Deploy SQL using: `Sql Query File`
 - Sql File: `$(System.DefaultWorkingDirectory)\_YourProject\drop\db.sql`
 - Execute within a transaction: `true`
 - Server Name: `<the sql server instance, with respect to where your release agent is running>`
 - Database Name: `NameOfYourDatabase` 
   - Make sure the database at exists 
   - If using `Windows Authentication`, make sure the user that the release agent runs as has access to this database.

Send it.

You should now be able to encompass all your database changes in source control, generate a change script that respects the database version and effectively deploy those changes to the database.

Enjoy!
