# VNC server setup for Fedora

Two Bash scripts that turn a Fedora machine into a VNC server with a full
desktop, step by step:

- **`vnc-setup`** installs TigerVNC (the maintained fork of TightVNC, with the
  same Tight encoding) and an X11 desktop, and runs a virtual desktop for one
  user as the systemd service `vncserver@:N`.
- **`vnc-desktop`** switches which desktop that VNC server starts, then
  restarts it cleanly.

## Requirements

- Fedora (the script checks `/etc/os-release`)
- `sudo` rights (the script re-runs itself with `sudo` when needed)

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
2. **TigerVNC**: installs `tigervnc-x11-server` and `tigervnc-selinux`.
3. **Desktop**: Xfce (the default, and the best fit for VNC), KDE Plasma on
   X11, Cinnamon, or GNOME Flashback. It installs the one you pick if needed.
4. **Display number**: display `:N` listens on TCP port `5900+N`
   (`:1` is port 5901). It is recorded in `/etc/tigervnc/vncserver.users`.
5. **VNC password**: separate from the login password; only the first 8
   characters count. On a first run this must be typed in, so `-y` stops here.
6. **Screen size and access**:
   - *SSH tunnel only* (recommended): the server listens on localhost only and
     SSH encrypts the connection. The script enables `sshd` if it is not
     running.
   - *Directly from the network*: opens the port in firewalld. TigerVNC
     encrypts with TLS, but not every viewer supports it.
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

It updates `session=` in `~/.config/tigervnc/config` and restarts the VNC
server, first stopping the systemd user services KDE and GNOME leave behind so
they don't take over the next desktop.

## Limitations

- **GNOME Shell cannot run under VNC** (it no longer has an X11 session), so
  `gnome` starts GNOME Flashback instead.
- **One GNOME or KDE session per user**: while VNC runs GNOME or KDE for a
  user, that user cannot log in to GNOME or KDE on the console, and vice
  versa. Xfce and Cinnamon have no such limit.

## Troubleshooting

```sh
systemctl status vncserver@:1          # is it running?
journalctl -u vncserver@:1             # why it failed
sudo systemctl restart vncserver@:1    # apply config changes
```

Settings such as `session`, `geometry` and `localhost` live in
`~/.config/tigervnc/config`.

## License

Copyright (C) 2026 Dominique Schaefer. Licensed under the GNU General Public
License, version 3 or (at your option) any later version. See
[LICENSE](LICENSE).
