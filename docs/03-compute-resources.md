# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the machines required for setting up a Kubernetes cluster.


## Machine Database
This tutorial will leverage a text file, which will serve as a machine database, to store the various machine attributes that will be used when setting up the Kubernetes control plane and worker nodes. The following schema represents entries in the machine database, one entry per line:

`IPV4_ADDRESS HOSTNAME`

Each of the columns corresponds to a machine IP address IPV4_ADDRESS, and a host name HOSTNAME.

Here is an example machine database similar to the one used when creating this tutorial. Notice the IP addresses have been masked out. Your machines can be assigned any IP address as long as each machine is reachable from each other and the jumpbox.

```bash
cat machines.txt
```

```text
XXX.XXX.XXX.XXX controller1
XXX.XXX.XXX.XXX controller2
XXX.XXX.XXX.XXX controller3
XXX.XXX.XXX.XXX worker1
```

Now it's your turn to create a machines.txt file with the details for the three machines you will be using to create your Kubernetes cluster. Use the example machine database from above and add the details for your machines.

## Configuring SSH Access

SSH will be used to configure the machines in the cluster. Verify that you have `root` SSH access to each machine listed in your machine database. You may need to enable root SSH access on each node by updating the sshd_config file and restarting the SSH server.

### Enable root SSH Access

If `root` SSH access is enabled for each of your machines you can skip this section.

By default, a new `debian` install disables SSH access for the `root` user. This is done for security reasons as the `root` user has total administrative control of unix-like systems. If a weak password is used on a machine connected to the internet, well, let's just say it's only a matter of time before your machine belongs to someone else. As mentioned earlier, we are going to enable `root` access over SSH in order to streamline the steps in this tutorial. Security is a tradeoff, and in this case, we are optimizing for convenience. Log on to each machine via SSH using your user account, then switch to the `root` user using the `su` command:

```bash
su - root
```

Edit the `/etc/ssh/sshd_config` SSH daemon configuration file and set the `PermitRootLogin` option to `yes`:

```bash
sed -i \
  's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  /etc/ssh/sshd_config
```

Restart the `sshd` SSH server to pick up the updated configuration file:

```bash
systemctl restart sshd
```

### Generate and Distribute SSH Keys

In this section you will generate and distribute an SSH keypair to all machines, which will be used to run commands on those machines throughout this tutorial. Run the following commands from the `jumpbox` machine.

Generate a new SSH key:

```bash
ssh-keygen
```

```text
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
```

Copy the SSH public key to each machine:

```bash
while read IP HOST; do
  ssh-copy-id root@${IP}
done < machines.txt
```

Once each key is added, verify SSH public key access is working:

```bash
while read IP HOST; do
  ssh -n root@${IP} hostname
done < machines.txt
```

## Hostnames

In this section you will assign hostnames to the machines. The hostname will be used when executing commands from the `jumpbox` to each machine. The hostname also plays a major role within the cluster. Instead of Kubernetes clients using an IP address to issue commands to the Kubernetes API server, those clients will use the `server` hostname instead. Hostnames are also used by each worker machine, `worker1` and `worker2` when registering with a given Kubernetes cluster.

To configure the hostname for each machine, run the following commands on the `jumpbox`.

Set the hostname on each machine listed in the `machines.txt` file:

```bash
while read IP HOST; do
    CMD="sed -i 's/^127.0.1.1.*/127.0.1.1\t${HOST}/' /etc/hosts"
    ssh -n root@${IP} "$CMD"
    ssh -n root@${IP} hostnamectl set-hostname "${HOST}"
    ssh -n root@${IP} systemctl restart systemd-hostnamed
done < machines.txt
```

Verify the hostname is set on each machine:

```bash
while read IP HOST; do
  ssh -n root@${IP} hostname --fqdn
done < machines.txt
```

```text
controller1
controller2
controller3
worker1
```

## Host Lookup Table

In this section you will generate a `hosts` file which will be appended to `/etc/hosts` file on the `jumpbox` and to the `/etc/hosts` files on all three cluster members used for this tutorial. This will allow each machine to be reachable using a hostname such as `server`, `node-0`, or `node-1`.

Create a new `hosts` file and add a header to identify the machines being added:

```bash
echo "" > hosts
echo "# Kubernetes Machines" >> hosts
```

Generate a host entry for each machine in the `machines.txt` file and append it to the `hosts` file:

```bash
while read IP HOST; do
    ENTRY="${IP} ${HOST}"
    echo $ENTRY >> hosts
done < machines.txt
```

Review the host entries in the `hosts` file:

```bash
cat hosts
```

```text

# Kubernetes Machines
XXX.XXX.XXX.XXX controller1
XXX.XXX.XXX.XXX controller2
XXX.XXX.XXX.XXX controller3
XXX.XXX.XXX.XXX worker1
```

## Adding `/etc/hosts` Entries To A Local Machine

In this section you will append the DNS entries from the `hosts` file to the local `/etc/hosts` file on your `jumpbox` machine.

Append the DNS entries from `hosts` to `/etc/hosts`:

```bash
cat hosts >> /etc/hosts
```

Verify that the `/etc/hosts` file has been updated:

```bash
cat /etc/hosts
```

```text
127.0.0.1       localhost
127.0.1.1       jumpbox

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# Kubernetes Machines
XXX.XXX.XXX.XXX controller1
XXX.XXX.XXX.XXX controller2
XXX.XXX.XXX.XXX controller3
XXX.XXX.XXX.XXX worker1
```

At this point you should be able to SSH to each machine listed in the `machines.txt` file using a hostname.

```bash
while read IP HOST; do
   ssh -n root@${HOST} hostname
done
```

```text
controller1.k8s.test
controller2.k8s.test
controller3.k8s.test
worker1.k8s.test
```

## Adding `/etc/hosts` Entries To The Remote Machines

In this section you will append the host entries from `hosts` to `/etc/hosts` on each machine listed in the `machines.txt` text file.

Copy the `hosts` file to each machine and append the contents to `/etc/hosts`:

```bash
while read IP HOST; do
  scp hosts root@${IP}:~/
  ssh -n \
    root@${IP} "cat hosts >> /etc/hosts"
done < machines.txt
```

At this point, hostnames can be used when connecting to machines from your `jumpbox` machine, or any of the three machines in the Kubernetes cluster. Instead of using IP addresses, you can now connect to machines using a hostname such as `controller1`, or `worker1`.

Next: [Provisioning a CA and Generating TLS Certificates](05-certificate-authority.md)
