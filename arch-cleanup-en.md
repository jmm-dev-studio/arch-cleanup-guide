# Keeping Arch Linux Lean: A Safe Cleanup Routine for Cache, Orphans and Logs

*By JMM Dev Studio*

Arch Linux does not clean up after itself. That is a feature: pacman keeps old package versions so you can downgrade, and systemd keeps logs so you can debug. But after a year of updates, these leftovers can eat tens of gigabytes. This guide shows a short, safe routine you can run once a month. Every command is explained, and nothing here deletes your personal files.

## 1. See where the space went

Before deleting anything, measure. Start with the package cache and the journal:

```
du -sh /var/cache/pacman/pkg
journalctl --disk-usage
```

On a system that has not been cleaned for a while, the first number is often 5 to 30 GB.

For an interactive view of the whole disk, install `ncdu` and run it on the root filesystem without crossing into other mounts:

```
sudo pacman -S ncdu
sudo ncdu -x /
```

## 2. Trim the pacman cache

Every package you ever installed or updated is still in `/var/cache/pacman/pkg`. The simplest tool is `paccache`, shipped in the `pacman-contrib` package:

```
sudo pacman -S pacman-contrib
paccache -d
```

The `-d` flag is a dry run: it only lists what would be removed. By default, `paccache` keeps the three most recent versions of each package. When the list looks right, run it for real:

```
sudo paccache -r
```

If you want to keep fewer versions, use `-k`:

```
sudo paccache -rk1
```

You can also remove the cache of packages that are no longer installed:

```
sudo paccache -ruk0
```

Avoid `pacman -Scc` unless you are short on space and understand the cost. It deletes the whole cache, which means you lose the ability to downgrade a broken update without downloading the old package again.

Optionally, let systemd do this for you every week:

```
sudo systemctl enable --now paccache.timer
```

## 3. Remove orphaned packages

Orphans are packages that were installed as dependencies but are no longer required by anything. List them first:

```
pacman -Qtdq
```

If the list is empty, pacman prints nothing and returns an error code. That is normal. Read the list carefully. If you recognise a package you still want, mark it as explicitly installed so it stays:

```
sudo pacman -D --asexplicit packagename
```

When you are happy with the list, remove the orphans together with their configuration files:

```
sudo pacman -Rns $(pacman -Qtdq)
```

Fish users can write the same thing as `sudo pacman -Rns (pacman -Qtdq)`.

## 4. Limit the systemd journal

The journal can grow to several gigabytes. Check the size and trim it to something reasonable:

```
sudo journalctl --vacuum-size=200M
```

You can also delete by age:

```
sudo journalctl --vacuum-time=4weeks
```

To make the limit permanent, create a drop-in file instead of editing the main config:

```
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=200M\n' | sudo tee /etc/systemd/journald.conf.d/size.conf
sudo systemctl restart systemd-journald
```

## 5. Look at your AUR build files and caches

If you use an AUR helper such as `yay`, it keeps cloned build directories under `~/.cache/yay`. These are safe to delete, because the helper clones them again when needed:

```
du -sh ~/.cache/yay
yay -Sc
```

Review what `yay -Sc` asks before you confirm. For other cached data in `~/.cache`, check the largest folders first:

```
du -sh ~/.cache/* | sort -h | tail
```

Only delete caches of programs you recognise. Never delete `~/.config` or `~/.local/share` as part of a cleanup.

## 6. Check for failed services and leftover config files

Cleanup is also a good moment for a health check:

```
systemctl --failed
sudo find /etc -name "*.pacnew" -o -name "*.pacsave"
```

A `.pacnew` file means a package shipped a new default configuration and pacman did not overwrite yours. Compare them with `diff` or the `pacdiff` tool from `pacman-contrib`, merge what you need, and delete the leftovers.

## A monthly routine in five lines

```
sudo paccache -rk2
sudo pacman -Rns $(pacman -Qtdq)
sudo journalctl --vacuum-size=200M
systemctl --failed
sudo find /etc -name "*.pacnew"
```

Run the orphan line only after reading the list. Automation is good for the cache and the journal, but package removal deserves a human look.

## Summary

- Measure first with `du`, `journalctl --disk-usage` and `ncdu`.
- `paccache` is safer than `pacman -Scc` because it keeps recent versions for downgrades.
- Check orphans with `pacman -Qtdq` before removing them.
- Cap the journal with a drop-in file so the limit survives reboots.
- Never clean `~/.config` or `~/.local/share` blindly.

With this routine, a typical Arch system stays small and predictable, and you keep your safety net for rollbacks.
