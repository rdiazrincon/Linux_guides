# Instructions to set up an SSH server and client with OpenSSH

Info on OpenSSH. Includes how to set up an ssh server, ssh key-based authentication and more.

## On the Client side

### Install the required packages

Look for SSH binaries
~~~
which ssh
~~~
Check if the OpenSSH client is installed 
~~~
apt search openssh-client
~~~
If it's installed you will see something like this:

![OpenSSH installed](images/OpenSSH/OpenSSH_installed.jpg)

If not, install some updates first
~~~
 sudo apt update && sudo apt upgrade 
~~~
And then install the OpenSSH client
~~~
sudo apt install openssh-client
~~~
### Connect to an SSH server
Since we have the client already installed, run:
~~~
ssh username@ipadress 
~~~
In my case I will long in to an AWS EC2 instance using ssh key authentication (more on this later) so it looks like this:

![Login into EC2 instance](images/OpenSSH/Logging_ssh_server.jpg)

As long as the server is running and accepting incoming traffic on the port **22** you should at least get a password prompt. Whenever we connect for the first time it will ask us if we are sure we want to connect.

If this is the first time we connect to an ssh server, in the client, we will now have a .ssh folder that includes a **known_hosts** file. 

![Known Hosts](images/OpenSSH/Known_hosts.jpg)

This file contains the fingerprint of all the SSH servers we have connected to. If we connect again to the same server, 
this time it won't ask if we are sure if we want to connect since the info is already in the **known_hosts** file. 

This is important because it prevents a man-in-the-middle attack. Let's say someone creates a malicious server and they were able to set our same ip address, if that's the case, when we try to connect, Ubuntu will see that out fingerprint has changed and when that happens it will raise a warning.

### Locating the auth.log file

In the client, when log into the server

![Login again](images/OpenSSH/Auth_log_1.jpg)

Locate the log directory
~~~
cd /var/log/
~~~

When we run
~~~
tail -f auth.log
~~~
we can track the changes for the auth.log for every login attempt

![Login again](images/OpenSSH/Auth_log_2.jpg)

This file is very important since it allows to see what is going on whenever we attempt to connect to the server.

### Configuring the OpenSSH Client

Let's create a config file for the SSH client. Move into the .ssh folder
~~~
cd .ssh
~~~
Create the config file
~~~
touch config
~~~
And change the file this way
![SSH config](images/OpenSSH/ssh_config.jpg)
Now instead of doing
~~~
ssh username@ipadress
~~~
or 
~~~
ssh -i "public_key" username@ipaddress
~~~
We can just say 
~~~
ssh sshserver
~~~
Since I'm using a Public Key on my side it looks like this:
![Login with public key](images/OpenSSH/ssh_config_2.jpg)

**Note** that the Host variable at the end can have whatever name you want.
Let's change the said variable this way:
![Changing the host variable](images/OpenSSH/ssh_config_3.jpg)

Now when I run:
~~~
ssh -i "public_key" newsshserver
~~~
We can see we are able to log in
![Login with new Host variable](images/OpenSSH/ssh_config_4.jpg)

We can also include multiple servers in the file. It would look something like this:
~~~
Host newsshserver
  Hostname ec2-3-22-70-11.us-east-2.compute.amazonaws.com
  Port 22
  User ubuntu

Host otherserver
  Hostname 172.21.284.1
  Port 22
  User ricardo 
~~~

### Creating an SSH key
An SSH key is a more secure way to login into a server. It adds another layer of security in your server since most passwords tend to be predictable.

Inside .ssh run the following command to generate an ssh key:
~~~
ssh-keygen
~~~
It will create a new ssh key at /home/user/.ssh/id_rsa by default
**Important:** You want to make sure you don't already have a key in that directory. If you do, it will be overwritten and can be bad if the key is the only way in to the server.

![SSH Keygen](images/OpenSSH/ssh-keygen.jpg)

Now, in the .ssh directory we will two new files: id_rsa and id_rsa.pub. The last one is the public key and, since it is public, it could be shown to everybody. The private key on the other hand (The one without the .pub extension) should not, under any circumstance, be shared or published.

### Setting up the server to autenthicate via SSH key

