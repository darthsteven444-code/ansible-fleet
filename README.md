# ansible-fleet

Patching and hardening for the Linux boxes in my homelab, all run from one control node.

I got tired of SSHing into every box to run apt upgrade and hoping I didn't forget one. So now one command patches everything, and the Kubernetes nodes go one at a time so my site never goes down while it happens.

## what's in here

- `inventory.ini` is the fleet. linux-lab is the control node and manages itself, plus my three k3s nodes.
- `report.yml` is read only. Pending updates, security updates, kernel, uptime, and whether a reboot is waiting.
- `harden.yml` is the baseline for every box. SSH keys only, no root login, fail2ban, kernel network settings, and no automatic reboots.
- `patch.yml` patches the control node, then rolls through the k3s nodes. Drain, upgrade, reboot only if needed, wait for Ready, put it back in service, wait for the site and the SIEM to be healthy, then the next node.

## how I run it

```
# dry run, shows every change on every box before anything happens
ansible-playbook report.yml harden.yml --check --diff -K

# the real thing
ansible-playbook harden.yml patch.yml report.yml -K
```

I run it inside tmux. Patching the box Ansible lives on can restart the same services that are carrying my SSH session, and tmux keeps the run going if I get kicked off.

## the guard rails

- harden refuses to turn off password login on any box that doesn't already have an SSH key in place. It can't lock me out.
- The full SSH config gets checked with `sshd -t` on every run before anything reloads.
- patch stops the whole rollout if one node fails, and puts that node back in service on the way out.
- Automatic reboots are off everywhere. Security updates still install on their own, but an unattended reboot once hung one of my nodes half shut down for 23 hours, so reboots happen on purpose now.

## first real run

4 boxes, 61 to 80 pending updates on each one, 6 to 11 of them security fixes. About 20 minutes start to finish. All three k3s nodes drained, patched and rebooted one at a time, 0 failed tasks, and my monitoring didn't fire a single real alert the whole time.

Write up: https://steven-garcia.dev/projects/ansible-fleet/
