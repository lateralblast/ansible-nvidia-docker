![alt tag](https://raw.githubusercontent.com/lateralblast/ansible-nvidia-docker/master/images/cat_monitor.jpg)

Automating Nvidia Driver and Docker Support Installation with Ansible
=====================================================================

Introduction
------------

This is a quick example of how to automate Nvidia driver and Nvidia Docker support installation with Ansible

If successful, as a simple example, you should be able to do something like this:

```
# docker run --gpus all nvidia/cuda:9.1-base nvidia-smi
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.138                Driver Version: 390.138                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla M2090         Off  | 00000000:04:00.0 Off |                    0 |
| N/A   N/A    P0    74W /  N/A |      0MiB /  5301MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  Tesla M2090         Off  | 00000000:42:00.0 Off |                    0 |
| N/A   N/A    P0    77W /  N/A |      0MiB /  5301MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

Resources
---------

Installing  Docker on Ubuntu:

https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04

Nvidia Docker:

https://github.com/NVIDIA/nvidia-docker

Requirements
------------

Required hardware:

- Nvidia Cuda cable GPU

Required software:

- Ansible
- Nvidia Drivers
- Nvidia Toolkit and Samples (optional)
- Docker
- Nvidia Docker image

A reboot is required to install Nvidia drivers

Overview
--------

This example does the following:

- Install Nvidia drivers
- Install Docker Community Edition
- Install Docker Toolkit and Samples (options)
- Install Docker

Ansible
-------

Install Nvidia Drivers and other required packages

```
- name: Check Nvidia Packages
  apt:
    name:   "{{ item.package }}"
    state:  "{{ item.state }}"
  loop:
    - { package: "python3-docker",            state: "present" }
    - { package: "python3-pip",               state: "present" }
    - { package: "python3-setuptools",        state: "present" }
    - { package: "nvidia-driver-390",         state: "present" }
    - { package: "freeglut3",                 state: "present" }
    - { package: "freeglut3-dev",             state: "present" }
    - { package: "libxi-dev",                 state: "present" }
    - { package: "libxmu-dev",                state: "present" }
    - { package: "gcc-7",                     state: "present" }
    - { package: "g++-7",                     state: "present" }
    - { package: "nvidia-container-toolkit",  state: "present" }
```

Set up python:

```
- name: Check Base Python Alternatives Configuration
  alternatives:
    link: /usr/bin/python
    name: python
    path: /usr/bin/python3.6
  when: ansible_distribution_version == '18.04

- name: Check Base Python Alternatives Configuration
  alternatives:
    link: /usr/bin/python
    name: python
    path: /usr/bin/python3.8
  when: ansible_distribution_version == '20.04'

- name: Check Base Symlink for pip
  file:
    src:   "/usr/bin/pip3"
    dest:  "/usr/bin/pip"
    state: link
    force: yes

- name: Check Base Symlink for easy_install
  file:
    path:  "/usr/lib/python3/dist-packages/setuptools/command/easy_install.py"
    owner: root
    group: root
    mode:  "0755

- name: Check Base Permissions for easy_install
  file:
    src:   "/usr/lib/python3/dist-packages/setuptools/command/easy_install.py"
    dest:  "/usr/bin/easy_install"
    state: link
```

Add Docker repository:

```
- name: Check Base Docker Repository
  apt_repository:
    repo:         deb [arch=amd64] https://download.docker.com/linux/ubuntu disco stable
    state:        present
    filename:     docker-ce
    update_cache: no
  register: docker_repo
  when: and ansible_distribution_version == '18.04'

- name: Check Base Docker Repository
  apt_repository:
    repo:         deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable
    state:        present
    filename:     docker-ce
    update_cache: no
  register: docker_repo
  when: ansible_distribution_version == '20.04'

- name: Check Base Docker Repository Key
  apt_key:
    url:    https://download.docker.com/linux/ubuntu/gpg
    state:  present
  when: docker_repo.changed == true
```

Install Docker packages:

```
- name: Check Base Docker Packages
  apt:
    name:         "{{ item.package }}"
    state:        "{{ item.state }}"
    update_cache: yes
  loop:
    - { package: "docker-ce",     state: "present" }
    - { package: "docker-ce-cli", state: "present" }
    - { package: "containerd.io", state: "present" }
```

Install Nvidia Docker repository:

```
- name: Check Nvidia Docker Repository Key
  apt_key:
    url:    https://nvidia.github.io/nvidia-docker/gpgkey
    state:  present

- name: Check Nvidia Docker Repository
  apt_repository:
    repo:     "{{ item.repo }}"
    state:    "{{ item.state }}"
    filename: nvidia-docker
  loop:
  - { repo: "deb https://nvidia.github.io/libnvidia-container/stable/ubuntu18.04/$(ARCH) /",  state: "present" }
  - { repo: "deb https://nvidia.github.io/nvidia-container-runtime/ubuntu18.04/$(ARCH) /",    state: "present" }
  - { repo: "deb https://nvidia.github.io/nvidia-docker/ubuntu18.04/$(ARCH) /",               state: "present" }
```

Install Nvidia Docker images:

```
- name: Check Base Docker Nvidia image
  docker_image:
    name:   nvidia/cuda:9.1-base
    source: pull
```

Install Nvidia Toolkit and Samples if required:

```
- name: Check Nvidia Base Directory
  stat:
    path: /usr/local/cuda-9.1
  register: cuda_base_dir

- name: Check Nvidia Toolkit and Samples Install Files
  copy:
    src:   ./base/files/tmp/cuda_9.1.85_387.26_linux.run
    dest:  /tmp/cuda_9.1.85_387.26_linux.run
    owner: root
    group: root
    mode:  "0755"
    force: no
  when: cuda_base_dir.stat.exists == false

- name: Check Nvidia Toolkit and Samples Install
  command: sh -c "chmod +x /tmp/cuda_9.1.85_387.26_linux.run ; sh /tmp/cuda_9.1.85_387.26_linux.run --toolkit --samples --samplespath=/usr/local/cuda-9.1 --override --verbose --silent"
  args:    
    chdir:   /usr/local
    creates: /usr/local/cuda-9.1
  when: cuda_base_dir.stat.exists == false
```