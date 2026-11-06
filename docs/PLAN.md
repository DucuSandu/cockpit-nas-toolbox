# NAS Toolbox: plan

Nov 6, 2026 · Ducu

> Copy of the live plan doc "NAS Toolbox: plan". Planning only: no code is written until Ducu says "Start Code".

## Verdict

Yes, it is feasible. **NAS Toolbox** is a **general, shareable Cockpit plugin for home and small-office NAS boxes**: install it on any Debian or Ubuntu server, VM or Proxmox container, and manage people, groups, shared folders and optional apps (Immich, Nextcloud, FTP) from one friendly page. Nothing in it is tied to one house. Ducu's own vera-nas setup is the **reference install and the first preset** ("Family"), the proof that it covers a real need.

## Who it is for and how it should feel

**For** people who run a NAS at home for their family (or a small office) and want Samba, photos and a cloud without learning smb.conf, occ or Docker. **Not for** enterprises, Active Directory or large clusters.

**Friendly by design**

- **First-run wizard** in 4 steps: pick a preset (**Family**, **Small office**, **Just me**), add people, pick the folders to share, tick the apps you want. Everything after that is the normal page.
- **Plain words**: "Shared folders", "Who can see / change", "Kids can only look", not `valid users` or `write list`. The real setting is one click away under "Show details" for those who want it.
- **Safe defaults**: everything off that could hurt (guest access, plain FTP, old SMB1), LAN-only, recycle bin on.
- **Nothing is a surprise**: every change shows a preview in plain sentences ("Vlad will be able to read Memories") before it is applied, and the last 20 versions can be restored with one click.
- **It explains itself**: a short help line under every section, warnings in words ("Windows permissions are not available in this container, because..."), and a health page that says what to fix and how.
- **Works on small screens**: usable from a phone, light and dark theme following Cockpit.
- **Languages**: English, and fully translatable: all text lives in translation files so anyone can add a language.

**Works anywhere, adapts to what it finds**

- Debian 12 and 13, Ubuntu 24.04; bare metal, VM, privileged or unprivileged LXC. It detects the environment and greys out what cannot work there (for example Windows ACLs in an unprivileged container), with the reason.
- No hard-coded names, paths, users or album names: all of them come from the wizard and the settings.
- Adopts an existing setup: it reads the current smb.conf, users and apps, shows what it found, and changes nothing until you confirm.

**How it is shared**

- Open source on GitHub under **GPL-3.0**, with a `.deb` package and a one-line installer that previews first.
- Docs site with screenshots, a "Family NAS in 30 minutes" guide based on Ducu's setup, and a plugin guide for people who want to add their own app.

## Scope

NAS Toolbox manages people, groups, what they can reach, and the health of the file services. It does not replace Proxmox (disks, ZFS, backups and the container itself stay on the host).

**In scope**

- **People**: add, disable, enable, reset password, remove, group membership. One form creates the Linux account, the Samba password and the accounts in the enabled plugins, with one password for all of them. Bulk create and import from a list.
- **Groups**: three start groups (Admins, Supervisors, Users) plus your own; every right comes from group checkboxes.
- **Samba**: full share and global settings editing on plain smb.conf (same options as the 45Drives plugin), who can read and write each share, Samba user and password.
- **Plugins**: Immich, Nextcloud and FTP, each optional, each with Connect existing / Install new.
- **API keys**: Immich keys used by the scripts and Nextcloud app passwords, listed, tested, replaced and revoked from one place.
- **Photo scripts** (part of the Immich plugin): the curation scripts run from the page, with their settings (album names, paths, limits, timer) editable in a form instead of inside the code.
- **File browser**: browse, upload, download, rename, set owner and permissions inside the share root.
- **General (overview)**: services, disk usage per share, who is connected right now (smbstatus).

**Out of scope**

- Active Directory / domain controller. Samba stays a standalone file server.
- **NFS**: the kernel NFS server cannot run in an unprivileged LXC, and the user-space server (NFS-Ganesha) needs file-handle access that unprivileged containers also block. If NFS is ever needed, export it from the Proxmox host on the same dataset. The installer removes the broken nfs-kernel-server package.
- Exposing Cockpit to the internet. It stays on the LAN or behind a VPN.
- Disk, RAID, ZFS and snapshot management (Proxmox host).

