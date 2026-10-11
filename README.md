# VNC server setup for Fedora, Debian, Ubuntu and Linux Mint

Two Bash scripts that turn a Linux machine into a VNC server with a full
desktop, step by step:

- **`vnc-setup`** installs TigerVNC (the maintained fork of TightVNC, with the
  same Tight encoding) and an X11 desktop, and runs a virtual desktop for one
  user as the systemd service `vncserver@:N` (`tigervncserver@:N` on Debian,
  Ubuntu and Mint).
- **`vnc-desktop`** switches which desktop that VNC server starts, then
  restarts it cleanly.

## Requirements

- A supported distribution: Fedora, Debian, Ubuntu or Linux Mint (see
  [Compatibility](#compatibility))
- `sudo` rights (the script re-runs itself with `sudo` when needed)

## Compatibility

| Distribution | TigerVNC | Service | Settings | Status |
|---|---|---|---|---|
| Fedora 45 | 1.16 | `vncserver@:N` | `~/.config/tigervnc` | original platform |
| Fedora 44 | 1.16 | `vncserver@:N` | `~/.config/tigervnc` | tested |
| Ubuntu 26.10 | 1.15 | `tigervncserver@:N` | `~/.config/tigervnc` | tested |
| Ubuntu 26.04 LTS | 1.15 | `tigervncserver@:N` | `~/.config/tigervnc` | same packages as 26.10 |
| Ubuntu 24.04 LTS | 1.13 | `tigervncserver@:N` | `~/.vnc` | tested |
| Debian 13 (trixie) | 1.15 | `tigervncserver@:N` | `~/.config/tigervnc` | tested |
| Debian 12 (bookworm) | 1.12 | `tigervncserver@:N` | `~/.vnc` | tested |
| Linux Mint 22.x | 1.13 | `tigervncserver@:N` | `~/.vnc` | via its base, Ubuntu 24.04 |
| LMDE 7 | 1.15 | `tigervncserver@:N` | `~/.config/tigervnc` | via its base, Debian 13 |

The scripts recognise the distribution by `ID` and `ID_LIKE` in
`/etc/os-release`, so other releases of these families are accepted too, but
they are untested. TigerVNC before 1.14 reads its settings from `~/.vnc`
instead of `~/.config/tigervnc`; the scripts pick the folder from the
installed version.

### What differs between the families

| | Fedora | Debian, Ubuntu, Mint |
|---|---|---|
| VNC packages | `tigervnc-x11-server`, `tigervnc-selinux` | `tigervnc-standalone-server`, `tigervnc-tools` |
| Service | `vncserver@:N` | `tigervncserver@:N` |
| Password tool | `vncpasswd` | `tigervncpasswd` |
| Firewall | firewalld | ufw, when it is enabled |
| SSH server | `sshd` | `ssh`, or `ssh.socket` on Ubuntu |
| User D-Bus | dbus-broker | dbus-daemon (or dbus-broker if installed) |

### How it was tested

- Each release marked *tested* ran in a systemd container: a first
  `vnc-setup -y` run, a VNC password, a second run, then a check that Xvnc,
  `xfce4-session` and `xfwm4` were running with the expected config.
- `vnc-setup` was re-run three times and `vnc-desktop` switched three times
  on every release; the desktop came back each time.
- On Ubuntu 26.10, ufw opened the VNC port (IPv4 and IPv6) for network
  access, and a tunnel-only run closed it again. The SSH server was enabled
  on every release.
- KDE Plasma, Cinnamon and GNOME Flashback were checked by resolving their
  package sets on each Debian-family release, not run end to end.

## Setup

```sh
git clone https://github.com/dschaefer79/tightvnc_server_setup.git
cd tightvnc_server_setup
./vnc-setup        # asks each question, with defaults in [brackets]
./vnc-setup -y     # takes every default without asking
```

Run it from your own account: it uses the user behind `sudo` as the default
VNC user. It walks through these steps:

1. **User**: which account gets the VNC desktop (not root).
2. **TigerVNC**: installs `tigervnc-x11-server` and `tigervnc-selinux` on
   Fedora, `tigervnc-standalone-server` and `tigervnc-tools` elsewhere.
3. **Desktop**: Xfce (the default, and the best fit for VNC), KDE Plasma on
   X11, Cinnamon, or GNOME Flashback. It installs the one you pick if needed:

   | Desktop | Fedora | Debian, Ubuntu, Mint |
   |---|---|---|
   | Xfce | `@xfce-desktop` | `xfce4` |
   | KDE Plasma (X11) | `plasma-workspace-x11` | `plasma-desktop` (+ `plasma-session-x11` on Ubuntu 26.04 and later) |
   | Cinnamon | `@cinnamon-desktop` | `cinnamon-core` |
   | GNOME Flashback | `gnome-flashback` | `gnome-session-flashback` |

4. **Display number**: display `:N` listens on TCP port `5900+N`
   (`:1` is port 5901). It is recorded in `/etc/tigervnc/vncserver.users`.
5. **VNC password**: separate from the login password; only the first 8
   characters count. On a first run this must be typed in, so `-y` stops here.
6. **Screen size and access**:
   - *SSH tunnel only* (recommended): the server listens on localhost only and
     SSH encrypts the connection. The script enables the SSH server if it is
     not running (`sshd` on Fedora, `ssh` or `ssh.socket` elsewhere).
   - *Directly from the network*: opens the port in firewalld or ufw,
     whichever is running. TigerVNC encrypts with TLS, but not every viewer
     supports it.
7. **Start**: enables `vncserver@:N` at boot and starts it.
8. **`vnc-desktop`**: optionally installs it to `/usr/local/bin`.

At the end it prints how to connect.

Running `vnc-setup` again is safe: it offers the user's existing display and
keeps the password unless you ask to change it, so you can use it to change
the desktop, screen size or access mode.

## Connecting

With the SSH tunnel (display `:1` shown):

```sh
ssh -L 5901:localhost:5901 you@server
vncviewer localhost:1
```

On Windows, PuTTY can make the same tunnel under
*Connection > SSH > Tunnels* (source port `5901`, destination
`localhost:5901`).

With direct access, point the viewer at `server:1`.

## Switching the desktop

```sh
vnc-desktop                                # pick from a menu
vnc-desktop xfce|cinnamon|kde|gnome [-y]   # switch and restart (-y: don't ask)
vnc-desktop --status                       # show the current desktop
```

It updates `session=` in the TigerVNC config (see [Compatibility](#compatibility)) and restarts the VNC
server, first stopping the systemd user services KDE and GNOME leave behind so
they don't take over the next desktop.

## Limitations

- **GNOME Shell cannot run under VNC** (it no longer has an X11 session), so
  `gnome` starts GNOME Flashback instead.
- **KDE Plasma** runs its X11 session: `plasmax11` with Plasma 6, `plasma`
  with Plasma 5 (Debian 12, Ubuntu 24.04, Mint 22).
- **One GNOME or KDE session per user**: while VNC runs GNOME or KDE for a
  user, that user cannot log in to GNOME or KDE on the console, and vice
  versa. Xfce and Cinnamon have no such limit.

## Troubleshooting

```sh
systemctl status vncserver@:1          # is it running?
journalctl -u vncserver@:1             # why it failed
sudo systemctl restart vncserver@:1    # apply config changes
```

On Debian, Ubuntu and Mint the service is `tigervncserver@:1`.

Settings such as `session`, `geometry` and `localhost` live in
`~/.config/tigervnc/config` (`~/.vnc/config` with TigerVNC before 1.14).

## License

Copyright (C) 2026 Dominique Schaefer. Licensed under the GNU General Public
License, version 3 or (at your option) any later version. See
[LICENSE](LICENSE).
