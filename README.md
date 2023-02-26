
While working on a product's development stage, you will be checking how the changes in the code are reflected in a live environment.

If the product is running on a docker container, you must create a new docker image and run a container based on the new image to check the changes, right?

So here I am trying to create a solution for that, whenever you are making an update to your GitHub code, you only need to run the ansible playbook. It will create a docker image and will push the image to the docker hub And it will create a docker container based on the new image.

![setup-new](https://user-images.githubusercontent.com/61390678/221403972-2140b4ad-0ca4-484f-be11-2ff9c120a331.png)

So let’s start.

Before proceeding please make sure that you have installed ansible on your master machine and that you have the key file (key.pem in this example) with the required permissions. We are using an amazon Linux instance here. Our application is a program to display hello world and it is written in python.

We need to configure the inventory files with the details of the build and test server details.

Our inventory file will be like the following.

```
$ cat inventory

[build]
<pri_IP> ansible_user="ec2-user" ansible_ssh_private_key_file="key.pem"
[test]
<pri_IP> ansible_user="ec2-user" ansible_ssh_private_key_file="key.pem"
```

Also, we need the docker hub’s details such as username and password for authenticating and pushing images to docker hub.

We are passing the credentials to the playbook from a file, so we are setting the file as dockervars.yml.

```

$ cat dockervars.yml
---
username: "enter your dockerhub username"
password: "enter your dockerhub password"
```

Now let’s check on the playbook.

```
$ cat git.yml
---
- name: "deploying docker from git"
  hosts: build
  become: true
  vars_files:
    - dockervars.yml
  vars:
    pkg:
      - docker
      - git
      - pip
    url: "https://github.com/antony-a-n/devops-flask.git"
    clone: "/var/flaskapp"
    img: "antonyanlinux/flask"

  tasks:
    - name: "installing packages"
      yum:
        name: "{{pkg}}"
        state: present

    - name: "adding user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true

    - name: "installing python module"
      pip:
        name: docker-py

    - name: "restarting service"
      service:
        name: docker
        state: restarted
        enabled: true

    - name: "creating document-root"
      file:
        path: "{{clone}}"
        state: "directory"

    - name: "cloning-repo"
      git:
        repo: "{{url}}"
        dest: "{{clone}}"
      register: clone_state
    - name: "login"
      when: clone_state.changed
      docker_login:
        username: "{{username}}"
        password: "{{password}}"
        state: present

    - name: "image building"
      when: clone_state.changed
      docker_image:
        source: build
        build:
          path: "{{clone}}"
          pull: true
        name: "{{img}}"
        tag: "{{item}}"
        push: true
        force_tag: true
        force_source: true
      with_items:
        - latest
        - "{{clone_state.after}}"

    - name: "removing local image"
      when: clone_state.changed
      docker_image:
        state: absent
        name: "{{img}}"
        tag: "{{item}}"
      with_items:
        - latest
        - "{{clone_state.after}}"

    - name: "logout"
      when: clone_state.changed
      docker_login:
        username: "{{username}}"
        password: "{{password}}"
        state: absent

- name: "testing image"
  hosts: test
  become: true
  vars:
    test_img: "antonyanlinux/flask"
    test_pkg:
      - docker
      - pip

  tasks:
    - name: "package installation"
      yum:
        name: "{{test_pkg}}"
        state: "present"
    - name: "attaching user"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true

    - name: "installing module"
      pip:
        name: docker-py
    - name: "restarting service"
      service:
        name: docker
        state: started
        enabled: true
    - name: "pulling image"
      docker_image:
        name: "{{test_img}}"
        source: pull
        force_source: true
      register: image_stat

    - name: "creating docker container"
      when: image_stat.changed
      docker_container:
        name: flaskdemo
        image: "{{test_img}}:latest"
        recreate: yes
        pull : yes
        published_ports:
          - "80:5000"
```

We are setting up a few variables such as GitHub repo URL, work directory for the application, and image name.

We are running this as 2 plays, In the 1st play we are installing the required packages and granting permissions to the ec2-user on docker.

Once you have added ec2-user to the group, we need to restart the service for reflecting the changes.

Please note that in order to python work with docker, you need to install a python module.

Once this is done, we are creating the document root, for the project and we are cloning the git repo to the directory which we have created. we are using the git module here.

When the repo is cloned, we are ready to build a new docker image, before that we need to access the docker hub for pushing the image. docker_login module can be used here.

In the next build, we are building the image. build refers that we are building a new image and we are adding the necessary tags to the image.

Once the image is pushed to the hub successfully we can remove the image from the local machine.

In the next play, we are repeating the procedures for installing packages, grading permissions, etc, Once this is done, we are pulling the image which was created as a part of the previous play.

And based on the image we are launching a docker container.

Now let’s execute the playbook.

```
$ ansible-playbook -i inventory git.yml
```

```$ ansible-playbook -i inventory git.yml
[WARNING]: ansible.utils.display.initialize_locale has not been called, this may result in incorrectly calculated text widths that can cause Display to print incorrect line
lengths

PLAY [deploying docker from git] *****

TASK [Gathering Facts] *****


TASK [installing packages] *****
changed: [172.31.34.11]

TASK [adding user to docker group] *****
ok: [172.31.34.11]

TASK [installing python module] ****
ok: [172.31.34.11]

TASK [restarting service] *****
changed: [172.31.34.11]

TASK [creating document-root] ******
changed: [172.31.34.11]

TASK [cloning-repo] *****
changed: [172.31.34.11]

TASK [login] *******
changed: [172.31.34.11]



--
--
--
TASK [creating docker container] ******

PLAY RECAP *****
172.31.33.88               : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
172.31.34.11               : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

Once it is executed successfully, you will be able to view the output from the browser.

![ss1](https://user-images.githubusercontent.com/61390678/221404141-40848d3a-336e-4291-882f-0a8898e63927.png)

Now let’s change the version to 2 and re-run the playbook.

```
$ ansible-playbook -i inventory git.yml
````

```
PLAY RECAP **************
172.31.33.88               : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
172.31.34.11               : ok=11   changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Now you will be able to view the newer version in the browser.

![ss2](https://user-images.githubusercontent.com/61390678/221404180-c9e704bc-186c-44aa-aa80-c3e3dc02a683.png)


Thank you for reading, if you are facing any issues, feel free to ping on LinkedIn :)