In .ssh cat-out the public key:
~~~
cat id_rsa
~~~
Login into the remote server
~~~
ssh username@ipadresss
~~~
Setup the server to accept connections via that key.
Go to .ssh in the server. If the directory doesn't exists create it with mkdir .ssh. Then create a file called authorized_keys

Edit the file authorized_keys and paste the key. You can have multiple keys, each one on it's own line
~~~
sudo nano authorized_keys
~~~
Save the file and then login into the server.

![SSH Authentication](images/OpenSSH/ssh_key_authentication.jpg)

Note that, in my case, since I have an Amazon EC2 instance, to go into the remote server I had to specify the key (.rem file) to login. Now I can just run
~~~
ssh newsshserver
~~~
Again, what we just did was adding the public key on it's own line into the authorized_keys file of the .ssh folder of the server. 

When we used the ssh command to connect to the server, it checked the private key against the public key on the remote end, since they are a match, we have a connection.

If you want more information about what Ubuntu does when trying to connect to the remote server you can run:
~~~
ssh -v username@ipadress
~~~
### Easy way to set up SSH authentication
Another way is to create a new ssh key with ssh-keygen and then run:
~~~
ssh-copy-id -i ~/.ssh/id_rsa.pub username@ipaddress
~~~
The argument after -i is the location of the public key

### Managing SSH keys

You can have different ssh keys that are specific for each remote server.

Generating a key with a different type. This one is more secure and shorter than the default type. What goes after C is a commentary and is located at the end of the key. If we don't include it, it will default to our username@nameofcomputer.

Adding a comment allows us to add extra information that can include the purpose of that key.
~~~
ssh-keygen -t ed25519 -C "godzilla"
~~~
You can choose to save it under a different name. Entering a passphrase is option but it add another security layer.

![New type of SSH key](images/OpenSSH/new_key_type.jpg)

Let's compare the two types of keys.

![Comparing Key typed](images/OpenSSH/comparing_key_types.jpg)

The second key is shorter and more secure because it uses a stronger type of cryptography. The comment is also added at the end.

Let's copy the new key to the server. Replace "ubuntu@ec2-3-22-70-11.us-east-2.compute.amazonaws.com" for username@ipaddress
~~~
ssh-copy-id -i ~/.ssh/godzilla_id_ed25519.pub ubuntu@ec2-3-22-70-11.us-east-2.compute.amazonaws.com
~~~
![Copying new Key](images/OpenSSH/copying_new_key.jpg)

Now, when we log into the server and cat the authorized_keys file, we will see the new key added at the end. In my case, I already had to keys but the new one is highlighted. That's why it is very important to add comments to the keys.

![cat command on new Key](images/OpenSSH/cat_new_key.jpg)

**Extremely important note:** When you decide to give your ssh key a different directory or name than the default, you will need to tell ubuntu **where** to look for the ssh key. By default ubuntu will look for files like *id_rsa* or *id_ed25519* if that name doesn't match you will need to be specific while trying to login into the server. See image below:

![Login error](images/OpenSSH/login_error.jpg)

Additionally, in .ssh/config you can add *IdentityFile* at the end and specify where your ssh key is located so that you can use the alias again to run the server. 

If you add IdentityFile to your ssh config, you'll find that the client still sends the default key (see ssh -vv output). This can be problematic when using sites like github with multiple accounts. You'll need to include IdentitiesOnly yes if you want ssh to use only the key you've specified.

After that you can login again normally.

~~~
Host newsshserver
  Hostname ec2-3-143-214-231.us-east-2.compute.amazonaws.com
  Port 22
  User ubuntu
  IdentityFile ~/.ssh/godzilla_id_ed25519
  IdentitiesOnly yes
~~~  
![New config file](images/OpenSSH/ssh_config_new.jpg)

### Using the ssh agent to catch the key in memory to only enter the passphrase once
If you set up a passphrase you will notice that we are going to be asked to provide it everytime. To avoid this (type it only one time) we can use the ssh agent.

The ssh agent will retain the key in memory and allow you to use it to connect to the server. 

Checking if the ssh agent is runing
~~~
ps aux |grep ssh-agent
~~~

![SSH agent running](images/OpenSSH/ssh-agent_running.jpg)

In my case it was already running and it had a specific pid (process).

