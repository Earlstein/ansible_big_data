Ansible Big Data Automation: Hadoop Installation

This repository contains an Ansible playbook designed to automate the installation and configuration of Apache Hadoop in a flexible, repeatable, and version-controlled manner. It serves as a foundational component for building a complete, automated big data platform.
Features

    Idempotent: The playbook can be run multiple times safely. It will only make changes if the system is not in the desired state.

    Dynamic Versioning: Upgrade Hadoop simply by changing a version number in a configuration file.

    Dual Installation Modes:

        Sudo Mode: Performs a system-wide installation in /opt for a multi-user environment.

        No-Sudo Mode: Installs Hadoop into a specified user's home directory, perfect for environments where you lack root privileges.

    Configuration Templating: All Hadoop XML configuration files are dynamically generated from templates, making customization easy and centralized.

    Dependency Management: Automatically handles Java installation in Sudo mode and verifies its existence in No-Sudo mode.

Prerequisites

    Control Node: A Linux/macOS machine with Ansible installed. The recommended installation method is via pipx:

    # Install pipx
    sudo apt update && sudo apt install pipx
    pipx ensurepath

    # Install ansible with pipx
    pipx install ansible

    Target Node(s): One or more Ubuntu/Debian-based servers.

    SSH Access: Your control node must have SSH key-based access to the target nodes for the user specified in ansible.cfg (default is ubuntu).

Directory Structure

The project follows Ansible best practices to ensure modularity and readability.

ansible_big_data/
├── inventory.ini             # Defines the servers (hosts) to manage.
├── ansible.cfg               # Configures Ansible's behavior.
├── install_hadoop.yml        # The main playbook entrypoint.
│
├── group_vars/
│   └── all.yml               # The central "source of truth" for all variables.
│
└── roles/
    └── hadoop/               # A self-contained role for all Hadoop tasks.
        ├── tasks/
        │   └── main.yml      # The sequence of tasks for installing Hadoop.
        ├── templates/        # Contains Jinja2 templates for config files.
        └── defaults/
            └── main.yml      # Default variables for the role.

Workflow Diagram

The playbook logic follows a conditional path based on the hadoop_install_no_sudo variable.

                                      +---------------------------+
                                      |   ansible-playbook runs   |
                                      +-------------+-------------+
                                                    |
                                +-------------------+------------------+
                                | Reads group_vars/all.yml             |
                                | Is hadoop_install_no_sudo: true?     |
                                +-------------------+------------------+
                                |                                      |
              +-----------------v----------------+      +--------------v---------------+
              | (Yes) NO-SUDO MODE               |      | (No) SUDO MODE                 |
              +----------------------------------+      +--------------------------------+
              | 1. Verify Java is installed.     |      | 1. Create hadoop group/user.   |
              | 2. Create user install dirs.     |      | 2. Install Java via apt.       |
              | 3. Unarchive Hadoop in home dir. |      | 3. Unarchive Hadoop in /opt.   |
              | 4. Set env vars in ~/.bashrc.    |      | 4. Set ownership & permissions.|
              +----------------------------------+      | 5. Set env vars in /etc/profile|
                                |                     +--------------------------------+
                                |                                      |
                                +-----------------v--------------------+
                                                  |
                                +-----------------v--------------------+
                                | COMMON TASKS (Both Modes)            |
                                +--------------------------------------+
                                | 1. Download & Verify Hadoop Tarball  |
                                | 2. Create version-agnostic symlink.  |
                                | 3. Deploy config from templates.     |
                                +--------------------------------------+

Usage Scenarios

Before running, ensure your inventory.ini file is correctly populated with the IP address(es) of your target node(s).
Scenario 1: System-Wide Installation (Sudo Mode)

This mode is ideal for a shared server where Hadoop should be available to all users. It installs Hadoop to /opt.

1. Configuration:
In group_vars/all.yml, ensure the following is set:

hadoop_install_no_sudo: false

In ansible.cfg, ensure password prompting is enabled if you don't have passwordless sudo set up on the remote host:

become_ask_pass = True

2. Run the Playbook:
From the ansible_big_data directory, execute:

ansible-playbook install_hadoop.yml

Ansible will prompt for the BECOME password (the sudo password for your remote user) when it reaches the first task requiring root privileges.

3. Post-Installation (One-Time Manual Step):
SSH into your NameNode and format HDFS. This command deletes all data in HDFS, run it only once.

# On the NameNode server (hadoop-master)
sudo -u hadoop hdfs namenode -format

# Start the services
sudo -u hadoop start-dfs.sh
sudo -u hadoop start-yarn.sh

Scenario 2: User-Local Installation (No-Sudo Mode)

This mode is for environments where you lack root access. It installs Hadoop into a subdirectory of your home directory (~/hadoop_installs by default).

1. Prerequisites:
You must manually install Java on the target machine first.

# On the target server
sudo apt update
sudo apt install openjdk-11-jdk -y

2. Configuration:
In group_vars/all.yml, set the flag to true:

hadoop_install_no_sudo: true

You can change the install location by editing hadoop_user_home_dir.

3. Run the Playbook:
This command will not ask for a sudo password.

ansible-playbook install_hadoop.yml

4. Post-Installation:
SSH into your NameNode. You will need to add the Hadoop bin and sbin directories to your PATH for the current session, or log out and log back in to load the new ~/.bashrc settings.

# On the NameNode server (hadoop-master)
source ~/.bashrc

# Format HDFS (as your user)
hdfs namenode -format

# Start the services
start-dfs.sh
start-yarn.sh

# Access Hadoop from Browser
http://localhost:9870