## Build vs reuse

We build our own Samba management on a plain `/etc/samba/smb.conf` (no registry), with the same options the 45Drives plugin offers, so everything about a person and their shares lives in one page. The 45Drives plugin is dropped. Only the file browser is reused.

| Need | Existing option | Plan |
| --- | --- | --- |
| Samba shares and global settings | 45Drives cockpit-file-sharing | Build our own on plain smb.conf, copying its option set. The installer moves the shares to smb.conf, then removes cockpit-file-sharing. |
| Linux users, groups, Samba user and password | 45Drives cockpit-identities; Cockpit's own Accounts page | Build our own, because neither touches the plugins. Cockpit Accounts stays for system admins. |
| File browser | Cockpit Files (official, newer) or 45Drives cockpit-navigator | Reuse. Prefer Cockpit Files if the Debian 13 package is available, else Navigator. |
| Nextcloud users, groups, quotas, external storage | None for Cockpit | Build. |
| NAS overview (services, sessions, disk per share) | None | Build. |

Moving from the registry to plain smb.conf is a one-time step in the installer: export the registry shares (`net conf list`), write them as normal sections in smb.conf, remove `include = registry`, run `testparm`, reload smbd, and only then empty the registry (`net conf drop`). The old smb.conf and the export are kept as backups, and the preview shows the new smb.conf before anything changes.

## Groups and rights

**Everything is a group; people have no checkboxes of their own.** A person is just a name, a password and a list of groups. All rights come from the groups they are in.

**Start config**: a fresh install creates three groups. They can be renamed, and more groups added.

- **Admins**: like root, full power. Members open NAS Toolbox, change the whole setup (settings, Samba, groups, plugins, people) and reach all files, including everyone's home folders. The plugin warns before the last admin is removed.
- **Supervisors**: members may use every feature of the NAS (all shares and apps) but cannot change the configuration. They get a small People page to manage people in the Users group and passwords, for themselves and for those users, never for admins. Settings, Samba, plugins and groups stay admin-only. In the Family example, Supervisors = parents.
- **Users**: everyone is in it; it holds the defaults.

**Rules**

- **Several groups**: the more generous group wins (write beats read-only). A person's row shows where each right comes from ("read-only, from kids").
- **One exception needed?** Give that person a group of their own. This keeps every right visible in one place: the Groups page.
- Changing a group's checkboxes updates every member in every system at once (Samba lists, Linux groups, Nextcloud and Immich accounts), shown in the preview first.

