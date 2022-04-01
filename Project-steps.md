## ANSIBLE REFACTORING AND STATIC ASSIGNMENTS (IMPORTS AND ROLES)

#### Step 1 – Jenkins job enhancement

1. Go to your Jenkins-Ansible server and create a new directory called ansible-config-artifact – we will store there all artifacts after each build.
    
        sudo mkdir /home/ubuntu/ansible-config-artifact

2. Change permissions to this directory, so Jenkins could save files there – chmod -R 0777 /home/ubuntu/ansible-config-artifact
3. Go to Jenkins web console -> Manage Jenkins -> Manage Plugins -> on Available tab search for Copy Artifact and install this plugin without restarting Jenkins

4. Create a new Freestyle project (you have done it in Project 9) and name it save_artifacts.
5. This project will be triggered by completion of your existing ansible project. Configure it accordingly:
img
img
6. The main idea of save_artifacts project is to save artifacts into /home/ubuntu/ansible-config-artifact directory. To achieve this, create a Build step and choose Copy artifacts from other project, specify ansible as a source project and /home/ubuntu/ansible-config-artifact as a target directory.
img
7. Test your set up by making some change in README.MD file inside your ansible-config-mgt repository (right inside master branch).
img
If both Jenkins jobs have completed one after another – you shall see your files inside /home/ubuntu/ansible-config-artifact directory and it will be updated with every commit to your master branch
img

#### Step 2 – Refactor Ansible code by importing other playbooks into site.yml
1. Within playbooks folder, create a new file and name it site.yml – This file will now be considered as an entry point into the entire infrastructure configuration. Other playbooks will be included here as a reference. In other words, site.yml will become a parent to all other playbooks that will be developed. Including common.yml that you created previously.
2. Create a new folder in root of the repository and name it static-assignments. The static-assignments folder is where all other children playbooks will be stored. 
3. Move common.yml file into the newly created static-assignments folder.
4. Inside site.yml file, import common.yml playbook.
         
         ---
          - hosts: all
          - import_playbook: ../static-assignments/common.yml
          The code above uses built in import_playbook Ansible module.
          
   img
   
 5. Run ansible-playbook command against the dev environment
Since you need to apply some tasks to your dev servers and wireshark is already installed – you can go ahead and create another playbook under static-assignments and name it common-del.yml. In this playbook, configure deletion of wireshark utility.
img

update site.yml with - import_playbook: ../static-assignments/common-del.yml instead of common.yml and run it against dev servers:
        
        sudo ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/dev.yml /home/ubuntu/ansible-config-mgt/playbooks/site.yaml
  img
  
  Make sure that wireshark is deleted on all the servers by running wireshark --version
  img
  
  #### Step 3 – Configure UAT Webservers with a role ‘Webserver’
  1. Launch 2 fresh EC2 instances using RHEL 8 image, we will use them as our uat servers, so give them names accordingly – Web1-UAT and Web2-UAT.
  2. To create a role, you must create a directory called roles/, relative to the playbook file or in /etc/ansible/ directory.
  - Use an Ansible utility called ansible-galaxy inside ansible-config-mgt/roles directory (you need to create roles directory upfront)
           
           mkdir roles
           cd roles
           ansible-galaxy init webserver
           
 - Create the directory/files structure manually

3.  Update your inventory ansible-config-mgt/inventory/uat.yml file with IP addresses of your 2 UAT Web servers
           
           [uat-webservers]
        <Web1-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user' ansible_ssh_private_key_file=<path-to-.pem-private-key>
        <Web2-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user' ansible_ssh_private_key_file=<path-to-.pem-private-key>

4. In /etc/ansible/ansible.cfg file uncomment roles_path string and provide a full path to your roles directory roles_path    = /home/ubuntu/ansible-config-mgt/roles, so Ansible could know where to find configured roles.
5. It is time to start adding some logic to the webserver role. Go into tasks directory, and within the main.yml file, start writing configuration tasks to do the following:
- Install and configure Apache (httpd service)
- Clone Tooling website from GitHub https://github.com/<your-name>/tooling.git.
- Ensure the tooling website code is deployed to /var/www/html on each of 2 UAT Web servers.
- Make sure httpd service is started

    The main.yml may consist of following tasks:                            
           
            ---
            - name: install apache
              become: true
              ansible.builtin.yum:
                name: "httpd"
                state: present

            - name: install git
              become: true
              ansible.builtin.yum:
                name: "git"
                state: present

            - name: clone a repo
              become: true
              ansible.builtin.git:
                repo: https://github.com/<your-name>/tooling.git
                dest: /var/www/html
                force: yes

            - name: copy html content to one level up
              become: true
              command: cp -r /var/www/html/html/ /var/www/

            - name: Start service httpd, if not started
              become: true
              ansible.builtin.service:
                name: httpd
                state: started

            - name: recursively remove /var/www/html/html/ directory
              become: true
              ansible.builtin.file:
                path: /var/www/html/html
                state: absent

    #### Step 4 – Reference ‘Webserver’ role
    Within the static-assignments folder, create a new assignment for uat-webservers uat-webservers.yml. This is where you will reference the role.

            ---
            - hosts: uat-webservers
              roles:
                 - webserver
    
Refer your uat-webservers.yml role inside site.yml.

So, we should have this in site.yml

        ---
        - hosts: all
        - import_playbook: ../static-assignments/common.yml

        - hosts: uat-webservers
        - import_playbook: ../static-assignments/uat-webservers.yml
