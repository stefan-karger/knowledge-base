# Windows Node Service Deployment Guide

This guide describes a practical production setup for running a Node.js application on a Windows Server as a Windows service, fronted by IIS, with repeatable deployments, fast rollback, and PowerShell automation.

The core model is:

- Build the app into a deployable artifact.
- Run Node as a Windows service, not from an interactive shell.
- Bind the app to `127.0.0.1` only.
- Put IIS in front for HTTPS, Windows Authentication, and reverse proxying.
- Deploy each release into a new immutable folder.
- Switch a `current` junction during deployment.
- Keep a `previous` junction for rollback.

## Table of Contents

- [Recommended Architecture](#recommended-architecture)
- [Prerequisites](#prerequisites)
- [Node Version Strategy](#node-version-strategy)
- [Folder Structure](#folder-structure)
- [Build Artifact Strategy](#build-artifact-strategy)
- [First-Time Server Setup](#first-time-server-setup)
- [PowerShell Smoke Test](#powershell-smoke-test)
- [WinSW Service Setup](#winsw-service-setup)
- [IIS Reverse Proxy And Windows Authentication](#iis-reverse-proxy-and-windows-authentication)
- [Security Hardening](#security-hardening)
- [PowerShell Scripts](#powershell-scripts)
- [Release, Update, And Rollback Flow](#release-update-and-rollback-flow)
- [Database Migration Strategy](#database-migration-strategy)
- [App-Side SSO Bridge](#app-side-sso-bridge)
- [Operations Checklist](#operations-checklist)
- [Troubleshooting](#troubleshooting)

## Recommended Architecture

Use this shape unless you have a strong reason not to:

```text
Browser
-> HTTPS
-> IIS on Windows Server
-> Windows Authentication
-> URL Rewrite + ARR reverse proxy
-> trusted header injection, for example X-Remote-User
-> Node app on 127.0.0.1:3000
-> database and other backend services
```

Why this model is usually the best fit:

- IIS handles TLS and Windows Authentication well.
- The Node app stays off the public network surface.
- Service startup survives logoff and server reboot.
- Deployments become predictable because you switch releases, not live files.
- Rollback is fast because it is only a junction change plus service restart.

## Prerequisites

Before first deployment, prepare these pieces:

- Windows Server with IIS installed.
- IIS Windows Authentication feature enabled.
- IIS URL Rewrite installed.
- IIS Application Request Routing, ARR, installed.
- A TLS certificate for the public hostname.
- Node.js LTS installed on the server.
- A dedicated service account, or preferably a gMSA in domain environments.
- Network access from the server to the database and any external dependencies.
- A build machine or CI runner that can produce the deployment artifact.

Recommended IIS-related features and software:

```text
IIS
IIS Windows Authentication
IIS URL Rewrite
IIS Application Request Routing
TLS certificate for your public hostname
```

Recommended operational prerequisites:

- A dedicated health endpoint such as `GET /healthz`.
- Centralized or at least retained logs for IIS and the Node service.
- A documented release naming format such as `YYYYMMDD-HHMMSS-gitsha`.
- A clear database migration step in the release pipeline.

## Node Version Strategy

Use different rules for developer machines and production servers.

### Developer And Build Machines

For Windows developer workstations, use a Node version manager.

Good options:

- `nvm-windows` for single-user Windows machines.
- `Volta` if you want a pinned cross-platform toolchain.

If the project uses `pnpm` or `yarn`, enable `corepack` on the build machine.

Example `nvm-windows` flow:

```powershell
nvm install lts
nvm use lts
node --version
npm --version
corepack enable
```

### Production Servers

On the runtime server, prefer a fixed Node.js LTS installation such as the official MSI. Keep the runtime simple.

Avoid `nvm-windows` on shared servers or multi-agent build machines. It works by switching a shared symlink and is not a good fit for shared server scenarios.

For production, the target should be stable and explicit:

```text
C:\Program Files\nodejs\node.exe
```

### Version Pinning Rule

Keep the Node major version pinned in your project and in your runtime documentation.

Example:

```text
Build with Node 22 LTS
Run in production with Node 22 LTS
```

That avoids subtle build-versus-runtime drift.

## Folder Structure

Use a stable service location and immutable release folders.

Recommended layout:

```text
C:\apps\my-node-app\
  incoming\
  releases\
    20260416-153000-a1b2c3d\
      .output\
        server\
          index.mjs
    20260410-093500-f6e7d8c\
      .output\
        server\
          index.mjs
  current\    -> junction to active release
  previous\   -> junction to previous release
  scripts\

C:\services\my-node-app\
  my-node-app.exe
  my-node-app.xml
  logs\
```

What each path is for:

- `incoming` stores copied artifacts before extraction.
- `releases` stores immutable extracted versions.
- `current` is the only app path the service uses.
- `previous` is the last known-good release.
- `scripts` stores deployment and rollback scripts.
- `C:\services\my-node-app` stores the WinSW executable, XML config, and service logs.

Important rules:

- Never edit files in `current` directly.
- Never point the service directly at a specific release folder.
- Always point the service at `current`.
- Keep secrets out of the release folder.

## Build Artifact Strategy

Your artifact should contain only what the app needs at runtime.

Typical cases:

- If your framework outputs a standalone server bundle such as `.output`, deploy that folder.
- If your app runs from `dist\server.js`, deploy `dist` and any required runtime files.
- If the app still needs `node_modules`, either package production dependencies or produce a standalone build first.

Keep configuration outside the artifact.

Examples of values that should not live inside the release zip:

- `DATABASE_URL`
- `DATABASE_URL_PG`
- `BETTER_AUTH_SECRET`
- `JWT_SECRET`
- API keys and certificates

Example build and package flow when the runtime output is `.output`:

```powershell
corepack enable
pnpm install --frozen-lockfile
pnpm run build
Compress-Archive -Path .\.output -DestinationPath .\my-node-app-20260416-153000-a1b2c3d.zip -Force
```

If your runtime entrypoint is not `.output\server\index.mjs`, adapt the examples in this guide to your actual entry file.

## First-Time Server Setup

### Step 1: Create The Base Folders

Create the directory layout before installing the service.

You can do it manually, but a script is easier and repeatable. A full script is provided later in this guide as `Initialize-NodeAppLayout.ps1`.

The minimum directories are:

```text
C:\apps\my-node-app
C:\apps\my-node-app\incoming
C:\apps\my-node-app\releases
C:\apps\my-node-app\scripts
C:\services\my-node-app
C:\services\my-node-app\logs
```

### Step 2: Install Node.js LTS

Install Node.js LTS on the server and verify the runtime path.

Copy-ready verification command:

```powershell
& "C:\Program Files\nodejs\node.exe" --version
```

### Step 3: Create A Dedicated Service Identity

Do not run the app as an administrator account.

Preferred options:

- gMSA in domain environments.
- A dedicated low-privilege service user for standalone servers.

That account should have:

- Read access to the release folders.
- Read access to service config or secret sources.
- Write access only where needed, usually app temp or log paths.
- No interactive admin rights.

If you use WinSW, you can define the service account in the XML config itself. A complete example appears in the WinSW section.

## PowerShell Smoke Test

PowerShell is useful for the first local smoke test only. Do not leave the app running this way in production.

For the smoke test, point at a specific extracted release folder, not `current`.

What you want to confirm:

- The process starts.
- It listens on `127.0.0.1:3000`.
- It can reach the database.
- A local request succeeds.

Example smoke test:

```powershell
$env:NODE_ENV = "production"
$env:HOST = "127.0.0.1"
$env:PORT = "3000"
$env:DATABASE_URL = "postgresql://user:password@db-host:5432/appdb"
$env:PUBLIC_APP_URL = "https://app.example.com"

& "C:\Program Files\nodejs\node.exe" "C:\apps\my-node-app\releases\20260416-153000-a1b2c3d\.output\server\index.mjs"
```

If the app starts successfully, test locally from the server:

```powershell
Invoke-WebRequest -Uri "http://127.0.0.1:3000/" -UseBasicParsing
```

If you already have a health route, use that instead:

```powershell
Invoke-WebRequest -Uri "http://127.0.0.1:3000/healthz" -UseBasicParsing
```

### Example: TanStack Start `healthz` Endpoint

If the app uses TanStack Start with React, a minimal `healthz` endpoint can be implemented as a file route with a server-side `GET` handler.

Recommended file path:

```text
src/routes/healthz.tsx
```

For basic deployment and load balancer checks, keep this endpoint simple.

Good behavior for `healthz`:

- Returns `200` quickly.
- Does not render UI.
- Does not depend on browser-only code.
- Avoids expensive dependency checks.

If you want a deeper readiness check later, add a separate endpoint such as `/readyz` for database or cache validation.

Copy-ready TanStack Start example:

```tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/healthz")({
  server: {
    handlers: {
      GET: async () => {
        return new Response("ok", {
          status: 200,
          headers: {
            "content-type": "text/plain; charset=utf-8",
            "cache-control": "no-store",
          },
        });
      },
    },
  },
});
```

## WinSW Service Setup

WinSW is a good default choice for running a Node app as a Windows service because it is explicit, stable, scriptable, and has first-class logging and restart behavior.

Use WinSW in bundled mode:

- Rename the WinSW executable to your app name.
- Put the XML file next to it.
- Install the service from that directory.

Example files:

```text
C:\services\my-node-app\my-node-app.exe
C:\services\my-node-app\my-node-app.xml
```

### What The Service Should Run

The runtime should always point at `current`.

Example target:

```text
C:\Program Files\nodejs\node.exe C:\apps\my-node-app\current\.output\server\index.mjs
```

### WinSW XML Notes

Key elements in the XML below:

- `<arguments>` points at the active release through `current`.
- `<workingdirectory>` also points at `current`.
- `<logpath>` keeps service logs outside the release tree.
- `<onfailure>` and `<resetfailure>` define restart behavior.
- `<serviceaccount>` keeps the service off `LocalSystem`.
- Environment variables hold runtime configuration.

Copy-ready WinSW XML example:

```xml
<service>
  <id>my-node-app</id>
  <name>My Node App</name>
  <description>Node.js application service</description>

  <executable>C:\Program Files\nodejs\node.exe</executable>
  <arguments>C:\apps\my-node-app\current\.output\server\index.mjs</arguments>
  <workingdirectory>C:\apps\my-node-app\current</workingdirectory>

  <startmode>Automatic</startmode>
  <stoptimeout>15 sec</stoptimeout>
  <hidewindow>true</hidewindow>

  <logpath>C:\services\my-node-app\logs</logpath>
  <log mode="roll" />

  <onfailure action="restart" delay="10 sec" />
  <resetfailure>1 hour</resetfailure>

  <serviceaccount>
    <username>.\svc_my_node_app</username>
    <password>REPLACE_WITH_REAL_PASSWORD</password>
    <allowservicelogon>true</allowservicelogon>
  </serviceaccount>

  <env name="NODE_ENV" value="production" />
  <env name="HOST" value="127.0.0.1" />
  <env name="PORT" value="3000" />
  <env name="DATABASE_URL" value="postgresql://user:password@db-host:5432/appdb" />
  <env name="PUBLIC_APP_URL" value="https://app.example.com" />
</service>
```

If you use a gMSA instead of a password-based user, the service account block becomes:

```xml
<serviceaccount>
  <username>DOMAIN\gmsa-my-node-app$</username>
  <allowservicelogon>true</allowservicelogon>
</serviceaccount>
```

### Install The Service

After placing the WinSW executable and XML side by side, install and start the service from an elevated PowerShell session.

Copy-ready commands:

```powershell
Set-Location "C:\services\my-node-app"
.\my-node-app.exe install
.\my-node-app.exe start
.\my-node-app.exe status
```

If you later change the XML, refresh the service configuration:

```powershell
Set-Location "C:\services\my-node-app"
.\my-node-app.exe refresh
.\my-node-app.exe restart
```

## IIS Reverse Proxy And Windows Authentication

IIS should be the public edge.

Recommended IIS settings for the site:

- Bind the public hostname and TLS certificate.
- Enable Windows Authentication.
- Disable Anonymous Authentication if the whole site requires Windows sign-in.
- Enable ARR proxying at the server level.
- Enable preserve host header.
- Enable reverse rewrite host in response headers.

### Trusted Header Rule

Do not try to trust every forwarded header.

Instead:

- Keep the Node app bound to `127.0.0.1` only.
- Have IIS overwrite exactly one trusted identity header, for example `X-Remote-User`.
- Optionally inject a second shared-secret header.
- In the app, trust only those exact values.

### Allowed Server Variables

For distributed rewrite rules, IIS must allow custom server variables first.

You will typically need these names:

```text
HTTP_X_REMOTE_USER
HTTP_X_FORWARDED_PROTO
HTTP_X_FORWARDED_HOST
HTTP_X_PROXY_SHARED_SECRET
```

Important note:

- On many servers, `allowedServerVariables` is effectively a server-level setting.
- If IIS rejects the site-level configuration, add these values once at the server level in IIS Manager or in `applicationHost.config`, then keep only the rewrite rule in the site's `web.config`.

### web.config Example

This example forwards all traffic to the local Node app and injects the trusted headers.

If you do not want the optional shared secret header, remove the `HTTP_X_PROXY_SHARED_SECRET` lines from both `allowedServerVariables` and `serverVariables`.

Copy-ready `web.config` example:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <allowedServerVariables>
        <add name="HTTP_X_REMOTE_USER" />
        <add name="HTTP_X_FORWARDED_PROTO" />
        <add name="HTTP_X_FORWARDED_HOST" />
        <add name="HTTP_X_PROXY_SHARED_SECRET" />
      </allowedServerVariables>

      <rules>
        <rule name="ReverseProxyToNode" stopProcessing="true">
          <match url="(.*)" />

          <serverVariables>
            <set name="HTTP_X_REMOTE_USER" value="{LOGON_USER}" />
            <set name="HTTP_X_FORWARDED_PROTO" value="https" />
            <set name="HTTP_X_FORWARDED_HOST" value="{HTTP_HOST}" />
            <set name="HTTP_X_PROXY_SHARED_SECRET" value="replace-with-long-random-secret" />
          </serverVariables>

          <action type="Rewrite" url="http://127.0.0.1:3000/{R:1}" appendQueryString="true" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

## Security Hardening

These controls matter more than trying to sanitize every possible client-supplied header.

### 1. Bind The App To Loopback Only

Set:

- `HOST=127.0.0.1`
- `PORT=3000`

That prevents remote clients from bypassing IIS and talking to the Node app directly.

### 2. Do Not Expose The Backend Port

Do not create inbound firewall rules for port `3000`.

### 3. Trust One Header Only

In the app, trust only the header IIS overwrites, for example:

- `X-Remote-User`

Optionally also validate:

- `X-Proxy-Shared-Secret`

Ignore other user identity headers such as:

- `X-Forwarded-User`
- `X-Original-User`
- Arbitrary client-supplied `X-*` headers

### 4. Keep Secrets Out Of The Artifact

Put secrets in one of these places:

- WinSW service environment configuration
- A protected machine-level secret store
- A secure external secret source

### 5. Use A Low-Privilege Service Account

The service account should only have the access it actually needs.

### 6. Log In Two Places

Keep both:

- IIS logs
- WinSW or Node service logs

## PowerShell Scripts

The scripts below implement the folder structure and release-switch model described above.

These examples assume:

- App root: `C:\apps\my-node-app`
- Service name: `my-node-app`
- Entry point: `.output\server\index.mjs`
- Health endpoint: `http://127.0.0.1:3000/healthz`

Adapt those values if your app uses a different runtime layout.

### Initialize-NodeAppLayout.ps1

Use this once to create the base directory structure.

Copy-ready script:

```powershell
param(
  [string]$AppRoot = "C:\apps\my-node-app",
  [string]$ServiceRoot = "C:\services\my-node-app"
)

$ErrorActionPreference = "Stop"

$paths = @(
  $AppRoot,
  (Join-Path $AppRoot "incoming"),
  (Join-Path $AppRoot "releases"),
  (Join-Path $AppRoot "scripts"),
  $ServiceRoot,
  (Join-Path $ServiceRoot "logs")
)

foreach ($path in $paths) {
  if (-not (Test-Path $path)) {
    New-Item -ItemType Directory -Path $path -Force | Out-Null
  }
}

Write-Host "Created or verified app layout under $AppRoot"
Write-Host "Created or verified service layout under $ServiceRoot"
```

### Deploy-NodeAppArtifact.ps1

Use this to extract a zip into a new versioned release folder.

This script does not switch the active release. It only prepares the new release.

Copy-ready script:

```powershell
param(
  [Parameter(Mandatory = $true)]
  [string]$ArtifactZip,

  [Parameter(Mandatory = $true)]
  [string]$ReleaseName,

  [string]$AppRoot = "C:\apps\my-node-app",
  [string]$EntryPointRelativePath = ".output\server\index.mjs",
  [switch]$Force
)

$ErrorActionPreference = "Stop"

$artifactPath = (Resolve-Path $ArtifactZip).Path
$releasesRoot = Join-Path $AppRoot "releases"
$releasePath = Join-Path $releasesRoot $ReleaseName

if (-not (Test-Path $releasesRoot)) {
  throw "Releases folder does not exist: $releasesRoot"
}

if (Test-Path $releasePath) {
  if (-not $Force) {
    throw "Release already exists: $releasePath"
  }

  Remove-Item $releasePath -Recurse -Force
}

New-Item -ItemType Directory -Path $releasePath -Force | Out-Null
Expand-Archive -LiteralPath $artifactPath -DestinationPath $releasePath -Force

$entryPoint = Join-Path $releasePath $EntryPointRelativePath

if (-not (Test-Path $entryPoint)) {
  throw "Entry point not found after extraction: $entryPoint"
}

Write-Host "Prepared release at $releasePath"
Write-Host "Verified entry point $entryPoint"
```

### Switch-NodeAppRelease.ps1

This is the core cutover script.

It does the following:

- Validates the new release.
- Stops the service.
- Moves `current` to `previous`.
- Points `current` at the new release.
- Starts the service.
- Runs a local health check.
- Rolls back automatically if startup or health fails.

On the very first deployment there may be no `previous` release yet, so rollback can only restore a prior version if one already existed.

Copy-ready script:

```powershell
param(
  [Parameter(Mandatory = $true)]
  [string]$NewReleasePath,

  [string]$AppRoot = "C:\apps\my-node-app",
  [string]$ServiceName = "my-node-app",
  [string]$EntryPointRelativePath = ".output\server\index.mjs",
  [string]$HealthUrl = "http://127.0.0.1:3000/healthz",
  [int]$HealthTimeoutSec = 10,
  [int]$HealthAttempts = 10,
  [int]$HealthDelaySec = 3
)

$ErrorActionPreference = "Stop"

function Wait-ForServiceStatus {
  param(
    [Parameter(Mandatory = $true)]
    [string]$Name,

    [Parameter(Mandatory = $true)]
    [string]$Status,

    [int]$TimeoutSec = 30
  )

  (Get-Service -Name $Name -ErrorAction Stop).WaitForStatus($Status, [TimeSpan]::FromSeconds($TimeoutSec))
}

function Test-Health {
  param(
    [Parameter(Mandatory = $true)]
    [string]$Url,

    [int]$TimeoutSec = 10,
    [int]$Attempts = 10,
    [int]$DelaySec = 3
  )

  for ($i = 1; $i -le $Attempts; $i++) {
    try {
      $response = Invoke-WebRequest -Uri $Url -TimeoutSec $TimeoutSec -UseBasicParsing

      if ($response.StatusCode -ge 200 -and $response.StatusCode -lt 400) {
        return $true
      }
    }
    catch {
    }

    Start-Sleep -Seconds $DelaySec
  }

  return $false
}

$newReleasePath = (Resolve-Path $NewReleasePath).Path
$entryPoint = Join-Path $newReleasePath $EntryPointRelativePath
$currentLink = Join-Path $AppRoot "current"
$previousLink = Join-Path $AppRoot "previous"

if (-not (Test-Path $newReleasePath)) {
  throw "Release path does not exist: $newReleasePath"
}

if (-not (Test-Path $entryPoint)) {
  throw "Release is missing entry point: $entryPoint"
}

$service = Get-Service -Name $ServiceName -ErrorAction Stop

try {
  if ($service.Status -ne "Stopped") {
    Stop-Service -Name $ServiceName -ErrorAction Stop
    Wait-ForServiceStatus -Name $ServiceName -Status "Stopped"
  }

  if (Test-Path $previousLink) {
    Remove-Item $previousLink -Force
  }

  if (Test-Path $currentLink) {
    Rename-Item -Path $currentLink -NewName "previous"
  }

  New-Item -ItemType Junction -Path $currentLink -Target $newReleasePath | Out-Null

  Start-Service -Name $ServiceName -ErrorAction Stop
  Wait-ForServiceStatus -Name $ServiceName -Status "Running"

  if (-not (Test-Health -Url $HealthUrl -TimeoutSec $HealthTimeoutSec -Attempts $HealthAttempts -DelaySec $HealthDelaySec)) {
    throw "Health check failed: $HealthUrl"
  }

  Write-Host "Deployment successful. Active release: $newReleasePath"
}
catch {
  Write-Warning "Deployment failed: $($_.Exception.Message)"
  Write-Warning "Attempting rollback..."

  try {
    $service = Get-Service -Name $ServiceName -ErrorAction Stop

    if ($service.Status -ne "Stopped") {
      Stop-Service -Name $ServiceName -ErrorAction SilentlyContinue
      Wait-ForServiceStatus -Name $ServiceName -Status "Stopped"
    }
  }
  catch {
  }

  if (Test-Path $currentLink) {
    Remove-Item $currentLink -Force
  }

  if (Test-Path $previousLink) {
    Rename-Item -Path $previousLink -NewName "current"

    Start-Service -Name $ServiceName -ErrorAction SilentlyContinue
    Wait-ForServiceStatus -Name $ServiceName -Status "Running"
  }

  throw
}
```

### Rollback-NodeAppRelease.ps1

Use this when you want an explicit manual rollback command.

Copy-ready script:

```powershell
param(
  [string]$AppRoot = "C:\apps\my-node-app",
  [string]$ServiceName = "my-node-app"
)

$ErrorActionPreference = "Stop"

$currentLink = Join-Path $AppRoot "current"
$previousLink = Join-Path $AppRoot "previous"

if (-not (Test-Path $previousLink)) {
  throw "No previous release is available."
}

$service = Get-Service -Name $ServiceName -ErrorAction Stop

if ($service.Status -ne "Stopped") {
  Stop-Service -Name $ServiceName -ErrorAction Stop
  (Get-Service -Name $ServiceName).WaitForStatus("Stopped", [TimeSpan]::FromSeconds(30))
}

if (Test-Path $currentLink) {
  Remove-Item $currentLink -Force
}

Rename-Item -Path $previousLink -NewName "current"

Start-Service -Name $ServiceName -ErrorAction Stop
(Get-Service -Name $ServiceName).WaitForStatus("Running", [TimeSpan]::FromSeconds(30))

Write-Host "Rollback complete."
```

### Prune-NodeAppReleases.ps1

Use this separately from deployment. Do not delete old releases during cutover.

This script keeps `current`, `previous`, and the newest extra releases you choose.

Copy-ready script:

```powershell
param(
  [string]$AppRoot = "C:\apps\my-node-app",
  [int]$KeepExtraReleases = 3
)

$ErrorActionPreference = "Stop"

$releasesRoot = Join-Path $AppRoot "releases"
$currentLink = Join-Path $AppRoot "current"
$previousLink = Join-Path $AppRoot "previous"

$protectedTargets = @()

if (Test-Path $currentLink) {
  $protectedTargets += (Get-Item $currentLink).Target
}

if (Test-Path $previousLink) {
  $protectedTargets += (Get-Item $previousLink).Target
}

$protectedTargets = $protectedTargets | Where-Object { $_ } | ForEach-Object { [System.IO.Path]::GetFullPath($_) }

$releaseDirs = Get-ChildItem -Path $releasesRoot -Directory | Sort-Object Name -Descending

$removable = $releaseDirs | Where-Object {
  [System.IO.Path]::GetFullPath($_.FullName) -notin $protectedTargets
}

$toDelete = $removable | Select-Object -Skip $KeepExtraReleases

foreach ($dir in $toDelete) {
  Remove-Item $dir.FullName -Recurse -Force
  Write-Host "Removed old release: $($dir.FullName)"
}
```

## Release, Update, And Rollback Flow

Use this exact operational flow for a normal release.

1. Build the artifact in CI or on a build machine.
2. Copy the artifact zip to `C:\apps\my-node-app\incoming`.
3. Extract the artifact into `releases\<release-name>`.
4. Run database migrations.
5. Run the cutover script.
6. Verify local health.
7. Verify the IIS public URL.
8. Verify authentication behavior.
9. Keep the previous release available for rollback.

Example commands:

```powershell
Set-Location "C:\apps\my-node-app\scripts"

.\Deploy-NodeAppArtifact.ps1 `
  -ArtifactZip "C:\apps\my-node-app\incoming\my-node-app-20260416-153000-a1b2c3d.zip" `
  -ReleaseName "20260416-153000-a1b2c3d"

.\Switch-NodeAppRelease.ps1 `
  -NewReleasePath "C:\apps\my-node-app\releases\20260416-153000-a1b2c3d" `
  -HealthUrl "http://127.0.0.1:3000/healthz"
```

Explicit rollback command:

```powershell
Set-Location "C:\apps\my-node-app\scripts"
.\Rollback-NodeAppRelease.ps1
```

## Database Migration Strategy

The database step must be part of the release process, not an afterthought.

Recommended rules:

- Make migrations an explicit deployment step.
- Prefer backward-compatible migrations where possible.
- Apply migrations before switching traffic to the new app version.
- Avoid releases where version `N` works only with schema `N` and version `N-1` immediately breaks.

The long-term goal should be:

```text
app version N and app version N-1 both work against the migrated schema whenever practical
```

If your app requires source code and package manager tooling to run migrations, treat migrations as a separate deployment concern from the minimal runtime artifact.

## App-Side SSO Bridge

IIS can authenticate the Windows user, but your app still needs to decide what to do with that identity.

Common pattern:

1. IIS authenticates the user.
2. IIS injects `X-Remote-User`.
3. The app reads that header.
4. The app maps the Windows identity to a local user record.
5. The app creates its normal session or cookie.
6. The rest of the app continues to use its existing auth model.

This is usually cleaner than replacing the app's whole auth stack.

Good mapping choices:

- `DOMAIN\username`
- UPN such as `user@example.com`
- An internal directory lookup if you already have one

Bad idea:

- Trusting arbitrary client-provided identity headers without IIS as the boundary

## Operations Checklist

After first deployment and after every release, verify these items:

1. The Windows service is running.
2. `http://127.0.0.1:3000/healthz` works locally on the server.
3. The public IIS URL works over HTTPS.
4. Windows Authentication behaves as expected.
5. The app receives the correct authenticated user identity.
6. Direct external access to port `3000` is not possible.
7. WinSW logs show clean startup.
8. IIS logs show the expected request flow.

Useful commands:

```powershell
Get-Service my-node-app
Invoke-WebRequest -Uri "http://127.0.0.1:3000/healthz" -UseBasicParsing
```

## Troubleshooting

### Service Starts And Stops Immediately

Common causes:

- Wrong Node executable path.
- Wrong entrypoint path.
- Missing environment variable.
- Service account lacks folder permissions.
- App crashes before listening.

Check:

- WinSW logs in `C:\services\my-node-app\logs`
- Windows Event Viewer
- Manual smoke test from PowerShell

### IIS Returns 502 Or 503

Common causes:

- The service is down.
- The app is not listening on `127.0.0.1:3000`.
- ARR proxy is not enabled.
- Rewrite target is wrong.

Check locally first:

```powershell
Invoke-WebRequest -Uri "http://127.0.0.1:3000/healthz" -UseBasicParsing
```

### Public Site Works But Authenticated User Is Missing

Common causes:

- Windows Authentication is disabled.
- Anonymous Authentication is still enabled when it should not be.
- `HTTP_X_REMOTE_USER` is not in `allowedServerVariables`.
- The app trusts the wrong header name.

### Direct Access To Port 3000 Works From Another Machine

That means the backend is exposed more than intended.

Check:

- `HOST` really is `127.0.0.1`
- The app is not separately bound to `0.0.0.0`
- No firewall rule exposes `3000`

## Final Recommendations

If you want the simplest reliable production model, use this:

1. Install a fixed Node.js LTS runtime on the server.
2. Run the app as a WinSW-managed Windows service.
3. Bind the app to `127.0.0.1:3000` only.
4. Put IIS in front for HTTPS and Windows Authentication.
5. Trust exactly one identity header injected by IIS.
6. Keep releases immutable in versioned folders.
7. Deploy by switching `current`.
8. Roll back by switching back to `previous`.

That gives you repeatability, fast rollback, no interactive shell dependency, and a clear separation between IIS, the Node service, and deployment automation.
