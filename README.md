# qlink

A tiny ssh launcher for the terminal. Run `qlink`, browse a table of your
machines, hit a key, you're connected. One file, stdlib Python only, no
dependencies and nothing to build.

```
 qlink                                          2/2 hosts  ~/.config/qlink/qlink.conf
  KEY  NAME        USER    HOST             PORT   STATUS       DESCRIPTION
  1    homelab_1   root    192.168.0.230    22122  up           internal network
  2    lgforms_2   deploy  10.213.123.12    22122  unreachable  intentlab (wireguard required)

 enter/key connect   ↑↓ jk move   / search   r reload   q quit
```

No colours: bold, grey and a reverse-video cursor line, so it sits on whatever
terminal theme you already use. The single exception is a red `unreachable`.

## Install

`qlink` is a single executable file. To type `qlink` from any directory it has
to live in a folder that is on your `PATH`. Pick one of these.

### Symlink into a folder already on your PATH (recommended)

A symlink means edits to the file in this repo take effect immediately - there
is nothing to reinstall.

```sh
cd /Users/q/Projects/bitroot/qlink
chmod +x qlink
mkdir -p ~/.local/bin
ln -sf "$PWD/qlink" ~/.local/bin/qlink
```

Check that `~/.local/bin` is on your PATH:

```sh
echo $PATH | tr ':' '\n' | grep '\.local/bin'
```

If that prints nothing, add it to your shell startup file (macOS defaults to
zsh) and reload:

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Then, from anywhere:

```sh
qlink
```

If your shell still says `command not found`, it cached the old PATH - run
`hash -r` (zsh: `rehash`) or open a new terminal tab.

### Alternatives

```sh
# homebrew's bin, already on PATH on Apple silicon - no shell edit needed
ln -sf "$PWD/qlink" /opt/homebrew/bin/qlink

# or keep the repo itself on PATH
echo 'export PATH="$HOME/Projects/bitroot/qlink:$PATH"' >> ~/.zshrc

# or copy it somewhere and forget about the repo (no live edits)
cp qlink ~/.local/bin/qlink
```

### Uninstall

```sh
rm ~/.local/bin/qlink        # and remove the PATH line from ~/.zshrc
rm -rf ~/.config/qlink       # if you kept your hosts there
```

### Requirements

macOS ships everything needed: Python 3 (`python3 -V`) and `ssh`. No pip
install, no build step.

First run with no config creates a starter one at `~/.config/qlink/qlink.conf`.

## Config

Plain text, one host per line:

```
key,name,user@host,port,description
```

```
# qlink hosts:  key,name,user@host,port,description
default_user = root

1,homelab_1,192.168.0.230,22122,internal network
2,lgforms_2,deploy@10.213.123.12,22122,intentlab (wireguard required)
a,work-jump,deploy@jump.example.com,,bastion - port defaults to 22
b,nouser,plain.example.com,,uses default_user if set, else your local username
```

* **key** — the shortcut. A single digit or letter; pressing it in the table
  connects immediately. Leave it empty and qlink numbers the line for you.
  `q j k g r c /` are taken by the table itself - qlink warns if you use one.
* **user@host** — hostname or IP, with an optional `user@` prefix. The user
  gets its own column in the table.
* **port** — optional, defaults to 22.
* **description** — optional, may contain commas.

### Username

Three ways, most specific wins:

1. `user@` on the line itself — `2,lgforms_2,deploy@10.213.123.12,22122,...`
2. `default_user = root` on its own line in the config — applies to every line
   below it that has no `user@`. Put several in the file to switch part-way
   down.
3. `QLINK_USER=root qlink` — an environment default, overridden by both above.

With none of them set, ssh falls back to your local username (or whatever
`~/.ssh/config` says for that host).

Blank lines and `#` comments are ignored.

## Reachability

Each host is probed in the background so the table is usable the instant it
opens - nothing is ever waited on. Rows start as `…` and fill in as answers
arrive; a host that does not answer turns red `unreachable`. Press `c` to probe
again.

The default probe is a TCP connect to the host's ssh port, which is what
actually matters for connecting: it survives hosts that drop ICMP, and it tells
you sshd is listening rather than merely that the box is up. Switch it with a
line in the config:

```
check = tcp     # default - TCP connect to the ssh port
check = ping    # ICMP ping instead, ignores the port
check = off     # no probing, no STATUS column
```

or per run with `QLINK_CHECK=ping qlink`. Probes run at most 8 at a time with a
1.5s timeout, on daemon threads - quitting qlink never waits for one.

`qlink -c` runs the same probes in the terminal and prints what qlink sees,
which is the quickest way to tell a real outage from a false alarm:

```
   1  homelab_1     q@192.168.1.230:22      up
   2  lgforms_2     10.213.123.12:22122     unreachable

(1/2 reachable, probed with nc)
```

### A LAN host shows unreachable but ssh works

macOS Local Network privacy can deny a binary access to `192.168.x.x` /
`10.x.x.x`, and a denied socket fails instantly with `EHOSTUNREACH`
("No route to host") - so Python sees a dead host while `ssh` and `ping`,
which hold their own permission, work fine.

qlink detects exactly that error and transparently re-probes through
`/usr/bin/nc`, so it reports the truth either way; `qlink -c` prints
`probed with nc` when the fallback is active. To use the faster native sockets
instead, tick your terminal (and `python3` if it is listed) under
**System Settings > Privacy & Security > Local Network**.

A host behind wireguard shows `unreachable` until the tunnel is up, which is
usually the answer you wanted before typing anything.

Config is the first of these that exists:

1. `$QLINK_CONFIG`
2. `~/.config/qlink/qlink.conf`
3. `~/.qlink.conf`
4. `qlink.conf` next to the script (symlinks resolved, so the copy in this
   repo counts)

## Keys

| key | action |
| --- | --- |
| `1` `2` `a` … | connect to that host straight away |
| `enter` | connect to the highlighted row |
| `↑` `↓` / `j` `k` | move |
| `g` `G`, pgup/pgdn | top / bottom, page |
| `/` | search — type to filter on any column, `enter` connects |
| `esc` | clear the filter / leave search |
| `c` | re-run the reachability checks |
| `r` | reload the config file (and re-check) |
| `q` | quit |

Typing any character that isn't a shortcut key also starts a search, so you can
just type `lgf` and hit enter.

When the ssh session ends qlink exits straight to the shell — there is no
prompt. Run `qlink` again to pick another host.

## From the command line

```sh
qlink 2          # connect by shortcut key
qlink lgforms_2  # ...or by name / unique prefix
qlink -l         # list hosts
qlink -c         # probe every host, print up/unreachable, exit 1 if any are down
qlink --edit     # edit the config (-e; uses $VISUAL/$EDITOR, default vim)
qlink -h         # help
```

Extra arguments are passed through to ssh: `qlink 1 -L 8080:localhost:80`.
