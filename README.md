

```bash

---
- name: "Ansible CD Creation"
  hosts: localhost
  vars:
    debug_enable: true
    key_pair: dockerlab
    region: us-east-2
    sgroup: ansible-lab
    instance_type: t2.micro
    build_ami: ami-0d8f6eb4f641ef691
    build_user: ec2-user
    production_ami: ami-0d8f6eb4f641ef691
    production_user: ec2-user
    git_url: https://github.com/jithinraju/delivery.git
    clone_dir: /var/image_content
    username: Dockerhub_username
    email: Dockerhub_Email
    password: Dockerhub_password
    user1: ec2-user
    group1: docker
  tasks:

     - name: "labCreation - Creating New KeyPair"
       ec2_key:
             region: "{{ region }}"
             name: "{{ key_pair }}"
       register: key_out

     - debug: var=key_out
       when: debug_enable

     - name: "labCreation - Saving Public KeyContent"
       copy:
          content: "{{ key_out.key.private_key}}"
          dest: "./{{ key_pair }}.pub"
          mode: "0400"
       when: key_out.changed

     - name: "labCreation - Creating Security Group"
       ec2_group:
            name: "{{ sgroup }}"
            description: "Security Group For Ansible Lab"
            region: "{{ region }}"
            rules:
              - proto: all
                cidr_ip: 0.0.0.0/0
            rules_egress:
              - proto: all
                cidr_ip: 0.0.0.0/0

     - name: "labCreation - Creating build Instance"
       ec2:
          instance_type: "{{instance_type}}"
          key_name: "{{key_pair}}"
          image: "{{build_ami}}"
          region: "{{region}}"
          group: "{{sgroup}}"
          wait: yes
          count_tag:
            Name: "build"
          instance_tags:
            Name: "build"
            role: "ansible-docker"
          exact_count: 1
       register: build_out

     - debug: var=build_out
       when: debug_enable

     - name: "labCreation - Creating production Instance"
       ec2:
         instance_type: "{{instance_type}}"
         key_name: "{{key_pair}}"
         image: "{{production_ami}}"
         region: "{{region}}"
         group: "{{sgroup}}"
         wait: yes
         count_tag:
            Name: "production"
         instance_tags:
            Name: "production"
            role: "ansible-docker"
         exact_count: 1
       register: production_out

     - debug: var=production_out
       when: debug_enable


     - name: "labCreation - Waiting For production To Come Online"
       wait_for:
          port: 22
          host: "{{ production_out.tagged_instances.0.public_ip }}"
          timeout: 20
          state: started
          delay: 5

     - name: "labCreation - Adding production To Inventory"
       add_host:
          hostname: production
          ansible_host: "{{ production_out.tagged_instances.0.public_ip }}"
          ansible_port: 22
          ansible_user: "{{ production_user }}"
          ansible_ssh_private_key_file: "./{{ key_pair }}.pub"
          ansible_ssh_common_args: '-o StrictHostKeyChecking=no'

     - name: "labCreation - Waiting For build To Come Online"
       wait_for:
          port: 22
          host: "{{ build_out.tagged_instances.0.public_ip }}"
          timeout: 20
          state: started
          delay: 5
     - name: "labCreation - Adding build To Inventory"
       add_host:
          hostname: build
          ansible_host: "{{ build_out.tagged_instances.0.public_ip }}"
          ansible_port: 22
          ansible_user: "{{ build_user }}"
          ansible_ssh_private_key_file: "./{{ key_pair }}.pub"
          ansible_ssh_common_args: '-o StrictHostKeyChecking=no'

     - name: "Installing git"
       delegate_to: build
       become: yes
       yum:
         name: git
         state: present

     - name: "Creating clone directory"
       become: yes
       delegate_to: build
       file:
         path: "{{ clone_dir }}"
         state: directory

     - name: "Cloning git repository"
       become: yes
       delegate_to: build
       git:
         repo: "{{ git_url }}"
         dest: "{{ clone_dir }}"
       register: repo_status

     - name: "Getting the latest tag from git"
       become: yes
       delegate_to: build
       shell: "git describe --tags"
       args:
         chdir: "{{ clone_dir }}"
       register: latest_tag

     - debug: var=latest_tag.stdout

     - name: "Installing Docker and pip"
       delegate_to: build
       become: yes
       yum:
          name: python-pip,docker
          state: present

     - name: "Installing docker-py"
       delegate_to: build
       become: yes
       pip: name=docker-py

     - name: "restart docker"
       become: yes
       delegate_to: build
       service: name=docker state=restarted

     - name: "Create docker group"
       delegate_to: build
       become: yes
       group:
        name: "{{group1}}"
        state: present
     - name: " Create/check and add the ec2-user  to group"
       delegate_to: build
       become: yes
       user:
         name: "{{user1}}"
         shell: /bin/bash
         groups: "{{group1}}"
         append: yes

     - name: "Docker Login"
       delegate_to: build
       become: yes
       docker_login:
          username: "{{ username }}"
          password: "{{ password }}"
          email: "{{ email }}"

     - name: "Building the DockerImage"
       delegate_to: build
       become: yes
       docker_image:
          path: "{{ clone_dir }}"
          name: "jithinraju/ansiblecd"
          tag: "{{ item }}"
          push: yes
          force: yes
       with_items:
          - latest
          - "{{ latest_tag.stdout }}"
       when: repo_status.changed

     - name: "Removing the images"
       delegate_to: build
       become: yes
       docker_image:
          state: absent
          name: "jithinraju/ansiblecd"
          tag: "{{ item }}"
       with_items:
          - latest
          - "{{ latest_tag.stdout }}"

     - name: "Installing Docker and pip"
       delegate_to: production
       become: yes
       yum:
         name: python-pip,docker
         state: present

     - name: "Installing docker-py"
       delegate_to: production
       become: yes
       pip: name=docker-py

     - name: "restart docker"
       delegate_to: production
       become: yes
       service: name=docker state=restarted


     - name: "Create docker group"
       delegate_to: production
       become: yes
       group:
         name: "{{group1}}"
         state: present

     - name: " Create/check and add the ec2-user  to group"
       delegate_to: production
       become: yes
       user:
         name: "{{user1}}"
         shell: /bin/bash
         groups: "{{group1}}"
         append: yes

     - name: "Running Docker Container"
       delegate_to: production
       become: yes
       docker_container:
          name: "webserver"
          recreate: true
          pull: yes
          image: "jithinraju/ansiblecd:{{ latest_tag.stdout }}"
          published_ports:
           - "80:80"

```
