```markdown
# Agent Task: Add "Advanced locations" (per-server selection) + Release EXE generation

Goal
- Implement an "Advanced locations" view in your fork of ProtonVPN/win-app that displays the full server list (per-server selection) and allows connecting to a chosen server — matching the GTK app capability.
- Produce a release EXE artifact and attach it to a GitHub Release (local steps + automated GitHub Actions workflow).

Success criteria (automatable)
- Agent creates a branch on the fork named `feature/advanced-locations`.
- Agent adds/edits files in the fork as described and pushes the branch.
- Build completes (dotnet build) and unit tests run (dotnet test) without new failures.
- New UI page `AdvancedLocationsPage` is wired to open from the main selector (hidden behind an "Advanced" button).
- Servers are fetched from the same public API endpoint the GTK client uses (placeholders replaced by actual endpoint).
- Selecting "Connect" on a server triggers the existing connection manager with the server's hostname/ID.
- The change is gated behind a feature flag (default OFF).
- A release EXE is produced by `dotnet publish` (self-contained single-file) and attached to a GitHub Release (workflow creates release on tag push or via gh CLI).
- A PR is opened on the fork with description, tests and screenshots, and the PR checklist below is completed.

Prerequisites (agent inputs)
- Fork repo URL (the agent operates on your fork): git@github.com:dxrkinfuser44/win-app.git
- Personal access token or SSH key for pushing to your fork (agent must have push rights configured).
- The agent must be able to run dotnet CLI commands (build/test) or at least run static checks.
- GTK repo location (for extracting endpoint and JSON schema): https://github.com/ProtonVPN/proton-vpn-gtk-app
- For signing (optional): code signing certificate as PFX uploaded to repo secrets as base64 `SIGNING_PFX` and the `SIGNING_PASSWORD` secret.
- For installer (optional): if using Inno Setup, include your .iss script in repo at `installer/ProtonVpnInstaller.iss`.

Top-level plan (actionable steps)
1. Prepare workspace
   - git clone the fork, create branch `feature/advanced-locations`.
   - Open solution in Rider/Visual Studio / or use CLI.

2. Discover/confirm API
   - Inspect GTK app to find the exact endpoint and response JSON for servers (look for `servers` or `locations` API usage).
   - Copy the JSON schema and sample response. Note exact field names.

3. Add models / DTOs
   - Create `src/Client/Models/ServerModel.cs` that maps exactly to the JSON fields from GTK.

4. Implement Servers API client
   - Create `src/Client/Services/ServersClient.cs` implementing `IServersClient`.
   - Use existing `HttpClient` factory and the app's `IAuthService`.

5. Add caching and TTL
   - Implement `ServerCache` persisted in AppData with TTL (default 10 minutes).

6. Add ViewModel & UI
   - Add `ServersViewModel` and `AdvancedLocationsPage` (XAML + code-behind).
   - Add "Advanced" button to main selector; gate behind feature flag.

7. Wire DI & connection integration
   - Register services in DI and extend `IConnectionManager` to accept hostname/ID.

8. Tests
   - Unit tests for ServersClient and ServersViewModel; UI tests manual or automated.

9. Release EXE — local steps
   - Build a self-contained single-file Windows EXE (x64) locally:
     - dotnet publish -c Release -r win-x64 -p:PublishSingleFile=true -p:PublishTrimmed=true -p:PublishReadyToRun=true --self-contained true -o artifacts/win-x64
   - Remark: `PublishTrimmed` sometimes removes required reflection-usage types; test thoroughly.
   - Optional signing locally (requires signtool + code signing cert):
     - Sign with signtool:
       - signtool sign /fd SHA256 /a /tr http://timestamp.digicert.com /td SHA256 /f "path\to\cert.pfx" /p "<pfx-password>" "artifacts\win-x64\MyApp.exe"
   - Create a ZIP for distribution:
     - powershell Compress-Archive -Path artifacts/win-x64\* -DestinationPath artifacts/ProtonVpn-windows-x64-v1.2.3.zip

10. Release EXE — CI workflow (recommended)
    - Add `.github/workflows/release.yml` (example provided) that:
      - Runs on push of a tag `v*`.
      - Builds dotnet publish on windows-latest.
      - Optionally decodes PFX from repo secret and signs executable using signtool.
      - Zips the artifacts and uploads them to a GitHub Release created for the tag.

11. Installer (optional)
    - Use Inno Setup or WiX:
      - Inno Setup: add `installer/ProtonVpnInstaller.iss` and call ISCC on windows-latest runner.
      - Workflow step runs Inno Setup compiler and uploads produced installer .exe to the Release.

12. Open PR & release
    - Push branch and open PR.
    - Create a Git tag (vX.Y.Z) to trigger the release workflow OR create a release via gh CLI and upload artifact.

Files to add / modify
- src/Client/Models/ServerModel.cs (new)
- src/Client/Services/IServersClient.cs (new)
- src/Client/Services/ServersClient.cs (new)
- src/Client/Services/ServerCache.cs (new)
- src/Client/ViewModels/ServersViewModel.cs (new)
- src/Client/Views/AdvancedLocationsPage.xaml (new)
- src/Client/Views/AdvancedLocationsPage.xaml.cs (new)
- src/Client/Resources/Strings.resx (add new keys)
- src/Client/Connection/IConnectionManager.cs (extend or overload)
- tests/Client.Tests/ServersClientTests.cs (new)
- tests/Client.Tests/ServersViewModelTests.cs (new)
- AGENT_TASKS/advanced-locations-task.md (this file)
- AGENT_TASKS/advanced-locations-task.json (updated manifest)
- .github/workflows/release.yml (new CI workflow for building/publishing release EXE and attaching artifacts)
- installer/ProtonVpnInstaller.iss (optional Inno Setup script)

CLI commands (local)
- Clone & branch
  - git clone git@github.com:dxrkinfuser44/win-app.git
  - cd win-app
  - git checkout -b feature/advanced-locations
- Build & test
  - dotnet build
  - dotnet test
- Publish release EXE locally
  - dotnet publish -c Release -r win-x64 -p:PublishSingleFile=true -p:PublishTrimmed=true -p:PublishReadyToRun=true --self-contained true -o artifacts/win-x64
- Sign (optional, local)
  - signtool sign /fd SHA256 /a /tr http://timestamp.digicert.com /td SHA256 /f "c:\path\to\cert.pfx" /p "PFX_PASSWORD" "artifacts\win-x64\ProtonVpn.exe"
- Zip
  - powershell Compress-Archive -Path artifacts/win-x64\* -DestinationPath artifacts/ProtonVpn-windows-x64-v1.2.3.zip
- Upload release artifact using gh (local)
  - gh release create v1.2.3 artifacts/ProtonVpn-windows-x64-v1.2.3.zip --title "v1.2.3" --notes "Release notes..."
  - or use the GitHub web UI.

CI notes (secrets required)
- SIGNING_PFX (base64-encoded PFX)
- SIGNING_PASSWORD
- GITHUB_TOKEN (already provided to workflows; GH release action uses this)

Testing checklist (for PR)
- [ ] Build passes on Windows & CI (dotnet build)
- [ ] Unit tests for ServersClient and ServersViewModel added and passing
- [ ] Manual test: Advanced page shows cached servers immediately and refreshes
- [ ] Manual test: Connect button triggers connection manager and completes a connection
- [ ] Release EXE produced locally and in CI, artifact attached to GitHub Release
- [ ] Feature gated default OFF
- [ ] Strings localized
- [ ] No secrets logged; tokens not persisted in plaintext
- [ ] Accessibility and keyboard navigation checked
- [ ] Localization keys extracted

Estimated effort
- Prototype (fetch + show list, no connect): 1–2 dev days.
- Integration with connection manager + release pipeline: 2–4 days depending on signing and installer choices.
- CI & release: 0.5–1 day to configure and secure signing secrets.
- QA & staged release: 1–2 days.

Acceptance criteria for hand-off
- Branch pushed, PR opened with checklist.
- Release workflow file included and configured (requires secrets added before sign step will run).
- Tag created to trigger workflow or manual release via gh.

Notes
- Publishing as self-contained single-file executable increases artifact size but simplifies distribution.
- PublishTrimmed may break reflection-heavy code; test thoroughly.
- Signing requires managing private signing keys responsibly — store PFX as a secret and avoid exposing passphrases.
```