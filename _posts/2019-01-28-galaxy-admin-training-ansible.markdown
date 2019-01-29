---
layout:     post
title:      "The Ansible Playbook you build isn't the Ansible Playbook you run"
date:       2019-01-28 23:39:25 -0500
tags:       ansible systems
---
# The Mistake

A common practice of mine in developing [Ansible][ansible] playbooks is to write a few pieces (add a role, write some
tasks, etc.), run the playbook, add some more, run again, and so on. At the end, I have a working playbook that I can
apply to other hosts - or so I think.

The mistake is that running that playbook on another target runs the whole playbook at once, not incrementally as I did
when I was writing it. This means that unless I only ever add new tasks and roles after the ones I've already run, and I
never go back and make changes to earlier ones, *the order of tasks will be different*, but that might not be obvious.
Here's an example.

[ansible]: https://www.ansible.com/

# Example

This week some colleagues and I are teaching the [2019 Galaxy Admin Training][gatc] at Penn State. For this training, we
decided to do away with command line editing as much as possible (although a student may choose not to use Ansible when
they get home, the methods taught here apply to deployment and configuration management in general). This means we
(which is to say, mostly [Helena][helena]) developed a [training exercise][exercise] for deploying [Galaxy][galaxy] with
Ansible.

As I do in development, we had the students build their playbooks step by step. At the end of the day, in preparation
for the next day, I ran the playbook on all the unused training cloud instances so that they'd be up to date, if anyone
needed to switch instances. That's where the problems began:

```json
TASK [Create the Galaxy FTP upload dir]
******************************************************************************
fatal: [jan-2019-training-inst-3]: FAILED! => {
    "changed": false,
    "msg": "chown failed: failed to look up user galaxy",
}
```

To understand what's happened, have a look at the playbook:

```yaml
- hosts: galaxyservers
  pre_tasks:
    - name: Create the Galaxy FTP upload dir
      file:
        path: "/srv/galaxy/ftp"
        owner: galaxy
        group: galaxy
        mode: 0750
        state: directory
  roles:
    - galaxyproject.repos
    - galaxyproject.postgresql
    - role: natefoo.postgresql_objects
      become: true
      become_user: postgres
    - galaxyproject.galaxy
    - usegalaxy-eu.supervisor
    - geerlingguy.nginx
    - galaxyproject.proftpd
```

When building the playbook, we added the FTP dir creation `pre_task` when adding the `galaxyproject.proftpd` role (and
we added the roles in the order seen, running the playbook between each one). **The `galaxyproject.galaxy` role creates
the `galaxy` user**. So on a fresh target system, the `pre_task` no longer works, since we now run it *before* the role
that creates the user, when the first time around, we ran it *after*.

Running it as a `post_task` doesn't work either. Galaxy starts when `usegalaxy-eu.supervisor`'s handlers fire, at the
end of all the role executions, before any `tasks` or `post_tasks` run. And when that happens, you get:

```json
RUNNING HANDLER [Restart Galaxy] *********************************************
fatal: [jan-2019-training-inst-3]: FAILED! => {
     "changed": false,
     "msg": "galaxy: stopped\ngalaxy: ERROR (spawn error)\n"
}
```

And the error is:

```
ubuntu@jan-2019-training-inst-3:~$ sudo supervisorctl tail galaxy stderr
f2f0
python threads support enabled
your server socket listen backlog is limited to 100 connections
your mercy for graceful operations on workers is 60 seconds
mapped 306752 bytes (299 KB) for 4 cores
created farm 1 name: job-handlers mules:1,2
*** Operational MODE: threaded ***
added /srv/galaxy/server/lib/ to pythonpath.
DEBUG:galaxy.app:python path is: /srv/galaxy/server/lib/, ., , /srv/galaxy/venv/bin, /srv/galaxy/venv/lib/python2.7, /srv/galaxy/venv/lib/python2.7/plat-x86_64-linux-gnu, /srv/galaxy/venv/lib/python2.7/lib-tk, /srv/galaxy/venv/lib/python2.7/lib-old, /srv/galaxy/venv/lib/python2.7/lib-dynload, /usr/lib/python2.7, /usr/lib/python2.7/plat-x86_64-linux-gnu, /usr/lib/python2.7/lib-tk, /srv/galaxy/venv/local/lib/python2.7/site-packages, /srv/galaxy/venv/lib/python2.7/site-packages
DEBUG:galaxy.containers:config file './config/containers_conf.yml' does not exist, running with default config
Traceback (most recent call last):
  File "/srv/galaxy/server/lib/galaxy/webapps/galaxy/buildapp.py", line 49, in app_factory
    app = galaxy.app.UniverseApplication(global_conf=global_conf, **kwargs)
  File "/srv/galaxy/server/lib/galaxy/app.py", line 67, in __init__
    self.config.check()
  File "/srv/galaxy/server/lib/galaxy/config.py", line 830, in check
    self._ensure_directory(path)
  File "/srv/galaxy/server/lib/galaxy/config.py", line 815, in _ensure_directory
    raise ConfigurationError("Unable to create missing directory: %s\n%s" % (path, e))
ConfigurationError: Unable to create missing directory: /srv/galaxy/ftp
[Errno 13] Permission denied: '/srv/galaxy/ftp'
```

