# LXC Container Basic Commands 
``` 
sudo apt-get install lxc
``` 
or 
``` 
sudo snap install lxd 
``` 

```
lxc-ls
```
View the list of created containers 

```
lxc-checkconfig
``` 
Chcking once everything is Ok! 

```
lxc-create
```
Creating a new container 
`lxc-create -n <container-name> -t <template>`

```
lxc-info -n <container-name>
``` 
Provides information about the container 

```
lxc-ls --active
```
Shows a list of active containers 

```
lxc-stop -n <container-name>
```
Stopping the given container name 

```
lxc-start -n <container-name>
``` 
To start the container 

```lxc-info -n <container-name>```
```lxc-ls -f``` 

```
lxc-attach -n <container-name
``` 
Getting a shell inside the container can be done by this 

```
lxc-stop -n <cotanier-name>
``` 
The container can be stopped using this 

```
lxc-destroy -n <container-name>
``` 
Removing the container can be done using this 

