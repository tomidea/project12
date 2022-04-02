## ANSIBLE REFACTORING AND STATIC ASSIGNMENTS (IMPORTS AND ROLES)

#### Step 1 – Jenkins job enhancement

1. Go to your Jenkins-Ansible server and create a new directory called ansible-config-artifact – we will store there all artifacts after each build.
    
        sudo mkdir /home/ubuntu/ansible-config-artifact

2. Change permissions to this directory, so Jenkins could save files there – chmod -R 0777 /home/ubuntu/ansible-config-artifact
3. Go to Jenkins web console -> Manage Jenkins -> Manage Plugins -> on Available tab search for Copy Artifact and install this plugin without restarting Jenkins

4. Create a new Freestyle project (you have done it in Project 9) and name it save_artifacts.
5. This project will be triggered by completion of your existing ansible project. Configure it accordingly:
    
    <img width="862" alt="General" src="https://user-images.githubusercontent.com/51254648/161381689-0c053de1-81ce-4dfa-8de8-d7c73832e52d.png">

<img width="858" alt="build triggers" src="https://user-images.githubusercontent.com/51254648/161381692-6d73c3fb-40c7-4d52-ab75-9213d90ba943.png">

6. The main idea of save_artifacts project is to save artifacts into /home/ubuntu/ansible-config-artifact directory. To achieve this, create a Build step and choose Copy artifacts from other project, specify ansible as a source project and /home/ubuntu/ansible-config-artifact as a target directory.

<img width="1066" alt="build" src="https://user-images.githubusercontent.com/51254648/161381693-a4ca253e-4180-4334-ac3e-0b9a7f99697c.png">

7. Test your set up by making some change in README.MD file inside your ansible-config-mgt repository (right inside master branch).

<img width="892" alt="2 builds" src="https://user-images.githubusercontent.com/51254648/161381695-733a882f-732c-426b-b84c-f8b39f6992c4.png">

If both Jenkins jobs have completed one after another – you shall see your files inside /home/ubuntu/ansible-config-artifact directory and it will be updated with every commit to your master branch

<img width="478" alt=":home:ubuntu:ansible-config-artifact " src="https://user-images.githubusercontent.com/51254648/161381696-8f2c0aed-98d8-411e-b6a8-29822b12bdfe.png">


#### Step 2 – Refactor Ansible code by importing other playbooks into site.yml
1. Within playbooks folder, create a new file and name it site.yml – This file will now be considered as an entry point into the entire infrastructure configuration. Other playbooks will be included here as a reference. In other words, site.yml will become a parent to all other playbooks that will be developed. Including common.yml that you created previously.
2. Create a new folder in root of the repository and name it static-assignments. The static-assignments folder is where all other children playbooks will be stored. 
3. Move common.yml file into the newly created static-assignments folder.
4. Inside site.yml file, import common.yml playbook.
         
         ---
          - hosts: all
          - import_playbook: ../static-assignments/common.yml
 The code above uses built in import_playbook Ansible module.
          
   <img width="443" alt="site file " src="https://user-images.githubusercontent.com/51254648/161381697-18cb1746-1914-4a67-b9bb-2d1b9121000e.png">

   
 5. Run ansible-playbook command against the dev environment
Since you need to apply some tasks to your dev servers and wireshark is already installed – you can go ahead and create another playbook under static-assignments and name it common-del.yml. In this playbook, configure deletion of wireshark utility.

<img width="338" alt="common-del" src="https://user-images.githubusercontent.com/51254648/161381698-fe183713-3a2c-498b-b79e-25342e9177c2.png">


update site.yml with - import_playbook: ../static-assignments/common-del.yml instead of common.yml and run it against dev servers:
        
        ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/dev.yml /home/ubuntu/ansible-config-mgt/playbooks/site.yaml
 
 <img width="966" alt="run playbook" src="https://user-images.githubusercontent.com/51254648/161381700-496040c0-b3d5-4571-85c9-7c9a99696d86.png">

  
  Make sure that wireshark is deleted on all the servers by running wireshark --version
 
 <img width="583" alt="check wireshark" src="https://user-images.githubusercontent.com/51254648/161381704-bd33b126-3455-44f8-854f-ddd140e6b77f.png">

  
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
- Clone Tooling website from GitHub https://github.com/<<<your-name>>>/tooling.git.
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
                repo: https://github.com/<<your-name>>/tooling.git
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
    
    
 #### Step 5 – Commit & Test
    
Commit your changes, create a Pull Request and merge them to master branch, make sure webhook triggered two consequent Jenkins jobs, they ran successfully and copied all the files to your Jenkins-Ansible server into /home/ubuntu/ansible-config-mgt/ directory.

Now run the playbook against your uat inventory and see what happens:
    
        sudo ansible-playbook -i /home/ubuntu/ansible-config-mgt/inventory/uat.yml /home/ubuntu/ansible-config-mgt/playbooks/site.yaml

<img width="822" alt="playbook success" src="https://user-images.githubusercontent.com/51254648/161381710-dda8aae3-55b4-4144-8907-63d9aac1f464.png">

    
Check both of UAT Web servers configured and try to reach them from your browser:
    
http://54.211.230.35/index.php
  
<img width="1182" alt="uat web 1" src="https://user-images.githubusercontent.com/51254648/161381708-965f1879-86d0-4048-a2f9-10300e059767.png">

    
http://3.81.208.64/index.php
<img width="1175" alt="uat web 2" src="https://user-images.githubusercontent.com/51254648/161381705-96b857e7-ff89-45fe-8922-890a5719580a.png">
