# Automated Network Module Synchronizer (ANMS)

Let's say you have multiple servers running and despite them having different specific purposes, there are some things that you may wan't to have the same across all of them. It may be custom scripts, specific shell setup like custom bashrc configs and so on. This will have to be applied to each of these and any later changes would also again have to be applied one at a time. 

ANMS is a small pure bash shell script that enables you to sync packages with a simple file server. It will install/update and remove packages based on a simple yaml manifest file and combined with a cron task, this can be fully automated. 

A package is an uncompressed/compressed tar archive packing optional files and optional install/uninstall scripts. Any file packaged within the package is automatically extracted into the filesystem and automatically removed once the package is uninstalled. The install/uninstall scripts can help with custom tasks during these steps. 

## Package Structure

```
src/
permissions
install
uninstall
```

The `src` directory represents the root `/` directory. So to extract a file into `/etc/` it must be placed within `src/etc/` in the package. 

Each line of the `permissions` file is used to define owner and permission of a path. The structure is `path:mod:owner:group` and each segment is optional. Example: `/bin/myscript:+x::`

The `install` and `uninstall` files are optional shell scripts that are executed during installation and removal of a package. 

There is also an optional `pe-install` file that is executed before files are extracted and before `install`. 

The `post-uninstall` is executed after `uninstall` and after files has been removed. 

## Example

Let's make an example package that will apply a custom `.bashrc`. 

```
src/etc/_home_bashrc
src/etc/_global_bashrc
permissions
install
uninstall
```

__install__
```sh
#!/bin/bash

for path in /root /home/*; do
    if [ -d $path ]; then
        user=$(basename $path)
        
        if [ -f $path/.bashrc ]; then
            mv $path/.bashrc $path/.bashrc.bak
        fi
        
        sudo -u $user -- ln -s /etc/_home.bashrc $path/.bashrc
    fi
done

for file in /etc/bash.bashrc /etc/skel/.bashrc; do
    if [ -f $file ]; then
        mv $file $file.bak
    fi
done

ln -s /etc/_home.bashrc /etc/skel/.bashrc
ln -s /etc/_global.bashrc /etc/bash.bashrc

```

__uninstall__
```sh
#!/bin/bash

for path in /root /home/*; do
    if [ -d $path ]; then
        if [ -f $path/.bashrc.bak ]; then
            if [ -L $path/.bashrc ]; then
                unlink $path/.bashrc
            fi
            
            mv $path/.bashrc.bak $path/.bashrc
        fi
    fi
done

for file in /etc/bash.bashrc.bak /etc/skel/.bashrc.bak; do
    if [ -f $file ]; then
        link=$(echo $file | sed 's/\.bak$//')
        
        if [ -L $link ]; then
            unlink $link
        fi
    
        mv $file $link
    fi
done

```

__permissions__
```
/etc/_global.bashrc:0644::
/etc/_home.bashrc:0644::
```

## Usage

To use the script you will need a simple HTTP or FTP file server. The script uses a very basic yaml file structure to avoid any server side logic. All that is needed is a `manifest.yml` file in the root of the fileserver. 

```yaml
modules: mymodule myothermod

mymodule:
  file:
    path: mymodule.tar.gz
    md5: 526082bc64e3d343bb9bcdc32f9087cc

myothermod:
  condition: uname -a | grep 'x86_64'
  file:
    path: myothermod.tar.xz
    md5: 63b7dc6f5317ef365b0bfdb7408f89eb
```

The `modules` key defines the enabled packages. Only those defined here will be installed and if a package is removed from here, it will be automatically uninstalled on a sync. 

The rest contains basic information of each package such as a relative file path and the package md5sum, which is used to both validate the downloaded package and to check for updated versions. 

A package may define a `condition` which is a one-line validation command that can be used to include/exclude packages from certain installations. If the condition fails, the package will not be installed on that particular server. 

__Run Sync__

To run a sync on a server, simply run the following command: 
```sh
anms [http(s)/ftp]://domain.com
```
