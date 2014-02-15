copied my public key to authorized_keys but public-key authentication still doesn't work.

Typically this is caused by the file permissions on $HOME, $HOME/.ssh or $HOME/.ssh/authorized_keys being more permissive than sshd allows by default.

In this case, it can be solved by executing the following on the server.

$ chmod go-w $HOME $HOME/.ssh
$ chmod 600 $HOME/.ssh/authorized_keys
$ chown `whoami` $HOME/.ssh/authorized_keys
