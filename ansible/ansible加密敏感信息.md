使用Ansible Vault实现对敏感信息的加密
===

### 一、Ansible使用加密信息场景

在使用Ansible进行自动化运维过程中，不可避免的需要使用各种密码信息来进行相关操作，例如用于ssh连接各个host的密码、数据库连接密码等，这些信息通常被保存在group_vars和host_vars中。如果Ansible脚本放在svn/git中进行管理，或者不小心扩散到不该扩散的地方时，相关的明文密码信息即遭到泄露，具有很大的安全隐患。因此对于敏感的密码信息应该使用密文进行保存。

### 二、避免泄露密码的两种方式

1. 使用Ansible Tower管理Credential
    Ansible Tower提供了Credential管理、多用户权限管理等强大的功能，由系统管理员配置好的各种用户名/密码信息保存在Tower所在的服务器上，其它用户可以使用相关的用户名/密码信息，但是看不到明文密码，所有的任务运行均可通过Web界面来进行。注意Ansible Tower是商用软件，试用的License仅能管理10台以内的服务器，超出需要付费。
2. 使用Ansible Vault保存加密后的密码信息
    如果不考虑使用Ansible Tower，仅仅使用Ansible也可实现密码信息的加密保存，这时需要借助Ansible vault。Ansible vault通过将密码信息加密后保存在yml文件中，并在运行时进行解密，达到保护敏感信息的效果。

### 三、Ansible vault命令行工具的使用

在安装了Ansible之后，会附带一个ansible-vault命令行工具，该命令的使用说明如下：<br>
Usage: ansible-vault [create|decrypt|edit|encrypt|encrypt_string|rekey|view] [options] [vaultfile.yml]
```
ansible-vault create file #输入两遍加密密钥，打开编辑器，输入加密前内容，保存退出后文件内容会被加密
ansible-vault encrypt file #file中提前输入好加密前内容，然后运行该命令，输入两遍加密密钥，完成后文件内容会被加密
ansible-vault decrypt file #file为已加密文件，运行该命令后，输入两遍加密密钥，完成后文件内容会被解密
ansible-vault edit file #file为已加密文件，运行该命令后，输入两遍加密密钥，会打开编辑器并显示加密前内容，编辑后保存退出，文件内容会被重新加密
ansible-vault view file #查看已加密文件的加密前内容
ansible-vault encrypt_string # 运行命令后，输入两遍加密密钥，输入加密前内容，完成后加密后内容会打印在终端
ansible-vault rekey file #为已加密文件修改加密密钥，输入一遍旧密码，然后输入两遍新密码
```

### 四、在Ansible工程中使用vault

在Ansible中使用vault的方式有两种，一种是将加密信息独立的放在一个文件中，然后在其中文件中引用；另一种方式是在变量的value部分直接赋值为加密后的字符串。下面分别介绍两种方式的具体使用方法。

示例场景说明：假设针对名为web的group定义group_vars，如果不采用加密保存变量，则会在group_vars目录下有一个名为web.yml的文件用于保存所有的变量，其内容为：
```yaml
mysql_user: "root"
mysql_pass: "123"
```
1. 使用独立文件保存加密信息
* 在group_vars目录下新建名为`web`的目录，并新建`vars.yml`和`vault`文件，其内容如下：
    ```yml
    #vars.yml
    mysql_user: "root"
    mysql_pass: "{{ vault_mysql_pass }}"

    #vault
    vault_mysql_pass: "123"
    ```
* 对web/vault文件进行加密（假设主密码为123456）
    ```shell
    ansible-vault encrypt vault
    ```
* 在运行ansible-playbook时交互式提供vault密码或者指定vault密码文件
    ```shell
    #交互式提供vault密码
    ansible-playbook web test.yml --ask-vault-pass
    #指定vault密码文件，如果采用此种方式，需要在/path/to/password文件中输入vault密码，此例中为123456
    ansible-playbook web test.yml --vault-password-file=/path/to/password
    ```

2. 使用加密字符串保存加密信息
* 对密码信息进行加密（假设主密码为123456）
    ```shell
    #运行以下命令，其中123为要加密的信息，运行后输入主密码
    ansible-vault encrypt_string 123
    #输出示例
    !vault |
          $ANSIBLE_VAULT;1.1;AES256
          34663862353330633763636462393963373265346331303236643931383063633438353761643338
          3132646462613564343137363636636137633564633232380a666466323830653963313266343233
          38646330353764363832656634613833393137333431333732636432613765623364363936623164
          3164313731623330650a636636366337336564646439323334643465366438616162343064646534
          3335
    ```
* 修改web.yml文件，将加密后的字符串作为mysql_pass的值，具体内容如下：
    ```yml
    mysql_user: "root"
    mysql_pass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          34663862353330633763636462393963373265346331303236643931383063633438353761643338
          3132646462613564343137363636636137633564633232380a666466323830653963313266343233
          38646330353764363832656634613833393137333431333732636432613765623364363936623164
          3164313731623330650a636636366337336564646439323334643465366438616162343064646534
          3335
    ```
* 在运行ansible-playbook时交互式提供vault密码或者指定vault密码文件
    ```shell
    #交互式提供vault密码
    ansible-playbook web test.yml --ask-vault-pass
    #指定vault密码文件，如果采用此种方式，需要在/path/to/password文件中输入vault密码，此例中为123456
    ansible-playbook web test.yml --vault-password-file=/path/to/password
    ```

两种方式最终实现的效果是一样的，都能够以加密方式保存敏感信息，但是个人建议采用第一种方式，主要有以下考虑：
* 加密信息和非加密信息独立存放，易于维护，例如需要修改加密信息，采用方式一只需运行ansible-vault edit命令即可，采用方式二需要运行ansible-vault encrypt_string，然后再手工复制加密字符串，相对比较繁琐，并且针对每个加密变量都要重复一次操作
* 方式二在某些情况下会影响效率，实测中发现如果采用方式二将ansible_ssh_pass进行加密存储，则ansible控制机每一次ssh连接被控制机时，均需要解密ssh密码，大大增加了任务运行的时间
