# DOCKER DATA FILE MIGRATION
## **NOTES**
- **These steps require understanding of Linux system** 
- **These steps require root access**
- **These steps depend on your current /var/lib/docker being an actual directory (not a symlink to another location)**

## **INFORMATION**
- **/path/to/existing** is current path (normally in **/var/lib/docker**)
- **/path/to/new** is new target path (new)
- **/backup/location/file** is backup file (new)

## **MIGRATION STEPS**
1. Stop docker: service docker stop. Verify no docker process is running
   > ```
   > sudo systemctl stop docker.service
   > ```
2. Check and verify **docker** really isn’t running
   > ```
   > ps faux |grep docker
   > ```
3. Take a look at the current docker directory
   > ```
   > ls /var/lib/docker/
   > ```
4. **[Optional]** Make a backup
   > ```
   > tar -zcC /var/lib docker > /backup/location/file/var_lib_docker-backup-$(date ++%Y%m%d.%H%m%S).tar.gz
   > ```
5. Move old /var/lib/docker directory to other location (or you may delete the directory)
   > ```
   > mv /var/lib/docker /other/location
   > ```
6. Make a symlink: 
   > ```
   > ln -s /path/to/new /var/lib/docker
   > ```
7. **[Optional]** Take a peek at the directory structure to make sure it looks like it did before (note the trailing slash to resolve the symlink)
   > ```
   > ls -al /var/lib/docker
   > ```
8. Start docker back up service docker start
   > ```
   > systemctl start docker.service
   > ```
9. Start your containers
