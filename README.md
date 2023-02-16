# Universal Wayland Session Manager

This is an experiment in creating a versatile tool for launching
any standalone Wayland WM with environment management and systemd integration.

WIP. Use at your onw risk.

## Concepts and features

- WM-specific behavior is handled by plugins
    - currently supported: sway, wayfire, labwc
- Maximum use of systemd units and dependencies for startup, operation, and shutdown
- Systemd units are treated with hierarchy and universality in mind:
    - use specifiers
    - named from common to specific: `wayland-${category}@${wm}.${unit_type}`
    - allow for high-level `name-.d` drop-ins
- Idempotently (well, best-effort-idempotently) handle environment:
    - On startup environment is prepared by:
        - sourcing shell profile
        - sourcing common `wayland-session-env` files (from $XDG_CONFIG_DIRS, $XDG_CONFIG_HOME)
        - sourcing `${wm}/env` files (from $XDG_CONFIG_DIRS, $XDG_CONFIG_HOME)
    - Difference between inital state and prepared environment is exported into systemd user manager
    - On shutdown variables that were exported are unset from systemd user manager
    - Lists of variables for export and cleanup are determined algorithmically by:
        - comparing environment before and after preparation procedures
        - boolean operations with predefined lists (tweakable by plugins)
- Better control of XDG autostart apps:
    - Propagate Stop from xdg-desktop-autostart.target via a drop-in
- Try best to shutdown session cleanly via more dependencies between units
- Written in POSIX shell (a smidgen of masochism went into this code)

## Installation

Put `wayland-session` script somewhere in `$PATH`.
Put `wayland-session-plugins` dir somewhere in `/lib:/usr/lib:/usr/local/lib:${HOME}/.local/lib`

Ensure your WM does this sequentially upon startup:

- exports `WAYLAND_DISPLAY` to systemd user manager
- runs `systemd-notify --ready` after that

Example snippet for sway:

    exec dbus-update-activation-environment --systemd \
    					SWAYSOCK \
    					DISPLAY \
    					I3SOCK \
    					WAYLAND_DISPLAY \
    					XCURSOR_SIZE \
    					XCURSOR_THEME \
    	&& systemctl --user import-environment \
    					SWAYSOCK \
    					DISPLAY \
    					I3SOCK \
    					WAYLAND_DISPLAY \
    					XCURSOR_SIZE \
    					XCURSOR_THEME \
    	&& systemd-notify --ready

## Slices

If your WM of choice is systemd-aware and supports launching apps explicitly scoped in app.slice,
or if you prefixed all app commands launched by WM with:

    systemd-run --user -qG --scope --slice=app.slice

then you can put WM in session.slice (as recommended by man systemd.special) by setting environment variable
`UWSM_USE_SESSION_SLICE=true` during `unitgen` phase. This will set `Slice=session.slice` for WM services.

Example snippet for sway on how to explicitly put apps in app.slice, scoped:

    set $scoper exec systemd-run --user -qG --scope --slice=app.slice
    bindsym --to-code $mod+t exec $scoper foot
    bindsym --to-code $mod+r exec $scoper rofi -show drun

## Full systemd service operation

### Short story:

Start with `wayland-session ${wm} sd-start` (it will hold while wayland session is running).

Use `exec wayland-session ${wm} sd-start` if you are in a login shell and want to bind login session's life to wayland session.

Stop with either `wayland-session ${wm} sd-stop` or `systemctl --user stop "wayland-wm@*.service"`

### Longer story:

(At least for now) Units are generated by the script (and plugins).

Run `wayland-session ${wm} unitgen` to populate `${XDG_RUNTIME_DIR}` with them.

After that: `systemctl --user start wayland-wm@${wm}.service` to start WM.

Add `--wait` to hold terminal until session ends.

`exec` it from login shell to bind to login session:

`exec systemctl --user start --wait wayland-wm@${wm}.service`

Then to stop: `systemctl --user stop "wayland-wm@*.service"` (no need to specify WM here).
If start command was run with `exec`, (i.e. from login shell on a tty or via `.profile`),
this stop command also doubles as a logout command.

`wayland-session` is smart enough to find login session associated with current TTY
and export (and later cleanup) `$XDG_SESSION_ID`, `$XDG_VTNR` to user manager environment.
Even though when started as a service it does not have those vars at immediate disposal.
(I really do not know it this is a good idea, but since there can be only one graphical session
per user with systemd, seems like such).

This example starts sway wayland session automatically upon login on tty1 if system is in `graphical.target`

Short snippet for `~/.profile`:

    if [ "${0}" != "${0#-}" -a "$XDG_VTNR" = "1" ]
    then
        exec wayland-session sway sd-start
    fi

Extended snippet for `~/.profile`:

    WM=sway
    if [ "${0}" != "${0#-}" -a "$XDG_VTNR" = "1" ] \
      && systemctl is-active -q graphical.target \
      && ! systemctl --user is-active -q wayland-wm@*.service
    then
        wayland-session ${WM} unitgen
        trap "if systemctl --user is-active -q wayland-wm@${WM}.service ; then systemctl --user --stop wayland-wm@${WM}.service ; fi" INT EXIT TERM
        echo Starting ${WM} WM
        systemctl --user start --wait wayland-wm@${WM}.service &
        wait
        exit
    fi

## WM-specific actions

Plugins provide WM support and associated functions. See `wayland-session-plugins/*.sh.in` for examples.

## TODO

- more plugins
- invent a better way to stop xdg-desktop-portal-gtk.service on WM stop
- maybe do some integration with `/usr/share/wayland-sessions/*.desktop`

## Compliments

Inspired by and adapted some techniques from:

- [sway-services](https://github.com/xdbob/sway-services)
- [sway-systemd](https://github.com/alebastr/sway-systemd)
- [sway](https://github.com/swaywm/sway)
- [Presentation by Martin Pitt](https://people.debian.org/~mpitt/systemd.conf-2016-graphical-session.pdf)