**Group capability checkboxes** (each one switches a capability on or off for the group's members):

1. **Access to admin shares**: the share that lets members curate the users' files: open, sort and tidy everyone's private folders (files they add stay owned by the folder's owner, through `inherit owner = unix only`). Only Admins and root can tick this checkbox on a group.
2. **Home folder**: members get a personal folder.
3. **Read Outside Home Folder**: members can read the shared space outside their own home folder. It never opens other people's home folders; only checkbox 1 (admin shares) does that. Off = they see only their own folder.
4. **Write Outside Home Folder**: members can also change files in the shared space outside their own home folder (again never other people's homes). In the Family example: parents on, kids off.
5. **Public share links**: members can share a link with someone outside the NAS in Nextcloud or Immich.
6. **Remote access**: members can reach the NAS from outside the house (Tailscale), not only at home.
7. **Can change own password**.
8. **Expires on a date**: for guests.
9. **FTP / SFTP access** (shown only when the FTP plugin is enabled).
10. **Immich account** (shown only when the Immich plugin is enabled).
11. **Nextcloud account** (shown only when the Nextcloud plugin is enabled).

**Not checkboxes**

- **Storage quota** is set per user, on their home folder, not on the group. Nextcloud and Immich enforce it; on the home folder over Samba the plugin only shows usage and warns when someone goes over (no hard limit).
- **Restore from the recycle bin** is always allowed for everyone.

**Folder layout is configurable** in Settings, with the preset filling in sensible values:

- where home folders live and their default subfolders (editable list, created for each new person);
- where shared folders live and which ones the wizard creates;
- the recycle bin folder name and how many days it keeps files.

## The Family example

Presets fill in groups, folders and checkboxes at first run, and everything stays editable. The **Family** preset is based on Ducu's real setup and is the example below; **Small office** and **Just me** work the same way. The names below are the example's, not built into the code.

- **People and groups**: radu, andreea (parents, in Supervisors), vlad, eliza (kids, in Users). All logins have no shell.
- **Family space is curated by parents**: `shared` (memories, docs, multimedia, misc) is read for everyone and writable only by parents (`write list = @parents`). Files there are written as the service user `nas` (uid/gid 1000, `force user` / `force group = nas`, masks 0664 / 2775), so Samba, Nextcloud and Immich never fight over permissions. The plugin hides `nas` from People and never lets it be disabled, removed or given a password.
- **Home folders with real owners**: `/mnt/homes/<name>`, owned by the person, group parents, mode 2770 (no `force user` there).
- **home** share (`path = /mnt/homes/%U`): each person lands in their own folder and sees nothing else. Ten people still means one share in smb.conf.
- **admin** share (`/mnt/homes`, parents only, `inherit owner = unix only`): parents see and edit every home folder; a file a parent adds in a kid's folder stays owned by the kid.
- **Global rules kept and locked in the form**: SMB2 to SMB3 only, `map to guest = never`, no master browser, `hosts allow` = LAN and tailnet only, `vfs objects = catia fruit streams_xattr` (Mac support), no wide links. Windows ACLs (`acl_xattr`) are greyed out with the reason: the unprivileged container cannot store them.

What the checkboxes give in this example:

| Share | Parents | Kids |
| --- | --- | --- |
| shared (includes memories) | write | read |
| home (own folder) | own folder | own folder |
| admin (all home folders) | write | none |
| any share you add | you pick | you pick |

Kids get **read-only** access to memories through `shared`, so no separate memories share is needed. The page turns the checkboxes into Samba settings (`valid users`, `write list`, `read only`). Kids' changes can be made undoable with the recycle bin (`vfs recycle`, files go to `.recycle/<user>` for 30 days). ZFS snapshots stay the safety net.

**Three rules keep it simple as the family or the NAS grows:**

1. **Rights go to groups, never to single people.** A new person only needs a group. No per-person exceptions to track.
2. **One folder with its own rights = one share.** Samba sets rights per share. Keep them few and top-level (shared, home, admin); subfolders inherit their share's rights.
3. **Home folders come from one `home` share, not one share per person.** Each person's folder is created automatically when the person is added.

## Samba

Our own Samba editor on plain `/etc/samba/smb.conf`. The helper owns the file: it reads it with `testparm -s`, writes a new version from the form, checks it with `testparm` before replacing the old one, keeps the last 20 versions, and reloads with `smbcontrol all reload-config` (no disconnects).

- **Share options** (the same set as the 45Drives plugin): name, description (`comment`), path, valid users and groups, write list, read only, guest ok, browseable, inherit permissions, Windows ACLs (`vfs objects = acl_xattr`, `map acl inherit`; greyed out where the container cannot store them), shadow copies (`shadow_copy2`), macOS share (`fruit`, `streams_xattr`), audit log (`full_audit`), recycle bin (`recycle`), and a free-form "advanced" box for any other `key = value`.
- **Global options**: server description, workgroup, log level, guest mapping, minimum protocol (SMB2 or newer), and the same advanced box.
- **Access**: share × group with none / read / write; changing a cell updates `valid users` and `write list`.
- **Folder permissions**: when a share is created or its groups change, set the folder's group, the setgid bit and default ACLs (`setfacl -d`) so new files keep the right group.
- **Samba user and password**: part of People (`smbpasswd -a -s`, `-d`, `-e`, password on stdin), plus a list from `pdbedit -L -v` showing last password change and disabled flags.
- **Usage and sessions**: `du` on each share path (cached, run in the background) and `smbstatus --json` for who is connected and which files are open, with a "disconnect" button.
- **Import / export**: download smb.conf, or paste a share section to import it.

## Plugins

**One rule for every plugin (Immich, Nextcloud, FTP and any later one):**

- Always listed on the Plugins page with **Connect existing** and **Install new** buttons and an on/off checkbox.
- Nothing is put on the box until a button is used.
- Enabling opens the user matching step.
- Enabled = its own entry in NAS Toolbox's left menu and its group checkbox (Immich account, Nextcloud account, FTP / SFTP access). No separate Cockpit sidebar entry.
- Uninstall removes the app and its config but never data unless you tick and type to confirm.
- Each plugin is a page file plus a helper file, and must have four functions: `connect`, `install`, `uninstall` and `accounts` (for matching). The core refuses to load a plugin that lacks one.

**Turning a plugin off** only hides its menu entry and checkbox and stops changes; the accounts stay in Immich or Nextcloud, and the links come back when it is enabled again. Turning one on first checks the service is reachable (`occ status`, Immich `/server/ping`) and shows what it found before saving.

**When a plugin is enabled, existing people are not changed automatically.** Enabling opens a matching step instead:

1. The page lists every person next to the plugin's accounts, matched by username, then email, then name, in three groups: **matched**, **only on the NAS**, **only in the plugin**.
2. **Matched**: linked, nothing else changes. Their plugin password stays as it is until the next password reset in People, which then sets all of them together.
3. **Only on the NAS**: those whose groups have the plugin's checkbox get an account. Passwords cannot be copied (every system stores them hashed), so each new account gets either a new password you type, or a generated one shown once (Immich can also force a change at first login).
4. **Only in the plugin** (for example Immich's admin or a test account): adopt as a person, or leave it unmanaged. Unmanaged accounts are shown greyed out and never touched.
5. Nothing runs until you confirm the preview. The same matching is available any time from an "Import from Immich / Nextcloud" button in People.

### Install new (Immich and Nextcloud)

A short wizard, shown as a preview first, then run with live output:

1. **Checks**: Docker and the compose plugin present (offer to install `docker.io` if not), free disk space, the port free.
2. **Choices**: data folder, port, version (the current stable release, pinned, never "latest").
3. **Files**: a stack folder `/opt/stack/<app>/` with `docker-compose.yml` and `.env`. Immich uses its official release compose file; Nextcloud uses the official `nextcloud` image with MariaDB and Redis (plain compose, not AIO, so `occ` works through `docker exec`). Database passwords are generated and stored only in `.env`, root-only.
4. **Start and set up**: `docker compose up -d`, wait until healthy, then create the admin automatically (Immich's first-admin sign-up API; Nextcloud `occ maintenance:install`), create the API key or app password the plugin needs, and save it in Keys.
5. **Finish**: enable the plugin, add its left-menu entry, and open the user matching step.

**After install**, the plugin's page has Update (pull the next pinned version, back up the database first), Start, Stop, Restart, logs, and health. **Uninstall** stops and removes the containers but keeps the data and database folders unless you tick a separate "also delete data" box and type the app name.

**For Nextcloud**, the wizard also sets the trusted domain, the maintenance window, a background job (cron) container, and Redis for file locking.

### Immich

Immich lives inside NAS Toolbox, as its own entry in the left menu, because it is an admin page. Accounts are managed from People through the Immich admin API with the admin key:

- **Add**: `POST /admin/users` with email, name, password and the person's storage quota. Immich logs in by email, so People gets an email field.
- **Password**: the same password as Samba and Nextcloud, set with `PUT /admin/users/{id}`.
- **Disable**: Immich has no disable switch, so the page locks the account by setting a long random password; enable sets a new one.
- **Remove**: not done by default. Deleting an Immich user also deletes all their photos after the grace period (7 days by default). The page only locks the account; a real delete needs a separate, explicit confirmation.

### Nextcloud

- Users, groups and quotas through `occ` (`user:list --output=json`, `user:setting <user> files quota`, `user:lastseen`), with the same group names as Linux.
- **NAS folders inside Nextcloud**: `occ files_external:create` with the "Local" backend pointing at a share path, limited to the matching group (`files_external:applicable --add-group`). Then the same files are reachable over SMB and in Nextcloud. Needs the Nextcloud process to see the path (bind mount if it runs in Docker) and file permissions that let www-data read or write.
- **Rescan** after files change over SMB: `occ files:scan --path` for local storage, or the external storage "check for changes" setting.
- **Health**: `occ status`, `occ setupchecks` (the same warnings as the admin page), with buttons for the safe fixes such as `occ maintenance:repair --include-expensive` and setting `maintenance_window_start`.
- Shares between Nextcloud users (links, user-to-user) stay in the Nextcloud web UI.

### FTP

Handled exactly like Immich and Nextcloud. **Connect existing** uses the SSH server or a vsftpd already on the box; **Install new** adds what is missing. Uninstall removes the packages and config again; files are never touched.

- **SFTP (recommended, on by default)**: uses the SSH server, encrypted, works with FileZilla, WinSCP and phone apps. People with FTP access get `internal-sftp` only (no shell), locked (chroot) to their view.
- **FTP / FTPS (optional, for old devices)**: `vsftpd` for scanners, cameras or TVs that only speak FTP. FTPS (TLS) when the device supports it; plain FTP only on the LAN, off unless ticked.
- **Who gets it**: members of groups with **FTP / SFTP access** ticked. Same password as Samba (the plugin also sets the Linux password, which SFTP and vsftpd check).
- **What they see**: a locked view that mirrors their Samba access, from the group checkboxes: Home (own home folder, if "Home folder"), Shared (read-only or writable per "Read / Write Outside Home Folder"), Admin (everyone's home folders, only with "Access to admin shares"). FTP never gives more than SMB does.
- **Settings**: ports, passive port range for FTP, TLS certificate, allow plain FTP yes/no, an optional upload-only folder (for a scanner).
- **General page**: FTP service status and active transfers.

## People

One row per person, showing their groups and which accounts they have (Linux, Samba, and each enabled plugin), with where each right comes from.

- **Add**: `useradd --no-create-home --shell /usr/sbin/nologin` (no SSH login, file access only), then `smbpasswd -a -s` with the password on stdin, then the enabled plugins' accounts as the groups' checkboxes say. Groups added in the same step.
- **Reset password**: one password for Samba and every enabled plugin, set together.
- **Disable / enable**: `smbpasswd -d/-e`, `usermod -L/-U`, and the same in each plugin. Nothing is deleted.
- **Remove**: only after disable; removes accounts, never the person's files. Their folder is moved to an archive folder instead.
- **Rules**: only users with uid 1000 and up; root and system accounts are never shown or touched.
- **Bulk add**: paste or upload a CSV (`username, full name, email, groups, password`), empty password = generate one. The page shows a preview (new / already exists / invalid) and creates nothing until you confirm. Generated passwords are shown once and can be downloaded as a sheet to hand out.
- **Export**: the same list as CSV, without passwords, as a record of who has access to what.

## File browser

Cockpit Files or Navigator, installed by our installer and limited by Linux permissions of the logged-in admin. Our page adds "Open in Files" links next to each share and home folder.

## Links into Nextcloud and Immich

- **Simple**: "Open in Nextcloud" links per person and per folder (Files app at that path, user admin page, admin overview). If the browser already has a Nextcloud session it opens logged in; otherwise you log in once and the browser remembers it.
- **Real single sign-on** (log in to Cockpit once, be logged in to Nextcloud and Immich too) needs an identity provider such as Authelia or Authentik in front of all three, plus Nextcloud's OIDC app and Immich's OAuth settings. Cockpit itself cannot hand out logins. TBD, parked by Ducu.

## API keys

- One **Keys** entry in the left menu listing every secret the plugins and scripts use: Immich keys and Nextcloud app passwords. Shown masked, with owner, where it is used, and a **Test** button (Immich `GET /users/me`, Nextcloud `occ user:auth-tokens:list`).
- **Replace**: paste a new key, the helper tests it first, then writes the file (root, mode 0600) and keeps the old one for one undo. **Create** a new Immich key through the Immich API for the owning user, and **Revoke** the old one there.
- Keys never reach the browser in full: the helper reads and writes them, the page only sees the last 4 characters.

## Immich photo workflows (part of the Immich plugin)

Generalised from Ducu's own keeper-curate scripts: an "inbox album → copy to the NAS photo folder → keep in a Stored album" workflow that anyone can switch on, with their own album names, folder pattern and schedule. Ducu's settings (toNAS, onNAS, Stored, `<year>/<MM_Mon>/`) are just the example values.

- The scripts' settings move out of the code into a config file: album names, the pool path, the bin folder and how many days it keeps files, the folder pattern, the stop limit, the scan wait, and the timer time.
- The page shows those settings as a form with checks (album must exist in Immich, path must exist), and a save restarts nothing until the next run.
- **Run** buttons per script: dry run first, then run, with live output and the log. One-off scripts go into a "Tools" list with dry run only by default, so they cannot be re-run by accident.
- The Immich page's health and container cards move into the Immich entry of the left menu, so there is one page instead of two.

## Architecture

A plain HTML/JS page that never runs commands itself, and one root helper that does all the work and answers in JSON, called with `cockpit.spawn` and `superuser: "require"`.

_Diagram (page, root helper, and what it touches): see the live plan doc._

**Files on the box**

- `/usr/share/cockpit/nas/` : `manifest.json`, `index.html`, `nas.js`, `nas.css` (sidebar entry "NAS Toolbox").
- `/usr/local/sbin/nas-admin` : Python 3 helper, root, mode 0700. `nas-admin status` (all read-only data as JSON), `nas-admin do <action> <args>` (password on stdin), `nas-admin log`.
- `/etc/nas-admin.conf` : settings (share root, enabled plugins and their settings, archive folder).

**Installer**

- `install-nas-toolbox.sh`, built by a `make-installer.sh` like the Immich one: preview by default, `--run` to install, `--remove` to uninstall. It moves the registry shares to smb.conf, removes cockpit-file-sharing and nfs-kernel-server, and installs Cockpit Files or Navigator.
- A one-line paste wrapper for the terminal.

### Menus and navigation

- **Sidebar**: one Cockpit entry, "NAS Toolbox" (searching "admin" or "nas" in Cockpit still finds it).
- **Inside NAS Toolbox**, a left menu: **General** (overview: services, disks, sessions, audit log), **People**, **Groups**, **Samba**, **Keys**, **Plugins**, plus one entry per enabled plugin (Immich, Nextcloud, FTP).
- **Plugins page**: every plugin with its buttons, on/off checkbox and settings form.
- **Refresh**: if a newly enabled plugin's entry does not show, the page asks Cockpit to re-read its packages (`cockpit.Packages.Reload`), then reloads the window (`window.top.location.reload()`). A manual "Refresh menu" button does the same.

### v1 architecture (planned)

Version 1 is the core only: People (Linux account + Samba user and password), Groups with the checkboxes, Samba shares and global settings on plain smb.conf, General, and the Plugins page. Immich, Nextcloud and FTP come next as plugins on the same contract.

**File structure (source, in the repository `cockpit-nas-toolbox/`)**

```
cockpit-nas-toolbox/
  README.md
  LICENSE                      GPL-3.0
  make-installer.sh            builds install-nas-toolbox.sh from the files below
  install-nas-toolbox.sh       generated; preview by default, --run, --remove
  helper/
    nas-admin                  entry point            -> /usr/local/sbin/nas-admin (0700)
    nasadmin/                  Python package         -> /usr/local/lib/nas-admin/nasadmin/
      core.py                  CLI, config, locking, audit log, dry-run runner, errors
      validate.py              field rules (username, group, path, share, password ...)
      smbconf.py               smb.conf parser / writer, testparm check, backups
      system.py                users, groups, pdbedit, services (read side)
      modules/
        people.py  groups.py  samba.py  overview.py
  page/                        Cockpit package        -> /usr/share/cockpit/nas/
    manifest.json  index.html  nas.css
    nas.js                     core UI: helper calls, menu, tables, forms, preview dialog
    modules/
      people.js  groups.js  samba.js  overview.js
  tests/
    test_smbconf.py  test_validate.py   unit tests, no root needed
    integration.sh                      real users and Samba, run as root in a throwaway container
```

**How a change flows**

1. The page builds a JSON request, for example `{"args": {"username": "ana", "password": "..."}, "dry_run": true}`, and sends it on stdin to `nas-admin do people add`. Passwords only travel on stdin.
2. The helper loads the module, checks every argument against the action's field rules, takes a lock, and runs the action through a runner. In dry-run mode the runner records each command instead of running it, with secrets masked.
3. The page shows that list of steps in a Preview dialog. Apply sends the same request with `dry_run: false`.
4. The helper runs the steps, stops at the first failure, writes one line to `/var/log/nas-admin.log`, and answers `{"ok": true, "steps": [...], "result": ...}` or `{"ok": false, "error": "..."}`.

**Safety rules built into the core**: accounts below uid 1000, root, members of sudo and the logged-in admin cannot be disabled or removed; only groups with gid 1000 and up can be managed; share paths must resolve inside the share root; smb.conf is only replaced after `testparm` passes, with the last 20 versions kept in `/var/lib/nas-admin/backups/`.

## Security

The page gets root only through Cockpit's own sudo prompt ("Administrative access").

- **Who can log in**: Admins have full NAS Toolbox. Supervisors see only their small People page. Everyone else never logs in to Cockpit; they only have file and app accounts with `nologin` shells.
- **One root entry point**: the page can only run `/usr/local/sbin/nas-admin` (root, mode 0700) with `superuser: "require"`. The helper accepts a fixed list of actions, never a raw command.
- **Input checks in the helper, not the page**: user and group names must match `^[a-z_][a-z0-9_-]{0,31}$`; uid below 1000 and root are refused; paths must resolve (after symlinks) inside the share root; share names are checked against a simple pattern.
- **Passwords**: passed on stdin, never on a command line (where `ps` would show them) and never logged. Minimum length enforced.
- **Nothing destructive by default**: remove = disable first, files moved to an archive folder, not deleted. Every change writes a line to `/var/log/nas-admin.log` (who, what, when), shown on the General page.
- **Backups before config changes**: the helper saves `smb.conf` before every share change, and `testparm` must pass before smbd reloads.
- **Network**: Cockpit (port 9090) stays reachable from the LAN only.

## Phasing

Each phase ships as its own paste-and-run installer (preview by default, `--run` to apply), so you can stop after any one.

1. **Phase 0, survey (read only).** One paste that prints: smb.conf and registry shares, Samba users (`pdbedit -L -v`), Linux users ≥ 1000 and groups, how Nextcloud and Immich are installed, folder owners and ACLs, which Cockpit plugins are available in apt, and which settings the photo scripts hard-code today. Changes nothing.
2. **Phase 1, read-only page.** General, People (merged list across Linux, Samba and the apps), shares, sessions, disk per share, Keys list with Test buttons, audit log.
3. **Phase 2, people and groups.** Groups with the checkboxes; add, reset password, disable, enable, remove, group membership. Then bulk add from CSV. Tested first on a test user.
4. **Phase 3, plugins and keys.** Connect or install Immich and Nextcloud from the Plugins page, user matching, quotas, NAS folders as External storage per group, rescan, health checks with the safe fix buttons. Replace, create and revoke API keys.
5. **Phase 4, photo scripts with settings.** Script settings in a config file, settings form, run / dry-run buttons for each script, Tools list for the one-off ones.
6. **Phase 5, shares and files.** Samba editor: move registry shares into smb.conf, all share and global options, access, folder ACLs; then remove the 45Drives plugin; install Cockpit Files or Navigator and add "Open in Files" links.
7. **Later, optional.** Email alerts (a service down, disk above 90%).

## Decisions and open questions

**Decided**

- Name: **NAS Toolbox**, used for the project and the Cockpit sidebar entry. The GitHub repository is `cockpit-nas-toolbox`.
- License: GPL-3.0.
- Languages: English, fully translatable (no fixed second language).
- Groups model: Admins, Supervisors, Users; group checkboxes only; the more generous group wins.
- Samba on plain smb.conf; 45Drives plugin dropped; NFS out.
- Same password for Samba and every plugin, set from one form.
- Immich, Nextcloud and FTP follow one plugin rule and live in NAS Toolbox's left menu.
- FTP mirrors Samba access, with an optional upload-only folder.
- Storage quota per user (warnings only on Samba); recycle-bin restore always allowed.
- File browser: **Cockpit Files** (`cockpit-files` 46 from `trixie-backports`, checked on vera-nas on Nov 6, 2026). Navigator is no longer needed.

**TBD (parked by Ducu)**

- Is "Open in Nextcloud" with the browser's own login enough, or is real single sign-on a must?
- Nextcloud is served over plain HTTP. Whether putting it behind HTTPS (reverse proxy or Tailscale) is part of NAS Toolbox.

**For sharing it**

- Public GitHub repository: https://github.com/DucuSandu/cockpit-nas-toolbox (created by Ducu).

**Facts the phase 0 survey will answer**

- Where Nextcloud runs (this LXC, Docker, AIO or another container) and the exact `occ` command.
- Which shares exist today in the registry and in smb.conf, so the move to plain smb.conf loses nothing.
- Whether `samba-ad-dc` (pulled in when Samba was installed) is inactive, so smbd runs as a plain file server.
