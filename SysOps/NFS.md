# NFS
## Introduction
NFS is a protocol that allows a server to serve a disk over the network to specific clients.

- **Good NFS setup ressource:** https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-debian-11
- **MacOS NFS:** https://www.cyberciti.biz/faq/apple-mac-osx-nfs-mount-command-tutorial/
### Use cases
- **Serving data read-only to workstations:** tools, software, ISO, etc
- **Simple shared storage:** music, movies, pictures
- **Simple backup**
- **Drop zone for files:** any host designated on the server can use the "drop zone"
### Pros & Cons
#### Pros
- Easy to setup
- Standard support on most \*nix machines: **BSD, Linux, MacOS**
#### Cons
- No credentials
### Server-side config
- To allow new hosts on the share append the host and it's options at the end of the line in */etc/exports* 
	`folder/you/want/to/share/on/the/server host1(rw,sync,no_subtree_check,no_root_squash) host2(rw,sync,no_subtree_check)`
### Client-side mounting
- **Mounting on client system ([[MacOS]] version):**`sudo mount -t nfs -o resvport,rw,nfsvers=3 hostname:/mnt/data DNAS`
- **Mounting on client system:**`sudo mount -t nfs -o rw,nfsvers=3 hostname:/mnt/data DNAS`

### Known issues
- Use NFS3 when mounting for clients that need to write to the shared disk. `-o nfsvers=3`