We're running Galaxy with privilege separation, so `/srv/galaxy` is owned by root. This failure is the reason we added
the `pre_task` in the first place.

The solution in this case was relatively simple: `galaxyproject.proftpd` has an option to create the FTP directory,
which I enabled. If not, however, this would be difficult to fix. There's no way to insert tasks between roles unless
you run all of your roles from tasks using [include_role][include-role], which is a hack. Assuming I don't want to
modify roles I've fetched from [Ansible Galaxy][ansible-galaxy] (no relation) to create the dir, the next best solution
would be to create a single-task role to run after `galaxyproject.galaxy` to create the FTP directory. Not ideal.

[gatc]: https://galaxyproject.org/events/2019-admin-training/
[helena]: https://github.com/erasche
[exercise]: https://training.galaxyproject.org/topics/admin/tutorials/ansible-galaxy/tutorial.html
[galaxy]: https://galaxyproject.org/
[include-role]: https://docs.ansible.com/ansible/latest/modules/include_role_module.html
[ansible-galaxy]: https://galaxy.ansible.com/

# Discussion

There are really two problems here, as far as I'm concerned.

1. Ansible doesn't have a way to specify that tasks can fail *and should be retried at a later time*. I actually came to
   Ansible from CFEngine which (it's been a few years, but as I recall) ran in *passes* - actions it attempted could
   fail, but it'd run other actions anyway and retry the failed ones later. Ansible has no such mechanism, its execution
   is pretty much linear. The fact that you can't insert a task between roles is really just an annoyance that makes it
   difficult to work around this larger complaint.

2. I try to write the world's most generalized and unopinionated roles, but Ansible does not make this easy. Creating
   directories isn't always as simple as you might think. Naïve roles will just use a `become` or assume they are
   running with the appropriate privileges, but neither of these are appropriate for a role that's going to be used by
   others, and could downright fail if the directory is on, for example, NFS with root squashing.  The
   `galaxyproject.proftpd`'s option to create the FTP directory is disabled by default for this reason (indeed, I myself
   can't use it for [usegalaxy.org][usegalaxy-org]).  This is why I end up writing a lot of my own roles, and the reason
   they end up so complex.

   There is no construct in Ansible for a role to assert that it needs certain privileges, and for the consumer of
   the role to configure the method by which those privileges should be obtained. You run a role as either the user you
   logged in as, or with `become`, but that's too limiting. The [galaxyproject.galaxy role][galaxy-role], for example,
   needs to create some files as one user and some as another, and execute commands or perform actions as both of those
   users. If it can't do that, you either have to:

    1. Run everything as one user, this is less secure,
    2. Use "enable/disable" variables and a bunch of `when`s on a bunch of `include_tasks`'s to break the role up by
       privilege, then instruct role consumers on how to run the role multiple times with the correct combination of
       `remote_user` or `become`/`become_user` and the enable/disable vars set as role vars, or
    3. Some kind of [nutty][pr51] [kludgery][pr55] to actually provide the missing assert/configure functionalty.

  Somewhat related, but Ansible does not make it easy to see what privileges it's obtaining and how it's obtaining them.
  You can see most of this with `-vvv` but it's not easy: The ssh user is displayed on the first line after the task
  header, but only if `ansible_user`/`remote_user` is set on some level. The `become_user` is buried in the module
  execution command lines.

[usegalaxy-org]: https://usegalaxy.org/
[galaxy-role]: https://github.com/galaxyproject/ansible-galaxy.git
[pr51]: https://github.com/galaxyproject/ansible-galaxy/pull/51
[pr55]: https://github.com/galaxyproject/ansible-galaxy/pull/55

# Conclusion

My complaints in the discussion aside, the takeaway is that simply building a playbook that works is not sufficient, it
needs to be run again from scratch. In addition, any time you're adding to it later, always consider whether the tasks
you're adding depend on other things set up by the playbook. If they are, further consider whether those dependencies
run earlier or later in playbook. Forgetting this consideration is a trap I've fallen in to many times.
