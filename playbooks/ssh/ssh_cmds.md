**Create a pair of SSH keys**
ssh-keygen -f /home/thor/.ssh/maria  
ssh-keygen -f /home/thor/.ssh/john  

**Distribute the public keys to the web and database servers**
ssh-copy-id -i /home/thor/.ssh/maria  maria@lamp-db  
ssh-copy-id -i /home/thor/.ssh/john  john@lamp-web