In case it isn't running in your machine, you can start the ssh agent by runing:
~~~
eval "$(ssh-agent)"
~~~
Keep in mind that once we are disconected from the terminal, the ssh agent is gone. While it's still in memory and with the terminal open we can add the passphrase to the agent. This time we need to pass the private key.
~~~
ssh-add ~/.ssh/godzilla_id_ed25519.pub
~~~
Once we type the passphrase, when we login into the server again, it won't ask for it unless the terminal window is closed.

## On the server side

To find out your username:
~~~
whoami
~~~
To find out your ip address:
~~~
ip a
~~~
Install OpenSSH server
~~~
sudo apt-get install openssh-server
~~~
Check the server status
~~~
systemctl status sshd
~~~

### SSH server configuration

The sshd binary represents the server. When you run
~~~
which sshd
~~~
It tells you the location of that binary. 

To check the status of the server run:
~~~
systemctl status sshd
~~~
Enabled here means that it will automatically start whenever the computer starts
![](images/OpenSSH/systemctl.jpg)

With ssh server component we can use all the standard systemctl commands to manage the server running in the background.

To restart the service
~~~
systemctl restart sshd
~~~

To restart the service
~~~
systemctl stop sshd
~~~
When you stop the service it won't terminate the existing connections. If you disconnect you won't be able to connect back. The service will be disconected. It's wise to start the service again after stopping it.
~~~
systemctl start sshd
~~~

To enable the service at boot
~~~
systemctl enable ssh
~~~

In the server, when you go to /etc/ssh you will find a lot of files. Specially some host_keys. Those files are created when we connect for the first time and they contain the accepted the fingerprints. Do not delete those files. If you do and then try to connect again to the server you won't be able to.

You will see the *ssh_config* and *sshd_config*. The *ssh_config* file contains global client configuration settings across the entire distribution. When you use the ssh client it is going configure itself along with this config file and the local ssh config file will override this one.

The *sshd_config* contains the server configuration. We can change the "Port" variable to something different than the default value (22). We can also change "PermitRootLogin" to no once we have another user that we can login with via ssh outside of root, that it since root is the account most hacker will try to login with. Additionally, we can also set the "PasswordAuthentication" variable to "no" so that we can use ssh authentication. 

Once you have finished modifiying the *sshd_config* update the config file (in the client side) as follows.

![Changing the port](images/OpenSSH/changing_port.jpg)

If you change the Port, make sure to add it to the file. On AWS you will also need to add the new port to the security group. Changing your port to something different to 22 actually helps your security.

The single most important security configuration change that you can do is **setting PasswordAuthentication to no**. Only set it to no if you already established an ssh key connection. Make sure you can login with ssh before set it to no.

The image below shows an stablished conection to the new port

![Connected to a new port](images/OpenSSH/connected_new_port.jpg)

## Troubleshooting

A common problem is invalid permissions. To check out your premissions, on the client side, run
~~~
ls -la |grep .ssh
~~~
*d* refers to a directory. *r* refers to read, *w* for write and *x* for execute permissions. The first set of three after the *d *refers to permissions
for the user. The next group of three refers to permissions for the group and the last set refers to everyone else.

![Permissions on the .ssh directory](images/OpenSSH/permissions_ssh_directory.jpg)

In this case the user has full access. The group and others have no access hence the hypens. What we get from this is that only our user can do anything with the folder. If we give permissions to anything else and open the folder up for everyone, SSH won't allow the connection via key because it won't trust a folder that is open and writeable by everybody.

Inside the directory, we can see that the public key is readable by the user, group and other. That is fine since it's a public key. Notice that the private keys can also be readable and writeable by the user that owns the file. 

![Permissions inside the .ssh folder](images/OpenSSH/permissions_ssh_folder.jpg)

If the permissions are set up to something else the connection won't be stablished. If you have a problem connecting to a server via SSH one of the first things you should do is check the permissions. Also, on the server side, make sure that the .ssh folder and the authorized_keys file inside that directory is only readable and writeable by you. 

Additionally, the auth_log file in /var/log on the server side will tell you what goes on when someone tries to login into the server.

Another way to follow up the log entries is by doing
~~~
journalctl -fu ssh
~~~
![Following the log](images/OpenSSH/log_file.jpg)